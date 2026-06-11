# LLM Wiki 저장소 구축 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Karpathy LLM Wiki 패턴에 따라 raw/ + wiki/ + CLAUDE.md 스키마 + 3개 워크플로 스킬을 갖춘 지식 베이스 저장소를 구축한다.

**Architecture:** 3계층 — 불변 원본(`raw/`), LLM 전유 위키(`wiki/`), 규약을 담은 스키마(`CLAUDE.md`). 워크플로(ingest/query/lint)는 `.claude/skills/`의 프로젝트 스킬로 코드화한다. 코드 없는 마크다운/설정 저장소이므로 검증은 파일 존재·내용 확인 커맨드와 샘플 ingest 시나리오로 수행한다.

**Tech Stack:** Markdown, git, Claude Code 프로젝트 스킬, Obsidian (뷰어)

**스펙:** `docs/superpowers/specs/2026-06-11-llm-wiki-design.md`

---

### Task 1: .gitignore + 디렉터리 골격

**Files:**
- Create: `.gitignore`
- Create: `raw/assets/.gitkeep`, `raw/notes/.gitkeep`, `raw/clips/.gitkeep`

- [ ] **Step 1: .gitignore 작성**

`.gitignore` 내용 (Obsidian의 기기별 상태 파일만 제외하고, vault 설정은 버전 관리에 포함):

```gitignore
# Obsidian per-device state
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/cache/

# OS
.DS_Store
Thumbs.db
```

- [ ] **Step 2: 디렉터리 골격 생성**

Run (PowerShell):
```powershell
New-Item -ItemType Directory -Force raw\assets, raw\notes, raw\clips | Out-Null
New-Item -ItemType File raw\assets\.gitkeep, raw\notes\.gitkeep, raw\clips\.gitkeep | Out-Null
```

- [ ] **Step 3: 검증**

Run: `git status --short`
Expected: `.gitignore`와 3개 `.gitkeep`이 untracked로 표시됨.

- [ ] **Step 4: Commit**

```powershell
git add .gitignore raw
git commit -m "chore: 저장소 골격 및 .gitignore 추가"
```

---

### Task 2: CLAUDE.md 스키마 작성

**Files:**
- Create: `CLAUDE.md`

- [ ] **Step 1: CLAUDE.md 작성**

전체 내용:

````markdown
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
````

- [ ] **Step 2: 검증**

Run: `grep -c "wiki-ingest\|wiki-query\|wiki-lint" CLAUDE.md` (Bash) 또는 `(Select-String -Path CLAUDE.md -Pattern "wiki-(ingest|query|lint)").Count` (PowerShell)
Expected: 3개 워크플로 모두 언급됨 (count ≥ 3).

- [ ] **Step 3: Commit**

```powershell
git add CLAUDE.md
git commit -m "feat: 위키 스키마(CLAUDE.md) 정의"
```

---

### Task 3: wiki/ 시드 파일 (index.md, log.md)

**Files:**
- Create: `wiki/index.md`
- Create: `wiki/log.md`

- [ ] **Step 1: wiki/index.md 작성**

```markdown
---
created: 2026-06-11
updated: 2026-06-11
---

# 위키 인덱스

모든 위키 페이지의 카탈로그. 모든 ingest마다 갱신된다.
형식: `- [[파일명]] — 한 줄 요약` (topic별 섹션으로 구분)

*(아직 페이지가 없다. 첫 /wiki-ingest 시 topic 섹션과 함께 추가된다.)*
```

- [ ] **Step 2: wiki/log.md 작성**

```markdown
# 작업 로그

append-only. 형식: `## [YYYY-MM-DD] <operation> | <제목>` (operation: ingest | lint | query-filed)

## [2026-06-11] init | 위키 초기화

- 저장소 구조, CLAUDE.md 스키마, 워크플로 스킬 생성.
```

- [ ] **Step 3: 검증**

Run: `grep "^## \[" wiki/log.md` (Bash) 또는 `Select-String -Path wiki\log.md -Pattern "^## \["` (PowerShell)
Expected: `## [2026-06-11] init | 위키 초기화` 1줄 출력 — 로그 파싱 규약 동작 확인.

- [ ] **Step 4: Commit**

