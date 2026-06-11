---
tags: [neochart-crm, installer, registry, bugfix]
created: 2026-06-11
updated: 2026-06-11
sources:
  - raw/notes/2026-06-11-crm-autosend-reinstall-heartbeat.md
---

# 설치 기본 경로 Program Files 버그와 레지스트리 가드 수정

Neochart CRM 설치 마법사에서 기본 설치 경로가 의도한 경로 대신 `Program Files`로 뜨던 문제의 원인과 수정 (커밋 816a4a0, 수정 완료).

## 원인

- 설치 모드에서 **"모든 사용자"** 를 고르면 UAC 승격된 내부 인스턴스가 **HKLM**의 `InstallLocation`을 기본 경로로 읽는다.
- 그런데 기존 가드는 두 레지스트리 hive(HKCU+HKLM)를 **묶어서** 검사했기 때문에, 실제로는 HKCU에만 기록되고 HKLM에는 영영 기록되지 않는 갭이 있었다.
- 결과: "모든 사용자" 모드에서는 HKLM에 값이 없어 기본 경로가 Program Files로 떨어졌다.

## 수정

- 가드를 **hive별 독립 검사**로 변경 (816a4a0). 다음 패키징부터 "모든 사용자" 모드에서도 새 경로가 기본으로 표시된다.
- 2026-06-11 재설치는 "현재 사용자" 모드로 진행되어 수정 전에도 의도대로 동작한 상태였다.

## 관련 항목

- [[electron-builder-path-sanitizer]] — 같은 설치 마법사의 별개 동작 (제품명 하위폴더 강제)
- [[autosend-heartbeat]] — 재설치 후 운영 상태
