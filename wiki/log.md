---
title: Activity Log
type: log
updated: 2026-04-21
---

# Activity Log

위키 활동의 시간순 기록. 각 항목은 `## [날짜] action | 제목` 형식.

---

## [2026-04-15] init | Wiki 초기 설정
- LLM Wiki 패턴 기반 Obsidian vault 구성
- 디렉토리 구조, 스키마(CLAUDE.md), 인덱스, 로그, 템플릿 생성

## [2026-04-15] ingest | Java Virtual Threads 공식 문서
- 원본: raw/articles/java-virtual-threads.md
- 요약: Oracle Java 21 공식 문서 기반 Virtual Thread 개요 — 생성, 스케줄링, pinning, 디버깅, 도입 가이드라인
- 새 페이지: [[summaries/2026-04-15-java-virtual-threads]], [[concepts/virtual-thread]], [[concepts/thread-pinning]], [[concepts/jvm-monitor]]

## [2026-04-15] ingest | Virtual Thread 심층 Q&A
- 원본: raw/articles/virtual-thread-QnA.md
- 요약: JVM 모니터 하드웨어 레벨 분석, JDK 24 pinning 해결, Semaphore 패턴, JDBC/MongoDB 드라이버 호환성
- 새 페이지: [[summaries/2026-04-15-virtual-thread-qna]]
- 업데이트: [[concepts/virtual-thread]], [[concepts/thread-pinning]], [[concepts/jvm-monitor]]

## [2026-04-15] ingest | SimpleAsyncTaskExecutor API
- 원본: raw/articles/SimpleAsyncTaskExecutor (Spring Framework 7.0.6 API).md
- 요약: Spring의 TaskExecutor 구현체 — Virtual Thread 모드, 동시성 제한, graceful shutdown
- 새 페이지: [[summaries/2026-04-15-simple-async-task-executor]]
- 업데이트: [[concepts/virtual-thread]]

## [2026-04-15] ingest | 모니터 개념 정리
- 원본: raw/articles/Monitors.md
- 요약: 동시성 프로그래밍의 모니터 개념 — 세마포어 한계, 모니터 구조, Hoare vs Mesa, Java synchronized와의 관계
- 새 페이지: [[summaries/2026-04-15-monitors]], [[concepts/monitor]], [[concepts/semaphore]]
- 업데이트: [[concepts/jvm-monitor]], [[concepts/virtual-thread]]

## [2026-04-21] ingest | LiveKit Agents 프레임워크 공식 문서
- 원본: raw/articles/LiveKit Documentation.md
- 요약: LiveKit Agents는 LiveKit Room의 리얼타임 참여자로 동작하는 Voice AI 에이전트 프레임워크. WebRTC/STT-LLM-TTS/턴 감지/멀티모달을 SDK 레벨로 추상화
- 새 페이지: [[summaries/2026-04-21-livekit-documentation]], [[entities/livekit]], [[concepts/voice-ai-agent]], [[concepts/stt-llm-tts-pipeline]]
- 인사이트: 위키 주제가 Java/동시성 중심에서 AI/Realtime 도메인으로 처음 확장. 장기적으로 에이전트 서버의 동시성 모델이 [[concepts/virtual-thread]]와 연결될 여지 있음

## [2026-04-21] ingest | Amazon Nova Sonic + Bedrock AgentCore 멀티 에이전트 보이스 어시스턴트
- 원본: raw/articles/Building a multi-agent voice assistant with Amazon Nova Sonic and Amazon Bedrock AgentCore.md
- 요약: speech-to-speech foundation model(Nova Sonic)이 오케스트레이터로서 tool use 이벤트를 통해 AgentCore 위의 Strands 서브 에이전트를 호출하는 멀티 에이전트 보이스 아키텍처. 뱅킹 어시스턴트 샘플로 설명
- 새 페이지: [[summaries/2026-04-21-nova-sonic-multi-agent-voice-assistant]], [[entities/amazon-nova-sonic]], [[entities/amazon-bedrock-agentcore]], [[entities/strands-agents]], [[concepts/multi-agent-voice-architecture]]
- 업데이트: [[concepts/voice-ai-agent]] — 구현 접근법(WebRTC Room vs speech-to-speech + tool use) 비교 섹션 추가
- 인사이트: Voice AI 도메인의 두 계열(LiveKit 방식 / Nova Sonic 방식)이 동시에 위키에 들어오며 [[concepts/multi-agent-voice-architecture]]가 이 둘을 묶는 상위 개념으로 생성됨. 핸드오프 모델 차이(Room 내 역할 전환 vs tool use round-trip)가 향후 비교/종합 페이지 후보

## [2026-04-21] ingest | OpenAI Realtime API 공식 문서 4건 (using models, conversations, MCP, server-side controls)
- 원본:
  - raw/articles/Using realtime models  OpenAI API.md
  - raw/articles/Realtime conversations  OpenAI API.md
  - raw/articles/Realtime API with MCP.md
  - raw/articles/Webhooks and server-side controls.md
- 요약: OpenAI Realtime API(gpt-realtime)의 프롬프팅 가이드, Session-Conversation-Response 이벤트 모델, MCP 도구 통합, WebRTC/SIP sideband 서버 제어를 다루는 공식 문서 4건 일괄 수집
- 새 페이지:
  - [[summaries/2026-04-21-openai-realtime-using-models]]
  - [[summaries/2026-04-21-openai-realtime-conversations]]
  - [[summaries/2026-04-21-openai-realtime-mcp]]
  - [[summaries/2026-04-21-openai-realtime-server-controls]]
  - [[entities/openai-realtime-api]]
  - [[concepts/realtime-session-event-model]]
  - [[concepts/mcp-model-context-protocol]]
  - [[concepts/sideband-control-channel]]
  - [[concepts/voice-agent-prompting]]
- 업데이트: [[concepts/voice-ai-agent]] (세 번째 구현 접근법 추가), [[concepts/stt-llm-tts-pipeline]] (s2s 모델과의 대비 확대), [[concepts/multi-agent-voice-architecture]] (MCP 변종 패턴 섹션 추가)
- 인사이트:
  - Voice AI 도메인이 이제 세 계열로 정리됨: LiveKit(WebRTC Room) / Nova Sonic(AWS s2s + AgentCore) / OpenAI Realtime(직결 API + MCP). 핸드오프·도구 통합 방식이 각기 다르며, 비교/종합 페이지 후보로 점점 익어가는 중
  - MCP는 function tool과 실행 주체(클라 vs 플랫폼)가 다른 것이 핵심 차이. 도구 권한·보안 모델에 직접 영향
  - Sideband 채널은 voice 앱에서 "클라이언트는 미디어·ephemeral key, 서버는 민감 제어·정식 key"의 보안 분리 기준을 제공