```powershell
git add wiki
git commit -m "feat: 위키 시드 파일(index.md, log.md) 추가"
```

---

### Task 4: /wiki-ingest 스킬

**Files:**
- Create: `.claude/skills/wiki-ingest/SKILL.md`

- [ ] **Step 1: SKILL.md 작성**

````markdown
---
name: wiki-ingest
description: 새 소스(raw/ 파일 또는 대화로 전달된 내용)를 위키로 컴파일한다. "위키에 넣어줘", "이거 정리해줘", "오늘 한 일 기록해줘" 류의 요청, 또는 raw/에 새 파일이 추가되었을 때 사용.
---

# Wiki Ingest

새 소스 1개를 위키로 컴파일한다. **한 번에 소스 1개만** 처리한다. 여러 소스가 주어지면 순서대로 1개씩, 각각 아래 전 단계를 거친다.

## 절차

1. **소스 확정**
   - raw/에 이미 있는 파일이면 그대로 사용.
   - 대화 내용·메모로 전달되었으면 먼저 `raw/notes/YYYY-MM-DD-slug.md`로 저장한다 (오늘 날짜, 내용을 요약한 영문 kebab-case 슬러그). 원문을 충실히 보존하고 임의로 윤색하지 않는다.
   - 웹 URL이면 본문을 가져와 `raw/clips/YYYY-MM-DD-slug.md`로 저장한다 (출처 URL을 frontmatter에 기록).

2. **읽기 및 주제 분류**
   - 소스를 정독하고 `wiki/index.md`를 읽어 기존 topic·페이지를 파악한다.
   - 기존 topic 재사용을 우선하고, 맞는 것이 없을 때만 새 topic 디렉터리를 만든다.

3. **개념 페이지 생성/갱신**
   - 소스에서 다루는 개념별로 `wiki/<topic>/<concept>.md`를 생성하거나 갱신한다. 1개 소스가 여러 페이지를 건드릴 수 있다.
   - CLAUDE.md의 아티클 규약(frontmatter, 한국어 본문, 출처 표기, [[wikilink]])을 따른다.
   - 기존 페이지와 모순되는 내용은 덮어쓰지 말고 `> ⚠️ 상충:` 블록으로 병기한다.
   - 관련 페이지 간 [[wikilink]] 상호참조를 양방향으로 갱신한다.

4. **인덱스/로그 갱신**
   - `wiki/index.md`에 새/변경 페이지를 반영한다 (topic 섹션, 한 줄 요약).
   - `wiki/log.md`에 `## [YYYY-MM-DD] ingest | <소스 제목>` 엔트리를 append하고 변경 파일 목록을 적는다.

5. **커밋 및 보고**
   - 변경 전체를 1커밋: `git add raw wiki && git commit -m "ingest: <소스 제목>"`
   - 사용자에게 보고: 어떤 페이지가 생성/갱신되었는지, 발견한 상충, 제안할 후속 질문.

## 금지 사항

- raw/ 기존 파일 수정·삭제.
- index.md / log.md 갱신 생략.
- 소스에 없는 내용을 출처 없이 단정적으로 추가 (추론은 "추정:"으로 명시).
````

- [ ] **Step 2: 검증**

Run: `Test-Path .claude\skills\wiki-ingest\SKILL.md`
Expected: `True`. 또한 frontmatter에 `name:`과 `description:`이 있는지 확인.

- [ ] **Step 3: Commit**

```powershell
git add .claude/skills/wiki-ingest
git commit -m "feat: /wiki-ingest 스킬 추가"
```

---

### Task 5: /wiki-query 스킬

**Files:**
- Create: `.claude/skills/wiki-query/SKILL.md`

- [ ] **Step 1: SKILL.md 작성**

````markdown
---
name: wiki-query
description: 위키 지식 베이스 기반 질의응답. "위키에서 찾아줘", "내가 전에 뭐라고 정리했지?", "X에 대해 아는 것 알려줘" 류의 질문에 사용.
---

# Wiki Query

위키를 근거로 질문에 답한다. 위키에 없는 내용은 없다고 말한다 — 일반 지식으로 메꾸지 않는다 (일반 지식을 보태야 한다면 위키 내용과 명확히 구분해서 표시).

