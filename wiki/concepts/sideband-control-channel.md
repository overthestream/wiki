---
title: Sideband Control Channel
type: concept
tags: [ai, theory, voice-ai, security, architecture]
sources: ["raw/articles/Webhooks and server-side controls.md"]
created: 2026-04-21
updated: 2026-04-21
---

# Sideband Control Channel

## Definition

**Sideband control channel**은 클라이언트(브라우저·전화기)와 모델 서버가 직접 미디어를 주고받는 구조에서, **애플리케이션 백엔드가 같은 세션에 "곁길"로 한 연결을 더 열어** 도구 호출·프롬프트 변경·감사 같은 비즈니스 로직을 서버에서 독점하는 패턴이다. [[entities/openai-realtime-api|OpenAI Realtime API]]가 WebRTC와 SIP에 대해 공식 지원한다.

## Key Ideas

### 왜 필요한가

Realtime API는 브라우저/전화기에서 모델 서버로 직접 접속하는 것이 효율적이다 (WebRTC/SIP의 지연·미디어 품질 이점). 그러나 그 경로에 **도구 구현, 시스템 프롬프트, 정책, API 키 등 민감 로직**을 얹으면 클라이언트가 전부 볼 수 있게 된다. Sideband는 이 분리를 가능하게 한다 — 클라이언트는 오디오와 공개 제어만, 서버는 민감 제어·도구를 담당.

### WebRTC 패턴

1. 클라이언트가 SDP offer를 `POST /v1/realtime/calls`로 보내고 SDP answer를 받음 (Ephemeral key 사용).
2. 응답 헤더 `Location: /v1/realtime/calls/rtc_xxxxx`에서 **`call_id` 추출**.
3. 클라이언트는 이 `call_id`를 자기 백엔드로 넘김.
4. 백엔드가 `wss://api.openai.com/v1/realtime?call_id={callId}`에 WebSocket으로 붙음 (정식 API key 사용).
5. 이제 백엔드는 같은 세션에 대해 `session.update`, tool 정의, 이벤트 청취를 수행.

### SIP 패턴 (webhook 기반)

1. 사용자가 SIP 번호로 전화 → OpenAI가 통화를 수신.
2. OpenAI가 개발자 서버의 webhook 엔드포인트로 `realtime.call.incoming` 이벤트 POST. 페이로드에 `call_id`, `sip_headers`(From/To/Call-ID) 포함.
3. 서버가 `call_id`로 sideband WebSocket을 연결. 통화 수명 동안 유지.
4. 서버는 이 WebSocket으로 도구 호출을 처리하고 통화 제어 이벤트를 흘려보냄.

webhook에는 `webhook-id`(idempotency), `webhook-timestamp`, `webhook-signature`(서명 검증)가 함께 온다. **서명 검증을 반드시** 수행해 OpenAI에서 왔는지 확인.

### 쓰임새

- **도구 실행 서버화**: function tool의 arguments를 서버가 받고 실행, 결과를 session에 반영. 클라이언트는 API key·내부 서비스 자격증명을 보지 않는다.
- **정책/프롬프트 동적 변경**: 세션 중 사용자 프로필·컴플라이언스 상태에 따라 `session.update`로 instructions 교체.
- **감사/녹취**: 서버가 `response.done`, 전사 delta 이벤트를 저장.
- **콜센터 통화 제어**: SIP 통화에서 휴먼 에스컬레이션, 통화 종료, 전사 저장.

### 보안 설계 체크리스트

- 클라이언트는 **Ephemeral key**만 가지고, 정식 API key는 절대 임베드하지 않는다.
- webhook 수신 시 서명·타임스탬프 검증, 재생 공격 방지.
- `call_id`는 URL 쿼리로 노출되므로 짧은 수명과 접근 경로에 주의.
- Tool 실행은 서버가 도맡고, 클라이언트는 **승인/UI 측면**만 다룬다.

## Examples

- **브라우저 보이스 앱**: 사용자는 WebRTC로 음성 통화. 브라우저는 API key 없이 ephemeral 키만 보유. 서버가 sideband WebSocket으로 결제·주문 tool 호출을 처리.
- **고객지원 전화 봇**: 사용자가 800번으로 전화 → OpenAI webhook → 서버가 call_id로 접속해 에스컬레이션 tool과 KB 검색 tool 연결.
- **세션 중 정책 강화**: 사용자 신원 확인 후, 서버가 `session.update`로 "Be extra strict about PII" 같은 instructions를 주입.

## Related

- [[entities/openai-realtime-api|OpenAI Realtime API]]
- [[concepts/realtime-session-event-model|Realtime Session/Event Model]]
- [[concepts/voice-ai-agent|Voice AI Agent]]
- [[summaries/2026-04-21-openai-realtime-server-controls|Webhooks & server-side controls 요약]]
