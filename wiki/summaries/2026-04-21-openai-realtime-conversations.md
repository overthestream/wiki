---
title: Realtime conversations (OpenAI Realtime API) 요약
type: summary
tags: [ai, voice-ai, openai, api, realtime]
sources: ["raw/articles/Realtime conversations  OpenAI API.md", "https://developers.openai.com/api/docs/guides/realtime-conversations"]
created: 2026-04-21
updated: 2026-04-21
---

# Realtime conversations (OpenAI Realtime API) 요약

## Source Info
- **출처**: OpenAI Developers Docs (`/api/docs/guides/realtime-conversations`)
- **저자**: OpenAI (공식 가이드)
- **날짜**: 2026-04-21 clipping

## Summary

이 가이드는 [[entities/openai-realtime-api|OpenAI Realtime API]]의 **세션 모델과 이벤트 플로우**를 처음부터 끝까지 훑는다. Realtime Session은 Session(설정)·Conversation(Item 시퀀스)·Response(모델 산출물)의 3계층으로 구성되며, 클라이언트의 **client events**와 서버의 **server events** 교환으로 상태가 진화한다는 골격을 제시한다. 세션 최대 길이 60분, voice는 첫 오디오 방출 이후 변경 불가 같은 제약도 여기서 나온다.

텍스트 IO부터 시작해 오디오 IO(WebRTC vs WebSocket 처리 차이), 이미지 입력, VAD 설정, Out-of-Band 응답, function calling, 오류 처리, 인터럽션/truncation, Push-to-Talk까지 각 기능이 요구하는 **정확한 이벤트 시퀀스**를 코드와 함께 보여준다. 특히 WebRTC는 오디오 버퍼를 서버가 관리해 인터럽션을 자동 truncate하는 반면, WebSocket은 클라이언트가 `conversation.item.truncate`를 수동 발사해야 한다는 차이를 강조한다.

후반부에서는 **응답 컨텍스트 조작** 패턴(`conversation: "none"`, 커스텀 `input` 배열, `item_reference`, 빈 `input`으로 컨텍스트 무시)과 function calling(세션/응답 레벨 tool 설정, `function_call` 감지, `function_call_output` 회신)이 설명된다. 이 모든 것은 한 세션 안에서 동시에 섞어 쓸 수 있다는 점이 API의 유연성을 드러낸다. Push-to-Talk는 VAD 끈 뒤 `input_audio_buffer.clear/append/commit` + `response.create` 시퀀스로 구현하는데, WebRTC에서는 `output_audio_buffer.clear`로 출력 측 정리까지 맞춰준다.

## Key Takeaways

- **Session-Conversation-Response 3계층**이 Realtime API의 핵심 멘탈 모델. 이 구분 없이 이벤트를 다루면 금방 꼬인다.
- **WebRTC는 "그냥 말해도 됨"**. 오디오 트랙만 추가하면 client event 없이도 대화가 시작된다. WebSocket은 모든 걸 JSON 이벤트로 수동 조립.
- Voice 변경은 **첫 오디오 방출 이후 불가**. 세션 시작 전 결정이 중요.
- **Out-of-band 응답**(`conversation: "none"` + `metadata`)으로 라우팅/분류기/모더레이션 같은 부가 작업을 본 대화 오염 없이 실행 가능.
- 커스텀 `input` 배열에 `item_reference`로 기존 아이템만 고르고, `input: []`로 완전히 컨텍스트 무시. "컨텍스트 엔지니어링"이 API 레벨 원시 요소.
- 인터럽션 시 WebSocket 측은 **클라이언트가 `conversation.item.truncate`(with `audio_end_ms`)를 직접 쏴줘야** 모델의 내부 기록과 재생된 음성이 일치한다. 전사 정렬은 정확히 맞지 않음을 감수.
- **Push-to-Talk 구현**은 WebSocket/WebRTC에서 각각 이벤트 셋이 약간 다르다. 공통은 `turn_detection: null` → commit → `response.create`.
- function calling은 텍스트 API와 동일하나, arguments가 delta 이벤트로 스트리밍되며 `call_id`로 결과를 묶어야 한다.

## Quotes & References

- *"To play output audio back on a client device like a web browser, we recommend using WebRTC rather than WebSockets."* — 브라우저 UX는 WebRTC가 사실상 의무.
- Voice 목록: alloy, ash, ballad, coral, echo, sage, shimmer, verse, marin, cedar. 권장: marin, cedar.
- 최대 세션: **60분**.
- audio append chunk 상한: **15 MB**.

## Related

- [[entities/openai-realtime-api|OpenAI Realtime API]]
- [[concepts/realtime-session-event-model|Realtime Session/Event Model]]
- [[concepts/voice-ai-agent|Voice AI Agent]]
- [[concepts/stt-llm-tts-pipeline|STT-LLM-TTS Pipeline]]
