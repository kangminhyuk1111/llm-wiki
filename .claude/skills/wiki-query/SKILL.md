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
