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
