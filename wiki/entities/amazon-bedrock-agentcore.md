---
title: Amazon Bedrock AgentCore
type: entity
tags: [tool, ai, aws, agent, runtime]
sources: ["raw/articles/Building a multi-agent voice assistant with Amazon Nova Sonic and Amazon Bedrock AgentCore.md"]
created: 2026-04-21
updated: 2026-04-21
---

# Amazon Bedrock AgentCore

## Overview

Amazon Bedrock AgentCore는 AWS가 제공하는 **AI 에이전트 호스팅·실행 런타임**이다. [[entities/strands-agents|Strands Agents]] 같은 프레임워크로 작성된 에이전트 코드를 서버리스 방식으로 배포하고, HTTP 엔드포인트로 호출 가능한 형태로 노출한다. [[entities/amazon-nova-sonic|Amazon Nova Sonic]] 같은 음성 오케스트레이터가 tool use 이벤트로 호출할 수 있는 서브 에이전트의 런타임으로 자주 쓰인다.

## Key Facts

- **제공처**: AWS (Amazon Bedrock 플랫폼 구성 요소).
- **용도**: Strands/기타 프레임워크로 작성한 에이전트 코드를 호스팅해 외부에서 호출 가능하도록 함.
- **SDK**: Python 패키지 `bedrock_agentcore` (예: `BedrockAgentCoreApp`, `@app.entrypoint` 데코레이터).
- **진입점 모델**: `BedrockAgentCoreApp` 인스턴스에 `@app.entrypoint`로 함수를 등록하면 payload를 받아 응답을 반환하는 서버리스 핸들러가 된다.
- **공식 문서**: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agents-tools-runtime.html
- **제품 페이지**: https://aws.amazon.com/bedrock/agentcore/

## Notes

### 기본 사용 패턴

```python
from strands import Agent, tool
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from strands.models import BedrockModel

app = BedrockAgentCoreApp()
agent = Agent(tools=[...], model=BedrockModel(model_id="amazon.nova-lite-v1:0"), system_prompt=...)

@app.entrypoint
def banking_agent(payload):
    response = agent(json.dumps(payload))
    return response.message['content'][0]['text']

if __name__ == "__main__":
    app.run()
```

즉, 에이전트는 **stateless 요청/응답 핸들러**로 래핑된다. 멀티턴 컨텍스트가 필요한 stateful 에이전트는 외부 상태 저장소를 별도로 구성해야 한다.

### 역할 분리에서의 위치

[[concepts/multi-agent-voice-architecture|멀티 에이전트 보이스 아키텍처]]에서 AgentCore는 **서브 에이전트 런타임** 자리를 차지한다. Nova Sonic은 오케스트레이터/음성 IO, AgentCore에 배포된 각 에이전트는 특정 도메인(인증·뱅킹·모기지 등)의 전문가 역할이다.

### 모델 선택과 지연

AgentCore에 배포되는 서브 에이전트는 `BedrockModel(model_id=...)`로 Bedrock의 모델을 붙인다. 기사는 Nova Lite 같은 경량 모델을 기본값으로 추천한다 — 서브 에이전트 호출이 음성 응답 지연 경로에 들어 있기 때문이다.

## Related

- [[entities/amazon-nova-sonic|Amazon Nova Sonic]]
- [[entities/strands-agents|Strands Agents]]
- [[concepts/multi-agent-voice-architecture|멀티 에이전트 보이스 아키텍처]]
- [[summaries/2026-04-21-nova-sonic-multi-agent-voice-assistant|Nova Sonic + AgentCore 멀티 에이전트 요약]]
