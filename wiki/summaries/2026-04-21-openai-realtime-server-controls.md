---
title: Webhooks and server-side controls (OpenAI Realtime API) 요약
type: summary
tags: [ai, voice-ai, openai, security, webhook, sip, webrtc]
sources: ["raw/articles/Webhooks and server-side controls.md", "https://developers.openai.com/api/docs/guides/realtime-server-controls"]
created: 2026-04-21
updated: 2026-04-21
---

# Webhooks and server-side controls (OpenAI Realtime API) 요약

## Source Info
- **출처**: OpenAI Developers Docs (`/api/docs/guides/realtime-server-controls`)
- **저자**: OpenAI (공식 가이드)
- **날짜**: 2026-04-21 clipping

## Summary

이 문서는 [[entities/openai-realtime-api|OpenAI Realtime API]]에서 **클라이언트는 WebRTC/SIP로 모델에 직접 붙더라도, 애플리케이션 백엔드가 같은 세션에 sideband 연결을 하나 더 열어 비즈니스 로직을 서버에 둘 수 있음**을 설명한다. 도구 정의, 시스템 프롬프트, 감시/감사 같은 민감 로직을 클라이언트에 노출하지 않는 것이 목적이다.

WebRTC 패턴은 SDP negotiation 응답의 `Location` 헤더에서 `call_id`를 뽑아 이를 서버로 전달하고, 서버가 `wss://api.openai.com/v1/realtime?call_id=...`로 WebSocket을 열어 sideband 제어 채널을 확보하는 구조다. 클라이언트는 ephemeral key로만 접속하고, 정식 API key는 서버에만 둔다.

SIP 패턴은 webhook 기반이다. 사용자가 SIP 번호로 전화를 걸면 OpenAI가 개발자 서버로 `realtime.call.incoming` 이벤트를 POST 한다. 페이로드에는 `call_id`와 SIP 헤더(From/To/Call-ID)가 들어 있고, `webhook-id`/`webhook-timestamp`/`webhook-signature`가 같이 와서 idempotency·재생방지·서명검증을 수행할 수 있다. 서버는 이 `call_id`로 sideband WebSocket을 열어 통화 수명 동안 유지한다.

## Key Takeaways

- **"sideband"의 정의**: 같은 Realtime 세션에 클라이언트 연결 하나 + 서버 연결 하나가 동시에 붙는 구조. 서버는 이 채널로 `session.update`·tool 정의·이벤트 청취를 수행.
- **WebRTC의 `call_id`** 획득: SDP POST 응답의 `Location` 헤더가 `/v1/realtime/calls/rtc_...` 포맷. 이걸 서버로 넘기기만 하면 된다.
- **SIP의 `call_id`** 획득: OpenAI가 보내는 `realtime.call.incoming` webhook 페이로드. 통화 한 건당 한 번.
- Webhook에는 **`webhook-signature` 필수 검증**. OpenAI가 보낸 것인지, 재생 공격이 아닌지 확인.
- 클라이언트는 **ephemeral key**만. 정식 API key는 반드시 서버 쪽에서만 쓴다.
- 도구 실행 책임을 서버로 올리면, 민감 자격증명(DB, 내부 API)을 클라이언트가 몰라도 된다. 프롬프트/정책도 서버 제어로 런타임 교체 가능.
- `call_id`는 URL 쿼리로 붙으므로 로깅·프록시 경로에서 유출되지 않도록 주의. 수명은 통화 1건.

## Quotes & References

- *"A sideband connection means there are two active connections to the same Realtime session: one from the user's client and one from your application server."*
- Webhook 예시 헤더: `webhook-id`, `webhook-timestamp`, `webhook-signature`(v1,... 서명).
- Webhook 이벤트 타입: `realtime.call.incoming`.
- 서버 WS URL: `wss://api.openai.com/v1/realtime?call_id=rtc_xxxxx`.

## Related

- [[entities/openai-realtime-api|OpenAI Realtime API]]
- [[concepts/sideband-control-channel|Sideband Control Channel]]
- [[concepts/realtime-session-event-model|Realtime Session/Event Model]]
- [[concepts/voice-ai-agent|Voice AI Agent]]
