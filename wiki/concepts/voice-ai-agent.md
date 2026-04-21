---
title: Voice AI Agent
type: concept
tags: [ai, theory, realtime, voice-ai]
sources: ["raw/articles/LiveKit Documentation.md", "raw/articles/Building a multi-agent voice assistant with Amazon Nova Sonic and Amazon Bedrock AgentCore.md"]
created: 2026-04-21
updated: 2026-04-21
---

# Voice AI Agent

## Definition

Voice AI Agent는 **음성을 주요 입출력 채널로 삼아 사용자와 실시간으로 상호작용하는 AI 프로그램**이다. 텍스트 기반 챗봇과 달리, 지연시간·턴 감지·인터럽션·네트워크 불안정성 같은 "대화의 리얼타임성"이 설계의 1차 제약이다. 음성 외에 텍스트·비전·화면공유 등을 보조 채널로 포함하면 멀티모달 에이전트로 확장된다.

## Key Ideas

### 상태ful 리얼타임 브리지

에이전트의 본질은 **"AI 모델과 사용자 사이의 상태ful 리얼타임 브리지"**다. 모델은 보통 안정된 데이터센터에 있지만, 사용자는 모바일 네트워크처럼 품질이 가변적인 환경에서 접속한다. 따라서 에이전트 계층이 네트워크 불안정을 흡수하고, 모델의 동기식 요청/응답을 스트리밍 대화로 변환해야 한다.

### 왜 HTTP가 아니라 WebRTC인가

음성 대화는 수백 ms 단위의 지연에도 품질 저하가 체감된다. HTTP 기반 polling이나 long-polling으로는:

- 인터럽션(사용자가 에이전트 말을 끊고 개입) 감지가 어렵다.
- 양방향 스트리밍 오디오의 jitter 흡수가 미흡하다.
- 네트워크 악화 시 복구 메커니즘이 빈약하다.

WebRTC는 이 세 가지를 프로토콜 레벨에서 처리한다. [[entities/livekit|LiveKit]] Agents가 프론트엔드 ↔ 에이전트 구간을 WebRTC로 고정하고, 비즈니스 로직용 HTTP/WebSocket은 에이전트 ↔ 백엔드 구간으로만 분리하는 이유다.

### 핵심 서브-문제들

보이스 AI 에이전트 프레임워크가 공통으로 풀어야 하는 과제:

1. **STT-LLM-TTS 파이프라인 오케스트레이션** → [[concepts/stt-llm-tts-pipeline|STT-LLM-TTS Pipeline]]
2. **턴 감지 (Turn Detection)**: 사용자가 말을 끝냈는지 언제 판단하는가. 단순 VAD(음성활동감지)로는 부족하고, 문맥·의미를 고려한 전용 모델이 사용된다.
3. **인터럽션 처리**: 에이전트가 TTS를 재생 중일 때 사용자가 끼어들면 즉시 재생 중단하고 새 입력 처리.
4. **도구 사용(Tool Use)**: LLM이 외부 함수/API를 호출하도록 연결. 프론트엔드로 tool call을 포워딩하는 패턴도 존재.
5. **멀티 에이전트 핸드오프**: 복잡한 워크플로우를 역할별 에이전트로 분해하고 제어권 이양. → [[concepts/multi-agent-voice-architecture|멀티 에이전트 보이스 아키텍처]]

### 배포 형태

LiveKit Agents의 모델을 일반화하면: 에이전트 프로세스는 **"서버로 등록 → 디스패치 대기 → job 서브프로세스 스폰 → 세션 join"** 단계를 거친다. Room/Session 생성마다 프로세스가 격리되어 한 사용자의 크래시가 다른 세션에 파급되지 않고, Kubernetes 수평 확장이 자연스럽다.

## Examples

- **멀티모달 어시스턴트**: 말하기·텍스트·화면공유를 혼용하는 개인 비서.
- **원격의료 삼자 통화**: AI가 환자와 1차 상담 후 의사에게 인수인계(human-in-the-loop).
- **콜센터 인바운드/아웃바운드**: SIP 텔레포니 통합으로 전화망과 직접 연결.
- **실시간 번역**: 한쪽 음성을 다른 언어로 합성해 상대에게 전달.
- **LLM 기반 게임 NPC**: 정적 스크립트 대신 동적 대화.
- **로보틱스**: 로봇 센서 입력을 클라우드의 강력한 모델로 보내고 행동을 내려받음.

### 구현 접근법 두 가지

같은 Voice AI Agent라도 서로 다른 프로토콜/제품 스택으로 구현된다:

- **WebRTC Room 모델** ([[entities/livekit|LiveKit]]): 사용자와 에이전트가 같은 Room의 참여자로 join해 미디어를 publish/subscribe. 인터럽션·턴 감지가 프로토콜 레벨에서 자연스럽다.
- **Speech-to-Speech 모델 + tool use** ([[entities/amazon-nova-sonic|Amazon Nova Sonic]] + [[entities/amazon-bedrock-agentcore|AgentCore]]): 음성-음성 foundation model이 IO와 라우팅을 동시 담당하고, tool use 이벤트로 서브 에이전트에 위임.

두 접근 모두 [[concepts/multi-agent-voice-architecture|멀티 에이전트 아키텍처]]로 확장 가능하지만, 핸드오프 모델(Room 내 역할 전환 vs tool use round-trip)이 다르다.

## Related

- [[entities/livekit|LiveKit]]
- [[entities/amazon-nova-sonic|Amazon Nova Sonic]]
- [[entities/amazon-bedrock-agentcore|Amazon Bedrock AgentCore]]
- [[concepts/stt-llm-tts-pipeline|STT-LLM-TTS Pipeline]]
- [[concepts/multi-agent-voice-architecture|멀티 에이전트 보이스 아키텍처]]
- [[summaries/2026-04-21-livekit-documentation|LiveKit Agents 프레임워크 공식 문서 요약]]
- [[summaries/2026-04-21-nova-sonic-multi-agent-voice-assistant|Nova Sonic + AgentCore 멀티 에이전트 요약]]
