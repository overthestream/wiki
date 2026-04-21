---
title: Amazon Nova Sonic + Bedrock AgentCore로 멀티 에이전트 보이스 어시스턴트 구축
type: summary
tags: [ai, voice-ai, aws, multi-agent, theory]
sources: ["raw/articles/Building a multi-agent voice assistant with Amazon Nova Sonic and Amazon Bedrock AgentCore.md", "https://aws.amazon.com/ko/blogs/machine-learning/building-a-multi-agent-voice-assistant-with-amazon-nova-sonic-and-amazon-bedrock-agentcore/"]
created: 2026-04-21
updated: 2026-04-21
---

# Amazon Nova Sonic + Bedrock AgentCore로 멀티 에이전트 보이스 어시스턴트 구축

## Source Info
- **출처**: AWS Machine Learning Blog (aws.amazon.com)
- **저자**: Lana Zhang (AWS Senior Specialist Solutions Architect, Generative AI)
- **날짜**: 2025-10-22 발행

## Summary

이 글은 [[entities/amazon-nova-sonic|Amazon Nova Sonic]]의 speech-to-speech 능력을 [[entities/amazon-bedrock-agentcore|Amazon Bedrock AgentCore]]와 결합하여 **도메인별로 특화된 서브 에이전트**를 가진 멀티 에이전트 보이스 어시스턴트를 구축하는 방법을 설명한다. 단일 모놀리식 보이스 에이전트가 기능 확장에 따라 시스템 프롬프트가 비대해지고 로직이 복잡해지는 문제를, 마이크로서비스 스타일의 분할로 해결하자는 관점이다.

예시는 뱅킹 보이스 어시스턴트다. Nova Sonic이 **음성 인터페이스 + 오케스트레이터** 역할을 맡고, Authenticate/Banking/Mortgage 세 서브 에이전트를 AgentCore Runtime에 배포한다. 서브 에이전트는 [[entities/strands-agents|Strands Agents]] 프레임워크로 작성하며, 각자 Nova Lite 같은 경량 모델과 전용 tool을 가진다. 사용자 입장에서는 지연·목소리 변경·핸드오프가 드러나지 않는 단일 대화 경험이 유지된다.

Nova Sonic과 서브 에이전트의 연결은 **tool use 이벤트**로 이뤄진다. `promptStart` 이벤트에서 tool 스펙(예: `bankAgent`, 입력 스키마 `accountId`/`query`)을 주입하면, Nova Sonic 내장 reasoning 모델이 발화를 분류해 해당 `toolUse` 이벤트를 클라이언트로 쏜다. 클라이언트가 AgentCore에 호스팅된 Strands 에이전트를 호출하고 응답을 돌려주면, Nova Sonic이 음성으로 합성해 재생한다. 서브 에이전트의 시스템 프롬프트는 "JSON 반환 금지, 2~3문장 자연어로 요약"처럼 **음성 친화적 출력**을 강제한다.

마지막에 저자는 음성 멀티 에이전트 설계의 실전 지침을 제시한다: ① 유연성과 지연의 균형(핸드오프마다 지연 누적), ② 서브 에이전트에는 Nova Lite 같은 작은 모델 우선, ③ 음성 응답은 짧고 초점 있게. 또한 **stateless vs stateful 서브 에이전트** 선택 기준을 정리한다 — 단순/일회성 작업은 stateless, 멀티턴·세션 캐싱이 필요하면 stateful.

## Key Takeaways

- **Nova Sonic = 음성 IO + 오케스트레이터**, AgentCore = 서브 에이전트 런타임. 이 역할 분리가 핵심이다.
- 모놀리식 보이스 에이전트의 한계(시스템 프롬프트 비대, 유지보수 난이도)는 텍스트 챗봇의 그것과 동일하며, [[concepts/multi-agent-voice-architecture|멀티 에이전트 아키텍처]]로 풀린다.
- Nova Sonic과 서브 에이전트의 접점은 **tool use**다. tool 스펙의 `description`이 Nova Sonic 내장 분류기의 입력이 된다 — 자연어 트리거 예시를 description에 넣는 것이 실무 팁.
- 서브 에이전트는 **자체 입력 검증**(예: 계좌 ID 형식)을 책임져 Nova Sonic의 reasoning 부담을 덜어야 한다. 소프트웨어 모듈 설계와 동일한 캡슐화 원칙.
- **지연이 1급 제약**이다. 핸드오프마다 latency가 쌓이므로 서브 에이전트에는 Nova Lite 등 경량 모델을 기본값으로 두고, 큰 모델은 복잡 reasoning에만.
- 서브 에이전트 system prompt에 "**natural language, 2~3 sentences, no raw JSON**"을 강제하는 것이 음성 UX의 최소 조건.
- Stateless 서브 에이전트가 기본. 멀티턴 컨텍스트가 필요한 경우에만 stateful로, 외부 상태 관리 비용을 감수한다.

## Quotes & References

- 저자의 핵심 비유: *"building a team of specialized AI assistants rather than relying on a single do-it-all helper"* — 회사의 부서 분할과 마이크로서비스의 언어로 멀티 에이전트를 설명한다.
- Nova Sonic `toolUse` 이벤트 페이로드 예:
  ```json
  {
    "event": {
      "toolUse": {
        "toolName": "bankAgent",
        "content": "{\"accountId\":\"one two three four five\",\"query\":\"check account balance\"}",
        "role": "TOOL", "toolUseId": "UUID"
      }
    }
  }
  ```
  — `accountId`가 숫자 그대로가 아닌 *"one two three four five"* 형태로 전달된다는 점은 음성 STT 결과가 그대로 흘러오는 것임을 암시한다. 서브 에이전트 쪽에서 숫자 정규화가 필요.
- 샘플 코드 저장소: https://github.com/aws-samples/amazon-nova-samples/tree/main/speech-to-speech/workshops/agent-core
- 관련 워크샵 랩: https://catalog.workshops.aws/amazon-nova-sonic-s2s/02-repeatable-pattern/05-multi-agent-agentcore

## Related

- [[entities/amazon-nova-sonic|Amazon Nova Sonic]]
- [[entities/amazon-bedrock-agentcore|Amazon Bedrock AgentCore]]
- [[entities/strands-agents|Strands Agents]]
- [[concepts/multi-agent-voice-architecture|멀티 에이전트 보이스 아키텍처]]
- [[concepts/voice-ai-agent|Voice AI Agent]]
- [[concepts/stt-llm-tts-pipeline|STT-LLM-TTS Pipeline]]
