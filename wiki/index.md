---
title: Wiki Index
type: index
updated: 2026-04-21
---

# Wiki Index

이 위키의 모든 페이지를 범주별로 정리한 마스터 인덱스.

## Entities (개체)
<!-- 인물, 조직, 도구, 장소 -->
- [[entities/amazon-bedrock-agentcore|Amazon Bedrock AgentCore]] — AWS의 서버리스 AI 에이전트 호스팅 런타임
- [[entities/amazon-nova-sonic|Amazon Nova Sonic]] — AWS의 speech-to-speech foundation model, tool use 기반 외부 연동
- [[entities/livekit|LiveKit]] — WebRTC 기반 리얼타임 인프라와 Voice AI Agents 프레임워크를 제공하는 오픈소스 플랫폼
- [[entities/openai-realtime-api|OpenAI Realtime API]] — gpt-realtime 기반 speech-to-speech API, WebRTC/WebSocket/SIP 접속과 MCP 도구 지원
- [[entities/strands-agents|Strands Agents]] — Python LLM 에이전트 프레임워크, Bedrock 모델 연동

## Concepts (개념)
<!-- 개념, 주제, 기술 -->
- [[concepts/jvm-monitor|JVM Monitor]] — Java synchronized의 내부 동작: Object Header, 락 에스컬레이션, CAS/MESI
- [[concepts/mcp-model-context-protocol|MCP (Model Context Protocol)]] — 원격 서버 도구를 모델에 표준 방식으로 노출하는 프로토콜, Realtime API 1급 통합
- [[concepts/monitor|Monitor]] — 동기화 추상화: 상호 배제 + 조건 변수, Hoare vs Mesa 의미론
- [[concepts/multi-agent-voice-architecture|멀티 에이전트 보이스 아키텍처]] — 보이스 에이전트를 오케스트레이터 + 도메인 서브 에이전트로 분할하는 설계 패턴
- [[concepts/realtime-session-event-model|Realtime Session/Event Model]] — OpenAI Realtime API의 Session-Conversation-Response 3계층 상태와 이벤트 플로우
- [[concepts/semaphore|Semaphore]] — 저수준 동기화 프리미티브: P/V 연산, 모니터의 전신
- [[concepts/sideband-control-channel|Sideband Control Channel]] — 클라이언트 미디어 경로와 별개로 서버가 같은 세션에 여는 제어 채널
- [[concepts/stt-llm-tts-pipeline|STT-LLM-TTS Pipeline]] — 음성 AI의 cascade 파이프라인 구조: 스트리밍 처리, 턴 감지, 인터럽션
- [[concepts/thread-pinning|Thread Pinning]] — Virtual Thread가 carrier에 고정되는 문제와 JDK 24 해결
- [[concepts/virtual-thread|Virtual Thread]] — Java 21 경량 스레드: 스케줄링, 도입 가이드라인, 실무 적용
- [[concepts/voice-agent-prompting|Voice Agent Prompting]] — speech-to-speech 모델(gpt-realtime) 프롬프팅 실전 가이드 10가지
- [[concepts/voice-ai-agent|Voice AI Agent]] — 음성을 주 채널로 삼는 리얼타임 AI 에이전트: WebRTC, 턴 감지, 인터럽션

## Summaries (요약)
<!-- 원본 자료별 요약 -->
- [[summaries/2026-04-15-java-virtual-threads|Java Virtual Threads 공식 문서 요약]] — Oracle Java 21 공식 문서 기반 Virtual Thread 개요
- [[summaries/2026-04-15-monitors|모니터 개념 정리]] — 모니터의 정의, Hoare vs Mesa, Java synchronized와의 관계
- [[summaries/2026-04-15-simple-async-task-executor|SimpleAsyncTaskExecutor API 요약]] — Spring의 TaskExecutor: Virtual Thread 지원, 동시성 제한, graceful shutdown
- [[summaries/2026-04-15-virtual-thread-qna|Virtual Thread 심층 Q&A]] — JVM 모니터, JDK 24 개선, JDBC/MongoDB 호환성까지 확장한 Q&A
- [[summaries/2026-04-21-livekit-documentation|LiveKit Agents 프레임워크 공식 문서 요약]] — LiveKit Agents의 구조, 수명주기, 주요 사용 사례
- [[summaries/2026-04-21-nova-sonic-multi-agent-voice-assistant|Nova Sonic + AgentCore 멀티 에이전트 요약]] — speech-to-speech 모델과 AgentCore 서브 에이전트 조합 패턴
- [[summaries/2026-04-21-openai-realtime-conversations|OpenAI Realtime conversations 요약]] — Realtime API 세션/이벤트 모델, 오디오 IO, Out-of-Band, Push-to-Talk
- [[summaries/2026-04-21-openai-realtime-mcp|Realtime API with MCP 요약]] — Realtime 세션에 MCP 도구 붙이는 설정·이벤트·실패 모드
- [[summaries/2026-04-21-openai-realtime-server-controls|OpenAI Realtime: Webhooks & server-side controls 요약]] — WebRTC/SIP sideband 연결로 비즈니스 로직 서버 분리
- [[summaries/2026-04-21-openai-realtime-using-models|OpenAI Realtime: Using realtime models 요약]] — gpt-realtime 프롬프팅 10가지 실전 팁

## Syntheses (종합)
<!-- 교차 분석, 비교 -->

## Projects (프로젝트)
<!-- 프로젝트, 업무 -->
