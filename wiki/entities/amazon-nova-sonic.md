---
title: Amazon Nova Sonic
type: entity
tags: [tool, ai, aws, voice-ai, foundation-model]
sources: ["raw/articles/Building a multi-agent voice assistant with Amazon Nova Sonic and Amazon Bedrock AgentCore.md"]
created: 2026-04-21
updated: 2026-04-21
---

# Amazon Nova Sonic

## Overview

Amazon Nova Sonic은 AWS가 제공하는 **speech-to-speech foundation model**이다. STT→LLM→TTS로 이어지는 전통적 cascade 파이프라인 대신, 음성 입력을 받아 음성 출력을 직접 생성하는 end-to-end 구조로 설계되어 있다. 톤 인식, 자연스러운 대화 흐름, 도구 사용(tool use)을 통한 외부 시스템 연동이 기본 기능으로 포함된다. 생성형 AI 애플리케이션의 리얼타임 음성 인터페이스를 목표로 한다.

## Key Facts

- **제공처**: AWS (Amazon Bedrock 계열, Nova 패밀리).
- **분류**: Speech-to-speech foundation model.
- **핵심 능력**: 실시간 양방향 음성 대화, 톤/흐름 이해, tool use 기반 액션 수행.
- **tool use 설정 지점**: `promptStart` 이벤트에서 tool 스펙 주입. 내장 reasoning 모델이 발화를 분류해 해당 `toolUse` 이벤트를 클라이언트로 방출.
- **토큰화 특성**: 음성 STT 결과가 tool 입력에 그대로 흐른다. 예: `accountId`가 `"one two three four five"` 형태로 전달되어 클라이언트/서브 에이전트 측 정규화가 필요.
- **공식 문서**: https://docs.aws.amazon.com/nova/latest/userguide/speech.html
- **워크샵**: https://catalog.workshops.aws/amazon-nova-sonic-s2s
- **기술 리포트**: https://www.amazon.science/publications/amazon-nova-sonic-technical-report-and-model-card

## Notes

### tool use 기반 통합이 기본 패턴

Nova Sonic은 [[concepts/voice-ai-agent|Voice AI Agent]]를 만들 때 자체로 모든 로직을 감당하는 방식이 아니라, tool use로 외부 에이전트/서비스에 delegation하는 것을 전제로 한다. 이 점이 [[entities/amazon-bedrock-agentcore|Amazon Bedrock AgentCore]]와 결합해 [[concepts/multi-agent-voice-architecture|멀티 에이전트 보이스 아키텍처]]를 구성하는 기반이 된다.

### 오케스트레이터 역할

멀티 에이전트 구성에서 Nova Sonic은 음성 IO 담당이자 **오케스트레이터**로 동작한다. 사용자 발화를 들으면서 내장 reasoning으로 분류하고, 필요한 서브 에이전트에 tool use 이벤트를 발사한다. 응답이 돌아오면 TTS로 합성해 재생하므로 사용자는 핸드오프를 감지하지 못한다.

### 설계 시 유의점

- **지연 누적**: tool use를 통한 서브 에이전트 호출은 응답 지연을 만든다. 음성 UX 관점에서 각 핸드오프는 잠재적 병목.
- **tool description이 곧 라우팅 스펙**: 내장 분류기의 입력이므로, 트리거 예시 발화를 description에 자세히 넣는 것이 routing 품질을 좌우.
- **음성 친화 응답**: 서브 에이전트 system prompt에 "JSON 금지, 2~3문장 자연어"를 강제해야 TTS 재생이 자연스럽다.

## Related

- [[entities/amazon-bedrock-agentcore|Amazon Bedrock AgentCore]]
- [[entities/strands-agents|Strands Agents]]
- [[concepts/multi-agent-voice-architecture|멀티 에이전트 보이스 아키텍처]]
- [[concepts/voice-ai-agent|Voice AI Agent]]
- [[summaries/2026-04-21-nova-sonic-multi-agent-voice-assistant|Nova Sonic + AgentCore 멀티 에이전트 요약]]
