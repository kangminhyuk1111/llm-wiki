# LLM Wiki

이 저장소는 LLM(Claude)이 작성·유지보수하는 개인 지식 베이스다.
패턴 출처: Andrej Karpathy의 LLM Wiki idea file.

## 핵심 원칙

1. **`raw/`는 불변이다.** 사람이 큐레이션한 원본 소스. Claude는 읽기만 하고 절대 수정·삭제하지 않는다.
2. **`wiki/`는 Claude의 영역이다.** Claude가 작성하고 유지한다. 사용자는 읽기만 하며, 수정이 필요하면 Claude에게 지시한다.
3. **이 파일(CLAUDE.md)은 스키마다.** 위키의 구조와 규약을 정의한다. 사용자와 Claude가 함께 진화시킨다. 규약을 바꾸면 반드시 이 파일에 반영한다.

## 디렉터리 구조

```
raw/                 # 불변 원본 (소스의 형태별 구분)
├── assets/          # 이미지 (Obsidian 첨부 폴더)
├── notes/           # 대화 정리, 메모, 생각, 운영 기록 — YYYY-MM-DD-slug.md
└── clips/           # 웹 아티클 (Obsidian Web Clipper) — YYYY-MM-DD-slug.md
wiki/                # LLM이 컴파일한 위키 (지식의 주제별 구분)
├── index.md         # 전 페이지 카탈로그 — 모든 ingest마다 갱신
├── log.md           # append-only 작업 기록
└── <topic>/         # 주제 디렉터리 (영문 kebab-case) — ingest 시 필요하면 신설
    └── concept-name.md
```

## 아티클 규약

- 본문은 **한국어**, 기술 용어는 영문 병기. 파일명/슬러그/topic명은 **영문 kebab-case**.
- 개념(concept)당 1페이지. 한 페이지가 두 주제로 비대해지면 분할한다.
- 슬러그는 한 번 정하면 변경 금지. 중복 개념 발견 시 한쪽을 리다이렉트 스텁(본문에 `[[새-슬러그]]`로 안내만 남김)으로 만들어 병합한다.
- 상호참조는 `[[wikilink]]` 형식 (Obsidian 호환). 다른 topic의 페이지도 파일명만으로 링크한다.
- 주장에는 출처를 표기한다: 본문 또는 frontmatter의 `sources`로 어느 raw/ 파일에서 왔는지 추적 가능해야 한다.
- 새 정보가 기존 내용과 **모순**되면 덮어쓰지 않는다. 아래 형식으로 병기하고 lint에서 해소한다:

  ```markdown
  > ⚠️ 상충: [raw/notes/2026-06-11-example.md]는 X라고 하나, 기존 내용은 Y라고 함. 확인 필요.
  ```

### Frontmatter (필수)

```yaml
---
tags: [topic-name, domain-tag]
created: 2026-06-11
updated: 2026-06-11
sources:
  - raw/notes/2026-06-11-example.md
---
```

## log.md 규약

- append-only. 기존 엔트리는 절대 수정하지 않는다.
- 엔트리 형식: `## [YYYY-MM-DD] <operation> | <제목>` — operation은 `ingest` | `lint` | `query-filed` (저장소 초기화 시 1회에 한해 `init`).
- 엔트리 본문에 변경된 파일 목록을 적는다.
- `grep "^## \[" wiki/log.md` 로 파싱 가능해야 한다.

## 워크플로

세 가지 작업은 프로젝트 스킬로 정의되어 있다. 해당 요청이 오면 반드시 스킬을 사용한다:

- **/wiki-ingest** — 새 소스를 위키로 컴파일. "이거 위키에 넣어줘", "정리해줘" 류의 요청 포함.
- **/wiki-query** — 위키 기반 질의응답. "위키에서 찾아줘", "내가 전에 뭐라고 했지?" 류의 요청 포함.
- **/wiki-lint** — 위키 건강 검진. 모순·고아 페이지·끊어진 링크 정리.

## 운영 원칙

- ingest는 한 번에 소스 1개. 처리 후 변경 사항을 사용자에게 요약 보고한다.
- 작업 단위(ingest 1건, lint 1회)로 git 커밋한다. 커밋 메시지는 log.md 엔트리와 대응시킨다.
- 새 페이지 ~20개마다 /wiki-lint 실행을 사용자에게 권한다.
- 검색 인프라(임베딩, BM25)는 도입하지 않는다. index.md 탐색으로 충분하지 않게 되면 그때 논의한다.