## 절차

1. **탐색**: `wiki/index.md`에서 관련 페이지를 찾는다. 부족하면 `wiki/` 전체를 Grep으로 검색하고, 필요 시 frontmatter의 `sources`를 따라 raw/ 원문까지 읽는다.

2. **답변 종합**: 출처를 명시하며 답한다 — 어느 위키 페이지([[wikilink]])와 어느 raw/ 파일에 근거했는지. 상충(⚠️) 블록이 있는 내용은 상충 사실을 함께 알린다.

3. **파일링 제안**: 답변이 여러 페이지를 종합한 **새로운** 내용이면, 위키에 새 페이지로 저장할지 사용자에게 제안한다. 동의하면:
   - `wiki/<topic>/<concept>.md` 생성 (아티클 규약 준수, sources에는 근거 위키 페이지가 아닌 원 raw/ 파일들을 기재)
   - `wiki/index.md` 갱신
   - `wiki/log.md`에 `## [YYYY-MM-DD] query-filed | <제목>` append
   - 커밋: `git add wiki && git commit -m "query-filed: <제목>"`
````

- [ ] **Step 2: 검증**

Run: `Test-Path .claude\skills\wiki-query\SKILL.md`
Expected: `True`.

- [ ] **Step 3: Commit**

```powershell
git add .claude/skills/wiki-query
git commit -m "feat: /wiki-query 스킬 추가"
```

---

### Task 6: /wiki-lint 스킬

**Files:**
- Create: `.claude/skills/wiki-lint/SKILL.md`

- [ ] **Step 1: SKILL.md 작성**

````markdown
---
name: wiki-lint
description: 위키 건강 검진 — 모순, 낡은 정보, 고아 페이지, 끊어진 링크, 인덱스 불일치를 찾아 정리한다. "위키 정리해줘", "lint 돌려줘" 요청 시, 또는 새 페이지 ~20개 추가 후 사용.
---

# Wiki Lint

위키 전체의 데이터 무결성을 점검하고 수정한다.

## 점검 항목

1. **상충 블록**: `Grep "⚠️ 상충" wiki/` — 각 건에 대해 raw/ 원문을 재확인해 해소 가능하면 해소하고, 판단이 필요하면 사용자에게 질문 목록으로 제시.
2. **끊어진 [[wikilink]]**: 모든 위키 페이지의 `[[...]]` 대상 파일이 실제 존재하는지 확인. 없는 링크는 페이지 신설 후보 또는 오타.
3. **고아 페이지**: 어떤 페이지에서도 링크되지 않고 index.md에만 있는 페이지 — 관련 페이지에서 링크를 추가하거나 병합 검토.
4. **index.md 정합성**: 실제 `wiki/<topic>/*.md` 파일 목록과 index.md 항목이 일치하는지 (누락/유령 항목).
5. **frontmatter 규약 위반**: tags/created/updated/sources 필수 필드 누락.
6. **비대 페이지**: 두 가지 이상의 주제를 다루게 된 페이지 — 분할 제안.
7. **낡은 정보**: 날짜에 민감한 단정("최신", "현재") 중 오래된 것 — 사용자에게 확인 요청 또는 시점 명시로 수정.

## 절차

1. 위 항목을 모두 점검하고 발견 사항을 보고한다 (자동 수정 가능 / 사용자 판단 필요로 구분).
2. 자동 수정 가능 건을 적용한다. 사용자 판단 필요 건은 질문한다.
3. `wiki/log.md`에 `## [YYYY-MM-DD] lint | <요약>` append (발견/수정 건수 포함).
4. 커밋: `git add wiki && git commit -m "lint: <요약>"`

## 금지 사항

- raw/ 수정.
- 사용자 확인 없이 페이지 삭제 (병합 시에도 리다이렉트 스텁을 남긴다).
````

- [ ] **Step 2: 검증**

Run: `Test-Path .claude\skills\wiki-lint\SKILL.md`
Expected: `True`.

- [ ] **Step 3: Commit**

```powershell
git add .claude/skills/wiki-lint
git commit -m "feat: /wiki-lint 스킬 추가"
```

---

### Task 7: README.md (사용 안내)

**Files:**
- Create: `README.md`

