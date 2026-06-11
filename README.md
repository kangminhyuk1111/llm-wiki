<p align="center">
  <br>
  ◯ ──────────────── ◯
  <br><br>
  <strong>📚 L L M &nbsp; W I K I</strong>
  <br><br>
  <em>사람은 질문하고, LLM이 위키를 가꾼다.</em>
  <br><br>
  ◯ ──────────────── ◯
  <br>
</p>

<p align="center">
  <a href="https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f">
    <img src="https://img.shields.io/badge/pattern-Karpathy%20LLM%20Wiki-blue" alt="Pattern">
  </a>
  <img src="https://img.shields.io/badge/maintained%20by-Claude%20Code-d97757" alt="Maintained by Claude Code">
  <img src="https://img.shields.io/badge/viewer-Obsidian-7c3aed" alt="Viewer: Obsidian">
  <img src="https://img.shields.io/badge/format-Markdown-lightgrey" alt="Format: Markdown">
</p>

<p align="center">
  <a href="#-어떻게-동작하나">동작 원리</a> ·
  <a href="#-사용법">사용법</a> ·
  <a href="#-저장소-구조">구조</a> ·
  <a href="#-규칙-세-가지">규칙</a> ·
  <a href="#-obsidian-설정">Obsidian</a>
</p>

---

LLM(Claude)이 **작성·유지보수하는** 개인 지식 베이스입니다.
RAG처럼 매 질문마다 원본을 다시 뒤지는 대신, LLM이 위키를 한 번 컴파일하고 계속 가꿉니다.
질문할수록 답이 위키로 쌓이는, **누적되는 지식 자산**입니다.

> *"Obsidian is the IDE; the LLM is the programmer; the wiki is the codebase."*
> — Andrej Karpathy, [LLM Wiki idea file](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)

## 🔄 어떻게 동작하나

```
   메모 · 대화 · 웹 아티클
            │
            ▼  /wiki-ingest
   ┌─────────────────┐         ┌──────────────────────┐
   │      raw/       │ ──────▶ │        wiki/         │
   │  불변 원본 보관  │  컴파일  │  개념 페이지 + 인덱스  │
   │  (사람이 큐레이션)│         │   (LLM이 작성·관리)   │
   └─────────────────┘         └──────────────────────┘
                                    │           ▲
                       /wiki-query  │           │  /wiki-lint
                                    ▼           │
                            출처 달린 답변    모순·고아 페이지
                            (위키로 재축적)     자동 정리
```

## 🚀 사용법

이 디렉터리에서 **Claude Code**를 열고:

| 하고 싶은 것 | 명령 | 이렇게 말해도 됨 |
|:---|:---|:---|
| 📥 메모·대화·아티클을 위키에 넣기 | `/wiki-ingest` | "이거 위키에 정리해줘" |
| 🔍 위키에 물어보기 | `/wiki-query` | "위키에서 X 찾아줘" |
| 🧹 위키 정리·건강 검진 | `/wiki-lint` | "위키 정리해줘" |

> [!TIP]
> 웹 아티클은 **Obsidian Web Clipper**로 `raw/clips/`에 저장한 뒤 ingest하면 됩니다.
> `/wiki-lint`는 새 페이지가 ~20개 쌓일 때마다 한 번 돌려주는 것을 권장합니다.

## 🗂 저장소 구조

```
llm-wiki/
├── CLAUDE.md            # 위키 스키마 — 규약의 단일 출처
├── raw/                 # 불변 원본 (사람이 큐레이션)
│   ├── notes/           #   메모, 대화 정리, 운영 기록
│   ├── clips/           #   웹 아티클 (Obsidian Web Clipper)
│   └── assets/          #   이미지
└── wiki/                # LLM이 컴파일한 위키 (사람은 읽기만)
    ├── index.md         #   전 페이지 카탈로그
    ├── log.md           #   append-only 작업 로그
    └── <topic>/         #   주제별 개념 페이지
```

## 📏 규칙 세 가지

| # | 규칙 | 의미 |
|:-:|:---|:---|
| 1 | **`raw/`는 불변** | 원본은 진실의 원천. LLM은 읽기만 한다 |
| 2 | **`wiki/`는 LLM의 영역** | 직접 편집하지 않는다. 고치고 싶으면 LLM에게 지시 |
| 3 | **스키마는 함께 진화** | 규약이 바뀌면 반드시 `CLAUDE.md`에 반영 |

## 🪨 Obsidian 설정

이 폴더를 vault로 열고, 최초 1회 설정:

<details>
<summary><strong>Settings → Files & Links</strong></summary>

- **Default location for new attachments**: `raw/assets/`
- **Use [[Wikilinks]]**: 켜기

이후 그래프 뷰에서 위키 페이지 간 연결을 시각적으로 탐색할 수 있습니다.

</details>

---

<p align="center">
  <em>"위키가 죽는 이유는 유지보수 부담이 가치보다 빨리 자라기 때문이다.<br>
  LLM에게 유지보수 비용은 0에 가깝다 — 그래서 이 위키는 살아남는다."</em>
  <br><br>
  ◯ ──────────────── ◯
</p>
