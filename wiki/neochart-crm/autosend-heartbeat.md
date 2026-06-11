---
tags: [neochart-crm, autosend, heartbeat, operations]
created: 2026-06-11
updated: 2026-06-11
sources:
  - raw/notes/2026-06-11-crm-autosend-reinstall-heartbeat.md
  - raw/notes/2026-06-11-db-auto-migration-zero-touch-design.md
---

# autosend 하트비트 (heartbeat) 운영 상태

Neochart CRM의 autosend가 운영 서버로 보내는 생존 신호(heartbeat) 체계와 그 운영 현황.

## 동작 구조

- autosend는 **1분 틱(tick)** 으로 돌며, 하트비트는 **5분 주기**로 운영 서버에 발신한다.
- 발신 성공 기준은 서버 응답 **204**.
- 발신 기록은 DB의 `CRM_HEARTBEAT` 테이블에 남는다 (`LAST_SENT_AT`, 상태 해시).
- Windows 스케줄러 작업 `NeochartAutoSend`가 autosend 실행을 담당하며, 설치 경로가 바뀌면 새 경로를 가리키도록 **자가 재등록**한다.

## 현재 상태 (2026-06-11 실측)

- 재설치 성공 후 autosend **0.7.5** 기준: 1분 틱 정상, 하트비트 12:50 → 12:55 정확히 5분 주기로 운영 서버 발신 성공 (204).
- 운영 서버의 401 응답이 풀려 있음 — 서버 팀이 반영한 것으로 추정.
- `CRM_HEARTBEAT.LAST_SENT_AT` 12:55 갱신 확인, 상태 해시 안정.
- 설치 경로: `C:\Sense\bin\New_CRM\Neochart CRM\` (경로에 하위폴더가 붙는 이유는 [[electron-builder-path-sanitizer]] 참고).

## 미해결 의문

- 새 설치 첫 틱(12:45)~12:49 사이 **하트비트 시도 로그가 없다**. 12:50부터는 완벽한 5분 주기.
- 추정: 재발 없는 전환기 1회성 현상. 다만 장부가 1행짜리라 사후 재구성이 불가 — 운영 모니터링에서 같은 패턴이 재발하면 그때 조사하기로 함.

## 남은 일 (2026-06-11 기준)

1. 서버 팀에 `HEARTBEAT_OFFLINE_THRESHOLD=12m` 조정이 401 해제와 함께 반영됐는지 확인.
2. v20260611.2.0 3파일 세트를 업데이트 서버에 업로드 (토큰 필요).

## 예정된 변경 (0.7.6)

- [[db-auto-migration]] 설계(승인됨)에 따라 틱의 락 승자 경로 말미(하트비트 다음)에 심야(03시대) EMR 인덱스 자동 생성 단계가 추가될 예정. 버전 0.7.6으로 bump 예정.

## 관련 항목

- [[db-auto-migration]] — 신규 병원 제로터치 DB 셋업 설계 (autosend 틱에 2단이 편입됨)
- [[electron-builder-path-sanitizer]] — 설치 경로에 제품명 폴더가 강제로 붙는 동작
- [[installer-registry-guard-fix]] — 설치 기본 경로가 Program Files로 떴던 원인과 수정
