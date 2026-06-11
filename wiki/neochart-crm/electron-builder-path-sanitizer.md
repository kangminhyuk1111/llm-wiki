---
tags: [neochart-crm, installer, electron-builder]
created: 2026-06-11
updated: 2026-06-11
sources:
  - raw/notes/2026-06-11-crm-autosend-reinstall-heartbeat.md
---

# electron-builder 설치 경로 sanitizer 동작

electron-builder의 assistedInstaller는 사용자가 지정한 설치 경로에 **제품명이 포함되어 있지 않으면 제품명 하위폴더를 강제로 붙인다** (sanitizer 동작, 비활성화 불가).

## Neochart CRM에서의 결과

- 설치 마법사에 `C:\Sense\Bin\New_CRM`을 표시해도, 실제 설치 위치는 `C:\Sense\bin\New_CRM\Neochart CRM\`이 된다.
- 폴더가 한 단계 더 생기는 것은 어쩔 수 없는 동작 — 버그가 아니라 electron-builder의 의도된 강제 규칙이다.

## 운영상 함의

- 설치 경로를 참조하는 모든 곳(스케줄러 등록, 경로 하드코딩)은 제품명 하위폴더까지 포함한 **최종 경로**를 기준으로 해야 한다. autosend 스케줄러는 자가 재등록으로 이를 흡수한다 ([[autosend-heartbeat]] 참고).

## 관련 항목

- [[autosend-heartbeat]] — 이 경로에 설치된 autosend의 운영 상태
- [[installer-registry-guard-fix]] — 같은 설치 마법사에서 기본 경로가 Program Files로 떴던 별개 이슈
