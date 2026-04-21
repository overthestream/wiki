---
title: 멀티 에이전트 보이스 아키텍처
type: concept
tags: [ai, theory, voice-ai, architecture, multi-agent]
sources: ["raw/articles/Building a multi-agent voice assistant with Amazon Nova Sonic and Amazon Bedrock AgentCore.md", "raw/articles/Realtime API with MCP.md"]
created: 2026-04-21
updated: 2026-04-21
---

# 멀티 에이전트 보이스 아키텍처

## Definition

**멀티 에이전트 보이스 아키텍처**는 단일 음성 에이전트가 모든 로직을 담당하는 대신, **도메인별로 특화된 서브 에이전트를 조합**하고 이들을 오케스트레이터가 라우팅하는 설계 패턴이다. 텍스트 챗봇의 멀티 에이전트 패턴을 보이스 도메인으로 옮긴 것이지만, 음성 UX 고유의 제약(지연, 핸드오프 비가시성, 인터럽션)이 설계에 추가된다.

## Key Ideas

### 모놀리식 보이스 에이전트의 한계

기능이 늘수록 system prompt가 비대해지고 reasoning 로직이 복잡해진다. 소프트웨어 공학의 모놀리식 설계가 겪는 유지보수·확장성 문제를 동일하게 겪는다 — 인증·계좌조회·모기지처럼 서로 독립적인 업무를 한 프롬프트에 밀어넣으면 회귀가 쉽고 테스트가 어렵다.

### 역할 분리: 오케스트레이터 + 도메인 전문가

- **오케스트레이터(보이스 IO + 라우터)**: 사용자의 발화를 듣고, 어떤 전문가에게 넘길지 분류한다. [[entities/amazon-nova-sonic|Nova Sonic]]이 이 역할을 맡는 AWS 사례에서는 내장 reasoning 모델이 tool use 이벤트로 라우팅을 수행한다.
- **도메인 서브 에이전트**: 각자 전문 도메인의 로직·도구·시스템 프롬프트를 캡슐화한다. 입력 검증, 외부 API 호출, 결과 요약까지 자기 책임으로 수행.
- **연결 프로토콜**: LLM [[concepts/voice-ai-agent|Voice Agent]]의 tool use가 사실상의 IPC다. 오케스트레이터가 tool 이벤트를 발사하고, 클라이언트가 서브 에이전트에 전달, 응답을 다시 오케스트레이터에게 돌려준다.

### 사용자에게는 단일 대화처럼 보여야 한다

음성 UX의 핵심 제약은 "핸드오프가 드러나지 않아야 한다"는 것이다. 지연, 목소리 변경, 맥락 끊김이 없어야 한다. 이 제약이 아키텍처 선택에 직접 영향을 준다:

- **지연 예산**: 핸드오프마다 tool use 라운드트립이 쌓이므로 서브 에이전트는 Nova Lite처럼 작은 모델 사용이 기본값.
- **음성 친화 응답 강제**: 서브 에이전트 system prompt에 "raw JSON 금지, 2~3문장 자연어"를 명시. 그렇지 않으면 TTS가 어색해진다.
- **입력 검증 로컬화**: 계좌 ID 형식 검증처럼 도메인 제약은 서브 에이전트가 자체 처리해 오케스트레이터 reasoning을 단순하게 유지.

### Stateless vs Stateful 서브 에이전트

| 특성 | Stateless | Stateful |
|------|-----------|----------|
| 메모리 | 매 요청 독립 | 세션/사용자 컨텍스트 유지 |
| 구현 난이도 | 낮음 | 높음 (외부 상태 저장소 필요) |
| 스케일링 | 수평 확장 쉬움 | 상태 공유 비용 |
| 적합 사례 | 단발 조회, 계산 | 멀티턴 상담, 개인화 |

기본은 stateless. 정말 필요한 경우에만 stateful로 승격한다.

### 마이크로서비스와의 유비

저자는 이 패턴을 **보이스 에이전트 버전의 마이크로서비스**로 설명한다. 모듈성·확장성·재사용성 이득은 비슷하지만, 멀티 에이전트 특유의 추가 비용이 있다:

- **프롬프트 버전 드리프트**: 오케스트레이터의 tool 스펙과 서브 에이전트 system prompt가 함께 진화해야 한다.
- **라우팅 품질이 LLM 분류기에 의존**: 새 tool description 하나가 전체 routing 정확도를 흔들 수 있다.
- **관측성 난이도**: 한 대화가 여러 에이전트에 걸쳐 있어 trace/로그 상관관계가 필수.

### MCP로 "서브 에이전트 대신 서버측 도구 집합"을 묶는 변종

[[entities/openai-realtime-api|OpenAI Realtime API]]는 서브 에이전트를 별도 프로세스로 띄우는 대신, 도메인 기능을 **[[concepts/mcp-model-context-protocol|MCP]] 서버**로 패키징해 세션에 붙이는 길을 제공한다. 이 경우 "서브 에이전트"의 지위는 약해지지만, 도구 실행을 OpenAI 측 서버가 대행하므로 클라이언트 복잡도가 낮아진다. 경계가 에이전트 프로세스 단위(AgentCore)냐, 도구 프로토콜 단위(MCP)냐의 차이다.

## Examples

- **AWS 뱅킹 보이스 어시스턴트**: Nova Sonic 오케스트레이터 + AgentCore에 올린 Authenticate/Banking/Mortgage 서브 에이전트 조합 ([[summaries/2026-04-21-nova-sonic-multi-agent-voice-assistant|요약]] 참조).
- **[[entities/livekit|LiveKit]] Agents의 멀티 에이전트 핸드오프**: 동일한 아이디어를 WebRTC Room 기반으로 구현. 세션 내에서 에이전트 역할을 전환하는 방식.
- **OpenAI Realtime + MCP 커넥터**: `gpt-realtime` 세션에 Google Calendar/Docs/사내 MCP 서버를 동시에 붙여, 각 도메인 tool을 하나의 모델이 오케스트레이션 ([[summaries/2026-04-21-openai-realtime-mcp|요약]] 참조).

## Related

- [[concepts/voice-ai-agent|Voice AI Agent]]
- [[concepts/stt-llm-tts-pipeline|STT-LLM-TTS Pipeline]]
- [[entities/amazon-nova-sonic|Amazon Nova Sonic]]
- [[entities/amazon-bedrock-agentcore|Amazon Bedrock AgentCore]]
- [[entities/strands-agents|Strands Agents]]
- [[entities/livekit|LiveKit]]
- [[entities/openai-realtime-api|OpenAI Realtime API]]
- [[concepts/mcp-model-context-protocol|MCP (Model Context Protocol)]]
- [[summaries/2026-04-21-nova-sonic-multi-agent-voice-assistant|Nova Sonic + AgentCore 멀티 에이전트 요약]]
- [[summaries/2026-04-21-openai-realtime-mcp|Realtime API with MCP 요약]]
