---
title: Realtime API with MCP 요약
type: summary
tags: [ai, voice-ai, openai, mcp, tool-use]
sources: ["raw/articles/Realtime API with MCP.md", "https://developers.openai.com/api/docs/guides/realtime-mcp"]
created: 2026-04-21
updated: 2026-04-21
---

# Realtime API with MCP 요약

## Source Info
- **출처**: OpenAI Developers Docs (`/api/docs/guides/realtime-mcp`)
- **저자**: OpenAI (공식 가이드)
- **날짜**: 2026-04-21 clipping

## Summary

이 문서는 [[entities/openai-realtime-api|OpenAI Realtime API]] 세션에 **원격 MCP 서버의 도구**를 붙이는 방법을 다룬다. 핵심 구조적 차이는 명확하다 — MCP tool은 Realtime API **서버가 직접 호출**하며, 클라이언트는 function calling처럼 `function_call_output`을 만들지 않는다. 대신 승인 요청(approval)을 받으면 `mcp_approval_response` item으로 답하고, lifecycle 이벤트를 관찰한다.

설정은 `session.tools`(세션 전역) 또는 `response.tools`(단일 턴)에 `type:"mcp"` + `server_label` + `server_url` 또는 `connector_id`를 담는 형태다. built-in 커넥터(예: `connector_googlecalendar`)의 경우 `connector_id`와 사용자 OAuth 토큰을 `authorization` 필드에 담는다. 보안을 위해 **`allowed_tools`로 표면을 좁히고, 쓰기성 액션은 `require_approval`을 유지**하라는 안내가 반복된다.

플로우는 `mcp_list_tools.in_progress` → `mcp_list_tools.completed`(또는 `failed`) → 모델의 `response.mcp_call_arguments.delta/done` → 필요 시 승인 왕복 → `response.mcp_call.in_progress` → 성공 시 `mcp_call` item이 `response.output_item.done`으로 확정되고, 최종 assistant message + `response.done`으로 턴 종료. 흔한 실패 원인(중복 label, server_url+connector_id 동시 지정, 누락, `headers.Authorization` 중복 등)과 server_label 재사용(session-scoped) 규칙도 정리되어 있다.

## Key Takeaways

- MCP tool은 **OpenAI 서버가 직접 실행**한다는 것이 function tool과의 결정적 차이. 클라이언트가 도구 실행 책임을 지지 않는다.
- **`type: "mcp"` + `server_label` + (`server_url` or `connector_id`)** 가 최소 스펙. connector는 토큰을 `authorization`에, `server_url` 방식은 직접 MCP 서버 URL을.
- **`allowed_tools`**로 도구 표면을 좁히는 것이 보안·비용·라우팅 품질 모두에 유리. Google Calendar 예시는 `["search_events", "read_event"]`만 노출해 읽기로 제한.
- Approval 누락이 가장 흔한 정지 원인. `mcp_approval_request`가 들어오면 `mcp_approval_response`로 명시적으로 처리해야 진행.
- **원격 MCP 서버는 전체 대화 컨텍스트를 자동으로 받지 않음**. 다만 모델이 tool call arguments에 담아 보낸 데이터는 본다.
- **server_label은 session-scoped 핸들**: 한 세션에 한 번만 전체 정의를 보내면 이후엔 label만으로 재참조. 새 세션에서는 다시 전체 정의 필요.
- **tool_choice: "required"** 사용 시, `mcp_list_tools.completed` 이전에 턴을 시작하면 호출 가능한 도구가 없어 실패. 동기화 필요.
- Validation 실패 흔한 패턴: 중복 `server_label`, `server_url`+`connector_id` 동시 지정, 커넥터에 `headers.Authorization` 중복.

## Quotes & References

- *"Your client does not run the remote tool and return a function_call_output. Instead, your client configures access, listens for MCP lifecycle events, and optionally sends an approval response if the server asks for one."*
- *"Keep the tool surface narrow with allowed_tools, and require approval for any action you would not auto-run."*
- Connector ID 예: `connector_googlecalendar`.

## Related

- [[entities/openai-realtime-api|OpenAI Realtime API]]
- [[concepts/mcp-model-context-protocol|MCP (Model Context Protocol)]]
- [[concepts/realtime-session-event-model|Realtime Session/Event Model]]
- [[concepts/multi-agent-voice-architecture|멀티 에이전트 보이스 아키텍처]]
