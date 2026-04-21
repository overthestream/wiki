---
title: OpenAI Realtime API
type: entity
tags: [tool, ai, openai, voice-ai, api, realtime]
sources: ["raw/articles/Using realtime models  OpenAI API.md", "raw/articles/Realtime conversations  OpenAI API.md", "raw/articles/Realtime API with MCP.md", "raw/articles/Webhooks and server-side controls.md"]
created: 2026-04-21
updated: 2026-04-21
---

# OpenAI Realtime API

## Overview

OpenAI Realtime API는 **speech-to-speech foundation model을 브라우저/서버에서 직접 호출**할 수 있게 해주는 OpenAI의 리얼타임 대화 API다. 중간의 STT/TTS 단계 없이 오디오 입력을 받아 오디오 출력을 직접 생성하며, 텍스트·이미지·함수 호출·[[concepts/mcp-model-context-protocol|MCP]] 도구까지 하나의 세션 안에서 엮을 수 있다. 프론트엔드 접속은 WebRTC·WebSocket·SIP 세 방식을 지원한다.

## Key Facts

- **대표 모델**: `gpt-realtime` (주력), `gpt-realtime-mini`, `gpt-realtime-1.5` (MCP 예시). 이미지 입력 지원.
- **접속 방식**: WebRTC(브라우저/클라이언트 권장), WebSocket(서버-서버·풀 제어), SIP(전화망).
- **세션 최대 길이**: 60분.
- **음성 옵션**: `alloy`, `ash`, `ballad`, `coral`, `echo`, `sage`, `shimmer`, `verse`, `marin`, `cedar`. 첫 오디오 방출 이후에는 session 내 voice 변경 불가. 품질은 `marin`/`cedar` 권장.
- **오디오 포맷**: `audio/pcm` 24kHz 입력이 기본 예시.
- **턴 감지**: 서버 VAD 기본 켜짐(`semantic_vad`, `server_vad`). `turn_detection: null`로 Push-to-Talk 전환 가능.
- **출처**:
  - 모델·프롬프팅 가이드: https://developers.openai.com/api/docs/guides/realtime-models-prompting
  - 대화/세션 가이드: https://developers.openai.com/api/docs/guides/realtime-conversations
  - MCP 통합: https://developers.openai.com/api/docs/guides/realtime-mcp
  - 서버사이드 제어: https://developers.openai.com/api/docs/guides/realtime-server-controls
  - 프롬프팅 쿡북: https://developers.openai.com/cookbook/examples/realtime_prompting_guide

## Notes

### 세션 모델

Realtime 세션은 상태 있는 구조다:

- **Session**: 모델, voice, 출력 모달리티, prompt, tool 등 세션 전역 설정.
- **Conversation**: 사용자 입력 Item + 모델 응답 Item 시퀀스 (기본 conversation).
- **Response**: 모델이 생성하는 개별 오디오/텍스트 산출물. `conversation: "none"`으로 out-of-band 응답도 가능.

상세는 [[concepts/realtime-session-event-model|Realtime Session/Event Model]] 참조.

### 음성 IO 처리의 전송 방식 차이

- **WebRTC**: 브라우저가 peer connection으로 미디어를 주고받음. 출력 버퍼를 서버가 관리하므로 **인터럽션 시 자동 truncate**. 브라우저 재생에는 WebRTC를 권장.
- **WebSocket**: 클라이언트가 직접 base64 PCM 청크를 `input_audio_buffer.append`로 송신, 응답 오디오도 `response.output_audio.delta`로 수동 조립. 인터럽션 시 클라이언트가 `conversation.item.truncate`로 잘라내야 한다.
- **SIP**: 전화망. 수신 호 도착 시 OpenAI가 사용자 서버에 webhook을 보내고, 서버는 `call_id`로 sideband WebSocket을 연다 → [[concepts/sideband-control-channel|Sideband Control Channel]] 참조.

### Prompt 객체와 override 규칙

`session.prompt.id`에 서버 저장 prompt를 참조할 수 있고 `version` 핀, `variables` 주입이 가능하다. 동시에 `session.instructions` 같은 직접 필드를 설정하면 **prompt 필드보다 우선**한다. 중간에 `session.update`로 prompt 버전·변수를 갈아끼우는 mid-call 스왑도 지원.

### 도구 통합의 두 계열

- **function tool**: 종전 function calling과 동일. 모델이 arguments를 내고 **클라이언트가 실행** → `function_call_output`으로 결과 회신.
- **MCP tool** (`type: "mcp"`): **Realtime API 서버가 직접 원격 MCP 서버를 호출**. 클라이언트는 approval만 처리. built-in connector(예: `connector_googlecalendar`)도 같은 shape. 자세히는 [[concepts/mcp-model-context-protocol|MCP]].

### Nova Sonic과의 포지셔닝 비교

[[entities/amazon-nova-sonic|Amazon Nova Sonic]]도 speech-to-speech + tool use 계열이지만, Nova Sonic은 **AWS 에코시스템(Bedrock AgentCore, Strands)** 안에서 오케스트레이터 역할을 맡는 쪽에 무게를 둔다. OpenAI Realtime API는 **원하는 클라이언트(브라우저·서버·전화)에서 직접 모델에 꽂는** 범용 API에 가깝다. 두 제품 모두 [[concepts/multi-agent-voice-architecture|멀티 에이전트 보이스 아키텍처]]의 구성 요소가 될 수 있다.

## Related

- [[concepts/realtime-session-event-model|Realtime Session/Event Model]]
- [[concepts/mcp-model-context-protocol|MCP (Model Context Protocol)]]
- [[concepts/sideband-control-channel|Sideband Control Channel]]
- [[concepts/voice-agent-prompting|Voice Agent Prompting]]
- [[concepts/voice-ai-agent|Voice AI Agent]]
- [[concepts/stt-llm-tts-pipeline|STT-LLM-TTS Pipeline]]
- [[entities/amazon-nova-sonic|Amazon Nova Sonic]]
- [[summaries/2026-04-21-openai-realtime-using-models|Using realtime models 요약]]
- [[summaries/2026-04-21-openai-realtime-conversations|Realtime conversations 요약]]
- [[summaries/2026-04-21-openai-realtime-mcp|Realtime API with MCP 요약]]
- [[summaries/2026-04-21-openai-realtime-server-controls|Webhooks & server-side controls 요약]]
