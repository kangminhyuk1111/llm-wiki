---
tags: [neochart-crm, database, migration, design]
created: 2026-06-11
updated: 2026-06-11
sources:
  - raw/notes/2026-06-11-db-auto-migration-zero-touch-design.md
---

# DB 자동 마이그레이션 — 신규 병원 제로터치 셋업

신규 병원이 "CRM 설치 → 첫 실행(UAC 승인)"만으로 DB 스키마가 자동 구성되게 하는 설계. 수동 SQL 작업 제로가 목표. **상태: 설계 승인 / 구현 대기** (2026-06-11 작성).

정본 SQL은 CRM 저장소의 `docs/query/migrations/apply-all.sql` — 이 설계는 그 적용을 자동화한다. 코드의 SQL 본문과 정본이 어긋나면 안 되며, 둘 중 하나를 바꾸면 같은 작업 단위에서 동기화한다 (작업 규칙 #8a).

## 전제

- apply-all.sql의 모든 항목은 idempotent 가드(`IF NOT EXISTS` / `IF NULL` / `OBJECT_ID IS NULL`)를 가짐 → 적용 이력 장부 없이 매번 실행해도 안전. 이미 적용된 병원은 카탈로그 조회 몇 번(수십 ms)으로 끝.
- DB 자격증명은 병원 PC의 `C:\Sense\Bin\Sense.ini` — CRM/autosend가 쓰는 기존 연결 재사용.
- 병원 SQL Server는 **Express Edition** → ONLINE 인덱스 빌드 불가(Enterprise 전용). 인덱스 생성 중 테이블 락이 불가피하므로 "언제 만드느냐"(심야)로 회피.

## 마이그레이션 2분류

| 구분 | 내용 | 위험도 |
|---|---|---|
| [B] senseReview 테이블 | `CRM_CONFIG` / `CRM_AUTOSEND`(+컬럼 보강 3건) / `CRM_HEARTBEAT` 신설 | 없음 — 빈 테이블 생성/컬럼 추가, 즉시 완료 |
| [A] EMR 인덱스 2건 | `TB_인적사항` 인덱스 2개, `TB_진료내역` covering index (859k행) | 빌드 중 테이블 락 — 진료시간 실행 금지 |

## 아키텍처 — 2단 자동화

1. **1단: CRM(Electron) 기동 시** — 즉시·비차단으로 [B] 전체 실행 (`ensureBundledAutosend`와 같은 패턴)
   - 신규 모듈 `electron/dbMigrations/`: `crmSchemaStatements.ts`(SQL 상수, 3-part 이름으로 `USE` 불요) + `ensureCrmSchema.ts`(순차 실행 러너, throw 금지)
   - `electron/main.ts`에서 `void ... .catch(log)` 비차단 호출. 로그는 `[dbMigrations] B-1 skip / B-2 created ...` 한 줄 요약
   - `CRM_CONFIG`는 테이블만 생성 — 설정 행은 환경설정 첫 저장 시 생성(기존 동작)
2. **2단: autosend 틱, 심야(03시대) 락 승자만** — [A] 인덱스 2건 존재 확인 후 없으면 생성
   - 신규 모듈 `electron/autosend/nightlyIndexes.ts`: `isNightlyWindow(now)`(03:00≤now<04:00, 순수 함수, vitest 대상) + `ensureEmrIndexes()`(throw 금지)
   - `tick.ts` 락 승자 경로 말미(하트비트 다음)에서 호출. 심야 윈도우 밖이면 검사 자체를 skip(비용 0)
   - 빌드가 수십 초 걸려도 무방 — 락 승자가 잡는 동안 다른 PC 틱은 자연 skip, 03시대 EMR은 무인. 스케줄러 실행 한도 72h에 영향 없음
   - autosend 버전 bump 예정: **0.7.6** ([[autosend-heartbeat]]의 현행은 0.7.5)

## 신규 병원 타임라인

설치 → CRM 첫 실행 → [B] 즉시 적용 (자동발송·하트비트·설정 저장 동작) → 그날 밤 03시대 틱이 인덱스 생성 → 검색/캘린더 성능 정상화. 인덱스 생성 전에도 기능은 동작한다 (느릴 뿐).

## 실패 격리 / 권한

- 두 단 모두 fire-and-forget: 실패해도 CRM UI/발송에 영향 없음, 로그만 남김. idempotent라 매 기동/매 심야 재시도.
- Sense.ini 계정에 DDL 권한이 없는 병원: warn 로그가 누적되며, 런북으로 안내해 해당 병원만 apply-all.sql 수동 적용. 권한 실태 파악은 운영 미결 항목(원문 §7, 미전달).

> ⚠️ 원문 §5(변경 파일) 후반과 §6~§7은 소스가 잘려 전달되지 않음 — 전체 문서 확보 시 보강 필요.

## 관련 항목

- [[autosend-heartbeat]] — 2단이 끼어드는 틱·락 승자·하트비트 구조, 버전 0.7.5→0.7.6
