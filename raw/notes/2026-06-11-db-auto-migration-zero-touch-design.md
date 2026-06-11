---
created: 2026-06-11
type: note
note: 원문이 §5 도중에서 잘린 채 전달됨 (§5 후반~§7 누락). 일부 특수문자가 "?"로 깨져 있음 (em-dash 추정).
---

# DB 자동 마이그레이션 설계 ? 신규 병원 제로터치 셋업

> 목표: 신규 병원이 "CRM 설치 → 첫 실행(UAC 승인)" 만으로 DB 스키마가 자동 구성되게 한다. 수동 SQL 작업 제로.
> 작성 2026-06-11. 상태: **설계 승인 / 구현 대기**.
> 정본 SQL: [`docs/query/migrations/apply-all.sql`](../../query/migrations/apply-all.sql) ? 본 설계는 그 적용을 자동화한다.

---

## 0. 배경 / 전제

- apply-all.sql 의 모든 항목은 idempotent 가드(`IF NOT EXISTS` / `IF NULL` / `OBJECT_ID IS NULL`)를 갖는다 → **적용 이력 장부 없이 매번 실행해도 안전**. 이미 적용된 병원은 카탈로그 조회 몇 번(수십 ms)으로 끝난다.
- DB 자격증명은 병원 PC 의 `C:\Sense\Bin\Sense.ini` ? CRM/autosend 가 이미 사용 중인 연결을 재사용한다.
- 병원 SQL Server 는 **Express Edition** ? ONLINE 인덱스 빌드 불가(Enterprise 전용). 인덱스 생성 중 대상 테이블 락이 불가피하므로 "언제 만드느냐"로 회피한다.
- 마이그레이션은 성격이 둘로 나뉜다:

| 구분 | 내용 | 위험도 |
|---|---|---|
| [B] senseReview 테이블 | `CRM_CONFIG` / `CRM_AUTOSEND`(+컬럼 보강 3건) / `CRM_HEARTBEAT` 신설 | 없음 ? 빈 테이블 생성/컬럼 추가, 즉시 완료 |
| [A] EMR 인덱스 2건 | `TB_인적사항` 인덱스 2개, `TB_진료내역` covering index (859k행) | 빌드 동안 테이블 락 ? 진료시간 실행 금지 |

## 1. 아키텍처 ? 2단 자동화

```
[1단] CRM(Electron) 기동 시 ? 즉시, 비차단
      └ [B] 섹션 전체 실행 (ensureBundledAutosend 와 같은 패턴)

[2단] autosend 틱 ? 심야(03시대) 락 승자만
      └ [A] 인덱스 2건 존재 확인 → 없으면 생성
```

- 신규 병원 타임라인: 설치 → CRM 첫 실행 → [B] 즉시 적용 (자동발송/하트비트/설정 저장 모두 동작) → 그날 밤 03시대 틱이 인덱스 생성 (이후 검색/캘린더 성능 정상화). 인덱스 생성 전에도 기능은 동작한다(느릴 뿐).
- `apply-all.sql` 은 수동/긴급/사전적용용 정본으로 유지. 코드의 SQL 본문과 어긋나면 안 된다 (작업 규칙 #8a ? 둘 중 하나 변경 시 같은 작업 단위에서 동기화).

## 2. 1단 ? CRM 기동 시 [B] 적용

- 신규 모듈 `electron/dbMigrations/`:
  - `crmSchemaStatements.ts` ? [B] 섹션 SQL 본문 (순수 상수. apply-all.sql B-1~B-4 와 동일 내용, 3-part 이름 사용 ? `USE` 불요: `CREATE TABLE senseReview.dbo.X ...` 형태)
  - `ensureCrmSchema.ts` ? 러너: 문장 순차 실행, 결과 요약 반환. throw 금지(전 구간 catch).
- 호출: `electron/main.ts` ? `ensureBundledAutosend()` 호출 옆에서 비차단(`void ... .catch(log)`)으로 실행.
- 결과 로그: `[dbMigrations] B-1 skip / B-2 created / ...` 한 줄 요약 (electron-log → main 로그).
- 실패 처리: DDL 권한 없음 등 → 앱 기동에 영향 0, `log.warn` 1줄. 매 기동 재시도(idempotent).
- 참고: `CRM_CONFIG` 는 테이블만 생성된다 ? 설정 행은 기존 동작대로 환경설정 첫 저장 시 생성.

## 3. 2단 ? autosend 심야 인덱스

- 신규 모듈 `electron/autosend/nightlyIndexes.ts`:
  - `isNightlyWindow(now)` ? 순수 함수: 03:00 <= now < 04:00 (vitest)
  - `ensureEmrIndexes()` ? [A] 섹션 SQL 2건 순차 실행 (apply-all.sql A-1, A-2 와 동일). throw 금지.
- 호출: `tick.ts` 락 승자 경로 말미(하트비트 다음) ? `isNightlyWindow` 가 아닐 때는 **검사 자체를 skip** (비용 0). 윈도우 안이면 `IF NOT EXISTS` 가드가 이미 있으니 생성됐으면 카탈로그 조회로 끝.
- 빌드가 수십 초 걸려도 무방: 락 승자 세션이 잡고 있는 동안 다른 PC 틱은 자연 skip, 03시대 EMR 은 무인.
- 한도: 작업 스케줄러 실행 한도 72h ? 영향 없음.
- 로그: 생성 시 `[nightly] IX_... 생성 (NNs)` / 이미 있으면 무로그(노이즈 방지).
- autosend 버전 bump (0.7.6) + 이력 주석.

## 4. 실패 격리 / 권한

- 두 단 모두 fire-and-forget: 실패해도 본 기능(CRM UI / 발송)에 영향 없음. 로그만.
- Sense.ini 계정에 DDL 권한이 없는 병원이 있으면: 매 기동/매 심야 재시도하며 warn 로그 누적 → 런북으로 안내 (해당 병원만 apply-all.sql 수동 적용). 권한 실태는 운영 미결 항목(§7).

## 5. 변경 파

*(이하 원문 잘림 — §5 후반, §6, §7 미전달)*
