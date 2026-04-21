---
title: MCP (Model Context Protocol)
type: concept
tags: [ai, theory, protocol, tool-use, agent]
sources: ["raw/articles/Realtime API with MCP.md"]
created: 2026-04-21
updated: 2026-04-21
---

# MCP (Model Context Protocol)

## Definition

**MCP (Model Context Protocol)**는 AI 모델이 원격 서버에 호스팅된 도구(tool)와 리소스를 표준화된 방식으로 발견·호출할 수 있게 해주는 프로토콜이다. LLM에 제공되는 도구 목록을 사람이 코드로 하드코딩하는 대신, **"MCP 서버" 엔드포인트에 tool list를 맡기고 런타임에 import**하는 방식으로 도구 표면을 동적으로 구성한다. [[entities/openai-realtime-api|OpenAI Realtime API]]에서는 `type: "mcp"` tool로 1급 통합되어 있다.

## Key Ideas

### function tool과의 실행 주체 차이

- **function tool (전통 방식)**: 모델이 arguments를 내면 **클라이언트가 실행**하고 `function_call_output`을 회신. 제어·보안·비즈니스 로직이 전부 클라이언트에 있다.
- **MCP tool**: Realtime API **서버가 직접 원격 MCP 서버에 호출**을 던진다. 클라이언트는 approval 처리·결과 관찰만 한다. 즉, 클라이언트는 `function_call_output`을 만들 필요가 없다.

이 차이가 보안 모델과 에이전트 아키텍처를 바꾼다 — 도구 접근 권한이 클라이언트에서 중앙 서버로 옮겨간다.

### MCP tool의 구성 요소

- `type: "mcp"`
- `server_label` — 세션 내 안정적 handle.
- `server_url` 또는 `connector_id` — 직접 MCP 서버 URL vs. built-in 커넥터(예: `connector_googlecalendar`).
- `authorization` — OAuth access token 등. 커넥터는 이 필드에 토큰을 넣고 `headers.Authorization`는 쓰지 않는다.
- `allowed_tools` — 도구 표면 축소. 보안·비용·라우팅 품질 모두에 유리.
- `require_approval` — `"never"` vs. 승인 요구.
- `server_description` — 모델에게 서버 용도를 설명하는 힌트.

### Realtime MCP 이벤트 플로우

1. 클라이언트가 `session.update` 또는 `response.create`에 `type:"mcp"` tool을 등록.
2. 서버가 원격 서버에서 tool을 가져오는 동안 `mcp_list_tools.in_progress` → 완료 시 `mcp_list_tools.completed`. 실패 시 `mcp_list_tools.failed`.
3. 실제로 import된 tool 목록은 `conversation.item.done` 이벤트 중 `item.type === "mcp_list_tools"`에서 확인.
4. 사용자 발화/텍스트가 도착하고 응답이 생성되는 과정에서 모델이 MCP tool을 고르면 `response.mcp_call_arguments.delta/done` 방출.
5. **승인 필요 시** 서버가 `mcp_approval_request` item을 conversation에 추가 → 클라이언트가 `mcp_approval_response` item으로 answer해야 진행.
6. 실행 단계: `response.mcp_call.in_progress` → 성공 시 `response.output_item.done`(`item.type === "mcp_call"`), 실패 시 `response.mcp_call.failed`.

### 서버 정의 재사용 (session-scoped)

한 세션에서 `server_label` + `server_url`/`connector_id`로 정의를 한 번만 보내면, 이후 `response.create`에서는 **`server_label`만** 포함해 재사용할 수 있다. 세션이 바뀌면 다시 전체 정의를 보내야 한다.

### 보안·신뢰 경계

- 원격 MCP 서버는 전체 대화 컨텍스트를 자동으로 받지 못한다. **다만 모델이 tool call arguments에 담아 보낸 데이터는 본다.**
- **`allowed_tools`로 표면을 좁히는 것**과 **쓰기성 액션에 `require_approval`을 유지하는 것**이 기본 위생 조치.
- 커넥터에서는 `headers.Authorization`를 보내지 말 것 — validation 실패 사유.

### 흔한 실패 모드

- 중복 `server_label`을 같은 tools 배열에 두 번 등록.
- `server_url`과 `connector_id`를 동시에 지정.
- 최초 세션에서 둘 다 빼먹음.
- 유효하지 않은 `connector_id`.
- `authorization`과 `headers.Authorization` 중복 송신.
- `tool_choice: "required"`인데 아직 import된 tool이 없음.

## Examples

- **세션 전역 MCP**: 문서 검색 MCP 서버(`search_openai_docs`, `fetch_openai_doc`)를 세션 내내 허용.
- **1-turn MCP**: `response.tools`에만 MCP tool 객체를 넣어 한 턴만 외부 컨텍스트 허용.
- **Google Calendar 커넥터**: `connector_id: "connector_googlecalendar"` + 사용자 OAuth 토큰 + `allowed_tools: ["search_events", "read_event"]`로 읽기 전용 표면만 노출.

## Related

- [[entities/openai-realtime-api|OpenAI Realtime API]]
- [[concepts/realtime-session-event-model|Realtime Session/Event Model]]
- [[concepts/multi-agent-voice-architecture|멀티 에이전트 보이스 아키텍처]]
- [[concepts/voice-ai-agent|Voice AI Agent]]
- [[summaries/2026-04-21-openai-realtime-mcp|Realtime API with MCP 요약]]
