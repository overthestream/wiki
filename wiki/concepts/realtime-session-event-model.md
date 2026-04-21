---
title: Realtime Session/Event Model
type: concept
tags: [ai, theory, voice-ai, realtime, api-design]
sources: ["raw/articles/Realtime conversations  OpenAI API.md"]
created: 2026-04-21
updated: 2026-04-21
---

# Realtime Session/Event Model

## Definition

**Realtime Session/Event Model**은 OpenAI Realtime API가 채택한, **상태 있는 양방향 이벤트 스트림으로 음성 대화를 표현**하는 설계 모델이다. HTTP 요청/응답처럼 호출 하나에 응답 하나가 묶이는 대신, 클라이언트는 일련의 **client events**를 보내고 서버는 생애주기·스트리밍 **server events**를 방출한다. 대화의 상태·음성·도구·prompt는 모두 "Session" 오브젝트에 속한다.

## Key Ideas

### 3계층 상태 구조

1. **Session**: 설정 레벨. 모델(`gpt-realtime`), voice, 출력 모달리티, `prompt`, `audio.input/output`, `turn_detection`, `tools` 등.
2. **Conversation**: 사용자 입력 Item + 모델 응답 Item의 시퀀스. 기본 conversation 하나가 세션에 묶이지만, out-of-band 응답으로 우회 가능.
3. **Response**: 모델이 생성하는 개별 산출물. 같은 세션 안에 여러 response가 동시 생성될 수 있음.

### 핵심 client events

- `session.update` — 세션 설정 변경. 대부분 언제든지 가능. **voice는 첫 오디오 방출 이후 변경 불가**, 세션 최대 60분 제한.
- `conversation.item.create` — 사용자 텍스트/오디오/이미지/function_call_output을 conversation에 추가.
- `response.create` — 응답 생성 트리거. `output_modalities`, `conversation: "none"`, `metadata`, 커스텀 `input` 배열, per-response `tools`/`tool_choice` 지정.
- `input_audio_buffer.append/commit/clear` — WebSocket에서 base64 PCM 청크 송신·커밋·비우기.
- `conversation.item.truncate` — 재생되지 못한 모델 오디오를 제거(인터럽션 정리).
- `response.cancel` — 진행 중 응답 취소.

### 핵심 server events

- **수명주기**: `session.created`, `session.updated`, `response.created`, `response.done`, `rate_limits.updated`.
- **오디오 IO**: `input_audio_buffer.speech_started/stopped/committed`, `response.output_audio.delta/done`, `response.output_audio_transcript.delta/done`.
- **텍스트**: `response.output_text.delta/done`, `response.content_part.added/done`, `response.output_item.added/done`.
- **도구**: `response.function_call_arguments.delta/done`; MCP는 `mcp_list_tools.in_progress/completed/failed`, `response.mcp_call_arguments.delta/done`, `response.mcp_call.in_progress/failed`.
- **오류**: `error`.

### 요청-오류 상관은 `event_id`로

HTTP처럼 암묵적 request/response 쌍이 없으므로, 클라이언트가 직접 각 이벤트에 `event_id`를 넣고 server의 `error` 이벤트가 그 id를 되돌려준다. **이 ID 규칙을 무시하면 디버깅이 급격히 어려워진다**.

### Out-of-Band 응답

`response.create`의 `response.conversation: "none"`으로, 기본 conversation 상태를 건드리지 않는 별도 응답을 생성할 수 있다. 분류기·RAG·검열 같은 **부가 작업**에 쓰인다. `metadata`로 나중에 `response.done` 이벤트를 식별. `input: []`으로 "기존 컨텍스트 완전 무시"도 가능.

### Turn Detection 모드

- **기본**: VAD 켜짐. 사용자 발화 감지 → 자동 전사 → 자동 `response.create`.
- **VAD + 수동 응답**: `turn_detection.create_response=false`로 응답만 수동. RAG/모더레이션에서 유용.
- **VAD 끔 (`turn_detection: null`)**: Push-to-Talk. 클라이언트가 `input_audio_buffer.commit` + `response.create`를 직접 발사.

### 인터럽션/Truncation

VAD 켜진 상태에서 사용자가 끼어들면 서버가 진행 중 응답을 취소한다(`response.cancelled`). 이후 **재생되지 않은 부분을 conversation에서 제거**하지 않으면 모델 내부 기록과 실제 들린 음성이 어긋난다.

- **WebRTC/SIP**: 서버가 출력 버퍼를 쥐고 있으므로 자동 truncate.
- **WebSocket**: 클라이언트가 `conversation.item.truncate`를 `audio_end_ms`와 함께 발사해야 함. 전사도 오디오 잘린 지점에 맞춰 텍스트가 잘린다(정확한 정렬은 불가).

## Examples

- **Push-to-Talk**: `turn_detection: null` → 키다운 시 `input_audio_buffer.clear` + 이전 응답이 있으면 `response.cancel` → 키업 시 `commit` + `response.create`.
- **Out-of-band 분류기**: 대화 중간에 `response.create` + `conversation: "none"`, `metadata: {topic: "classification"}`로 라우팅 결정용 텍스트 응답만 뽑고, 기본 conversation에는 반영하지 않음.
- **커스텀 컨텍스트**: `input: [{type: "item_reference", id: ...}, {type: "message", ...}]`로 일부 이전 아이템 + 새 메시지로 구성된 축약 context 요청.

## Related

- [[entities/openai-realtime-api|OpenAI Realtime API]]
- [[concepts/mcp-model-context-protocol|MCP (Model Context Protocol)]]
- [[concepts/sideband-control-channel|Sideband Control Channel]]
- [[concepts/voice-ai-agent|Voice AI Agent]]
- [[summaries/2026-04-21-openai-realtime-conversations|Realtime conversations 요약]]