- [ ] **Step 1: README.md 작성**

````markdown
# llm-wiki

LLM(Claude)이 작성·유지보수하는 개인 지식 베이스. [Karpathy의 LLM Wiki 패턴](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 기반.

## 사용법

이 디렉터리에서 Claude Code를 열고:

| 하고 싶은 것 | 명령 |
|---|---|
| 메모·대화·아티클을 위키에 넣기 | `/wiki-ingest` (또는 "이거 위키에 정리해줘") |
| 위키에 물어보기 | `/wiki-query` (또는 "위키에서 X 찾아줘") |
| 위키 정리·건강 검진 | `/wiki-lint` (새 페이지 ~20개마다 권장) |

웹 아티클은 Obsidian Web Clipper로 `raw/clips/`에 저장한 뒤 ingest한다.

## 구조

- `raw/` — 원본 소스 (불변, 사람이 큐레이션)
- `wiki/` — Claude가 컴파일한 위키 (사람은 읽기만)
- `CLAUDE.md` — 위키 스키마 (규약의 단일 출처)

## Obsidian 설정

vault로 이 폴더를 연다. Settings → Files & Links에서:
- Default location for new attachments: `raw/assets/`
- Use [[Wikilinks]]: 켜기
````

- [ ] **Step 2: Commit**

```powershell
git add README.md
git commit -m "docs: README 추가"
```

---

### Task 8: E2E 검증 — 샘플 ingest → query → lint

이 태스크는 스펙 §8의 검증 기준을 수행한다. 샘플 소스는 실제 사용자 데이터가 아니므로 검증 후 **되돌린다**.

- [ ] **Step 1: 검증용 브랜치 생성**

```powershell
git checkout -b verify-e2e
```

- [ ] **Step 2: 샘플 소스 작성**

`raw/notes/2026-06-11-llm-wiki-pattern.md`:

```markdown
---
created: 2026-06-11
type: note
---

# LLM Wiki 패턴 메모

Karpathy가 2026년 4월에 제안한 패턴. LLM이 raw 소스를 읽고 위키를 컴파일·유지보수한다.
RAG와 달리 위키는 누적되는 영속 산출물이다. 사람은 소스 큐레이션과 질문, LLM은 나머지 전부를 맡는다.
~100개 소스, 40만 단어 규모까지는 index.md 탐색만으로 충분하고 임베딩 RAG가 필요 없다.
```

- [ ] **Step 3: /wiki-ingest 스킬 절차를 그대로 수행**

wiki-ingest SKILL.md의 절차 1~5를 수행한다. Expected 산출물:
- `wiki/<topic>/` 아래 1개 이상의 개념 페이지 (frontmatter에 sources로 위 raw 파일 경로 포함)
- `wiki/index.md`에 해당 페이지 항목 추가
- `wiki/log.md`에 `## [2026-06-11] ingest | ...` 엔트리 추가
- ingest 커밋 1개

- [ ] **Step 4: 산출물 검증**

Run: `grep -l "raw/notes/2026-06-11-llm-wiki-pattern.md" wiki/*/*.md` (Bash)
Expected: 생성된 개념 페이지 경로 출력 (sources 추적 동작 확인).

Run: `grep "^## \[" wiki/log.md | tail -1` (Bash)
Expected: `## [2026-06-11] ingest | ...` 라인.

- [ ] **Step 5: /wiki-query 검증**

wiki-query SKILL.md 절차로 "LLM wiki 패턴에서 사람과 LLM의 역할 분담은?"에 답해본다.
Expected: 생성된 위키 페이지를 출처로 명시한 한국어 답변.

- [ ] **Step 6: /wiki-lint 검증**

wiki-lint SKILL.md 절차를 수행한다.
Expected: 끊어진 링크·인덱스 불일치 0건 (있다면 스킬 또는 ingest 산출물의 버그 — 수정 후 재실행).

- [ ] **Step 7: 검증 결과 보고 후 샘플 데이터 되돌리기**

검증 결과를 사용자에게 보고한 뒤:

```powershell
git checkout master
git branch -D verify-e2e
```

Expected: master에는 샘플 데이터 없음. `git status` clean. (사용자가 샘플을 남기길 원하면 merge로 대체.)
