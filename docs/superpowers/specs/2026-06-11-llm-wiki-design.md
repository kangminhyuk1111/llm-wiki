# LLM Wiki 설계 문서

- 날짜: 2026-06-11
- 상태: 승인됨
- 근거: Andrej Karpathy의 LLM Wiki idea file (https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)

## 1. 목적과 배경

이 저장소는 Karpathy가 제안한 "LLM Wiki" 패턴의 개인 지식 베이스다. 핵심 아이디어:

- LLM이 원본 소스(raw)를 읽고 위키(마크다운 파일 모음)를 **점진적으로 컴파일·유지보수**한다.
- RAG처럼 매 질의마다 원본을 재해석하는 것이 아니라, 위키는 **누적되는 영속 산출물**이다.
- 역할 분담: **사람** = 소스 큐레이션, 질문, 판단 / **LLM** = 작성, 요약, 상호참조, 인덱싱, 정리 등 나머지 전부.
- 사용자는 1인 (강민혁). 업무 지식과 개인 지식을 한 저장소에서 주제(topic) 단위로 함께 다룬다.
- 뷰어는 Obsidian (vault = 저장소 루트). 에이전트는 Claude Code.

## 2. 저장소 구조

```
llm-wiki/                      # = Obsidian vault 루트
├── CLAUDE.md                  # 스키마: 위키 규약 + 워크플로 (LLM의 계약서)
├── .gitignore                 # .obsidian/workspace 등 제외
├── .claude/skills/
│   ├── wiki-ingest/SKILL.md   # /wiki-ingest 워크플로
│   ├── wiki-query/SKILL.md    # /wiki-query 워크플로
│   └── wiki-lint/SKILL.md     # /wiki-lint 워크플로
├── docs/                      # 메타 문서 (설계, 계획) — 위키 본체 아님
├── raw/                       # 불변 원본 — 사람이 큐레이션, LLM은 읽기만
│   ├── assets/                # 이미지 (Obsidian 첨부 폴더로 지정)
│   ├── notes/                 # 대화 정리, 메모, 생각, 운영 기록
│   │   └── YYYY-MM-DD-slug.md
│   └── clips/                 # 웹 아티클 (Obsidian Web Clipper 저장처)
│       └── YYYY-MM-DD-slug.md
└── wiki/                      # LLM 전유 영역 — 사람은 읽기만
    ├── index.md               # 전 페이지 카탈로그 (링크 + 한 줄 요약, 카테고리별)
    ├── log.md                 # append-only 기록
    └── <topic>/               # 주제 디렉터리 — 미리 정하지 않고 ingest 시 생성
        └── concept-name.md    # 개념당 1페이지
```

- `raw/`는 소스의 **형태**(notes/clips)로 구분하고, `wiki/`는 지식의 **주제**로 구분한다.
- 업무/개인 구분은 별도 최상위 분리 없이 topic 디렉터리와 태그로 자연스럽게 나뉜다.

## 3. 아티클 규약

- 본문은 한국어, 기술 용어는 영문 병기. 파일명/슬러그는 영문 kebab-case.
- YAML frontmatter 필수 필드:
  - `tags`: 주제·도메인 태그 목록
  - `created`, `updated`: ISO 날짜
  - `sources`: 근거가 된 raw/ 파일 경로 목록
- 상호참조는 `[[wikilink]]` 형식 (Obsidian 그래프 뷰 호환).
- 주장에는 출처를 표기한다 (어느 raw 파일에서 왔는지).
- 새 정보가 기존 내용과 모순되면 덮어쓰지 않고 `> ⚠️ 상충:` 인용 블록으로 병기하고, lint에서 해소한다.
- 슬러그는 한 번 정하면 변경하지 않는다. 중복 개념 발견 시 리다이렉트 스텁(원 슬러그 파일에 새 슬러그로의 `[[wikilink]]`만 남김)으로 병합한다.
- 개념(concept)당 1페이지. 한 페이지가 두 가지 주제로 비대해지면 분할한다.

## 4. 특수 파일

### index.md
- 모든 위키 페이지의 카탈로그: 링크 + 한 줄 요약, 카테고리(주제)별로 정리.
- 모든 ingest마다 갱신. ~수백 페이지 규모까지는 이 파일이 검색 인프라를 대체한다.

### log.md
- append-only. 엔트리 형식: `## [YYYY-MM-DD] <operation> | <제목>`
- operation은 `ingest`, `lint`, `query-filed`(질의 결과 파일링) 중 하나.
- `grep "^## \[" log.md | tail -5` 같은 단순 도구로 파싱 가능해야 한다.

## 5. 세 가지 워크플로

### /wiki-ingest
입력: raw/에 놓인 새 파일, 또는 대화로 전달된 내용(이 경우 먼저 raw/notes/에 파일화).
1. 소스를 읽고 주제 분류 (기존 topic 재사용 우선, 없으면 신설)
2. 관련 개념 페이지 생성/갱신 — 1개 소스가 여러 페이지를 건드릴 수 있음
3. 관련 페이지 간 `[[wikilink]]` 상호참조 갱신
4. index.md 갱신
5. log.md에 기록
6. 무엇이 바뀌었는지 사용자에게 요약 보고
- 한 번에 소스 1개씩 처리한다 (조용한 품질 저하 방지).

### /wiki-query
1. index.md에서 관련 페이지 탐색 → 페이지·필요 시 raw/ 원문 읽기
2. 출처(위키 페이지·raw 파일)와 함께 답변 종합
3. 답변이 위키에 없는 새로운 종합이라면, 사용자 동의 하에 위키 페이지로 파일링하고 log.md에 `query-filed` 기록

### /wiki-lint
1. 모순(⚠️ 상충 블록 포함), 낡은 정보, 고아 페이지(들어오는 링크 0), 끊어진 `[[wikilink]]`, index.md 누락/불일치, 비대해진 페이지 탐지
2. 발견 사항과 수정안을 보고한 뒤 적용
3. log.md에 기록
- 권장 주기: 새 페이지 ~20개 추가마다.

## 6. 운영 원칙 (CLAUDE.md에 명문화)

- `raw/`는 불변. LLM은 절대 수정·삭제하지 않는다.
- `wiki/`는 LLM이 작성·유지한다. 사용자는 직접 편집하지 않는다 (수정하고 싶으면 LLM에게 지시).
- ingest는 한 번에 1개 소스, 변경 요약을 반드시 보고한다.
- git으로 버전 관리. ingest/lint 작업 단위로 커밋한다.
- 스키마(CLAUDE.md)는 사용하면서 사람과 LLM이 함께 진화시킨다.

## 7. 범위 제외 (YAGNI)

다음은 이번에 만들지 않는다. index.md 방식이 한계에 도달하면(수백 페이지 이상) 그때 추가한다.

- 검색엔진 (qmd 등 BM25/벡터 검색)
- Dataview 대시보드
- 자동/주기 lint (cron)
- 임베딩 기반 RAG
- 파인튜닝, 합성 데이터 생성

## 8. 테스트/검증 기준

- 초기화 후: 샘플 소스 1개를 `/wiki-ingest`로 투입 → 개념 페이지 생성, index.md·log.md 갱신, Obsidian에서 `[[wikilink]]`와 그래프 뷰 정상 동작 확인.
- `/wiki-query`로 해당 내용 질의 → 출처 포함 답변 확인.
- `/wiki-lint` 실행 → 빈 위키에서 오탐 없이 통과 확인.
