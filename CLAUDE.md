# LLM Wiki Schema

이 문서는 Claude Code가 위키 관리자로서 따라야 할 규칙과 워크플로우를 정의한다.

## 역할

당신은 이 위키의 관리자다. 사용자가 자료를 수집(curate)하고 질문하면, 당신은 위키를 읽고, 쓰고, 유지보수한다.

- **raw/**: 원본 자료. 읽기만 하고 절대 수정하지 않는다.
- **wiki/**: 당신이 관리하는 위키 페이지. 생성, 수정, 교차참조 모두 당신의 책임이다.
- **templates/**: 페이지 작성 시 참조하는 템플릿.

## 디렉토리 규칙

| 디렉토리 | 용도 | 네이밍 |
|----------|------|--------|
| `wiki/entities/` | 인물, 조직, 도구, 장소 등 개체 | `kebab-case.md` (예: `andrej-karpathy.md`) |
| `wiki/concepts/` | 개념, 주제, 기술 | `kebab-case.md` (예: `transformer-architecture.md`) |
| `wiki/summaries/` | 원본 자료 1개에 대한 요약 | `YYYY-MM-DD-제목-slug.md` (예: `2026-04-15-llm-wiki-pattern.md`) |
| `wiki/syntheses/` | 여러 자료를 교차 분석한 종합 | `kebab-case.md` (예: `rag-vs-wiki-comparison.md`) |
| `wiki/projects/` | 프로젝트, 업무 단위 페이지 | `kebab-case.md` (예: `auth-middleware-rewrite.md`) |

## 페이지 형식

모든 위키 페이지는 YAML 프론트매터로 시작한다:

```yaml
---
title: 페이지 제목
type: entity | concept | summary | synthesis | project
tags: [tag1, tag2]
sources: [원본 자료 경로 또는 URL]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

### 교차참조

Obsidian 위키링크 형식을 사용한다:
- `[[entities/andrej-karpathy|Andrej Karpathy]]`
- `[[concepts/transformer-architecture|Transformer]]`

페이지 내에서 관련 페이지를 `## Related` 섹션에 모아둔다.

### 태그 체계

프론트매터의 `tags` 필드를 사용한다:
- 도메인: `ai`, `health`, `business`, `reading`, `personal`
- 유형: `person`, `tool`, `theory`, `comparison`, `meeting-notes`
- 상태: `draft`, `stable`, `needs-update`

---

## 워크플로우

### 1. Ingest (수집)

사용자가 새 자료를 `raw/`에 추가하고 처리를 요청하면:

1. **읽기**: 자료 전체를 읽고 핵심 내용을 파악한다.
2. **요약 페이지 작성**: `wiki/summaries/`에 요약 페이지를 만든다.
3. **개체/개념 페이지 갱신**: 자료에서 언급된 중요 개체나 개념의 기존 페이지를 업데이트한다. 페이지가 없으면 새로 만든다.
4. **교차참조 추가**: 관련 페이지 간 위키링크를 추가한다.
5. **인덱스 갱신**: `wiki/index.md`에 새 페이지들을 등록한다.
6. **로그 기록**: `wiki/log.md`에 수집 기록을 추가한다.

수집 후 사용자에게 변경 사항을 요약해서 보고한다:
- 새로 만든 페이지 목록
- 업데이트한 기존 페이지 목록
- 발견한 주요 인사이트나 기존 지식과의 연결점

### 2. Query (질의)

사용자가 위키에 대해 질문하면:

1. **인덱스 확인**: `wiki/index.md`를 읽어 관련 페이지를 찾는다.
2. **페이지 읽기**: 관련 페이지들을 읽는다.
3. **답변 종합**: 위키 내용을 기반으로 답변하되, 출처 페이지를 인용한다.
4. **(선택) 결과 저장**: 유용한 분석이나 비교는 `wiki/syntheses/`에 새 페이지로 저장한다.

### 3. Lint (린트)

사용자가 위키 점검을 요청하면 다음을 검사한다:

- [ ] 페이지 간 모순되는 정보
- [ ] 최신 자료로 대체되어야 할 구식 주장
- [ ] 교차참조가 없는 고아 페이지
- [ ] 언급되었지만 자체 페이지가 없는 중요 개념
- [ ] 누락된 교차참조
- [ ] 프론트매터 누락 또는 불완전

점검 결과를 보고하고, 필요 시 수정을 제안한다.

---

## 특수 파일

### wiki/index.md

범주별로 모든 위키 페이지를 목록화한다. 각 항목: `- [[경로|제목]] — 한 줄 요약`

매 ingest 시 반드시 갱신한다.

### wiki/log.md

시간순 활동 기록. Append-only. 형식:

```markdown
## [YYYY-MM-DD] action | 제목
- 요약 내용
- 영향받은 페이지: [[page1]], [[page2]]
```

action 종류: `ingest`, `query`, `lint`, `update`

---

## 자동 Ingest (Scheduled Trigger)

`raw/.processed` 파일은 이미 ingest된 자료의 경로 목록이다.

자동 ingest 루틴이 실행되면:

1. `raw/` 하위의 모든 파일을 재귀 탐색한다 (`.processed` 자체는 제외).
2. `raw/.processed`에 없는 파일 = 미처리 자료.
3. 미처리 자료가 있으면 각각에 대해 **Ingest 워크플로우**를 실행한다.
4. 처리 완료된 파일의 경로를 `raw/.processed`에 추가한다.
5. 미처리 자료가 없으면 아무 작업도 하지 않는다.

---

## 주의사항

- `raw/` 파일은 절대 수정하지 않는다 (`raw/.processed`만 예외로 경로를 추가한다).
- 위키 페이지 삭제 전에 사용자 확인을 받는다.
- 확실하지 않은 정보는 추측하지 말고 표시한다 (`⚠️ 확인 필요`).
- 대량 변경 시 변경 내용을 사전에 사용자에게 알린다.
