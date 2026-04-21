---
title: Strands Agents
type: entity
tags: [tool, ai, agent, framework, python, open-source]
sources: ["raw/articles/Building a multi-agent voice assistant with Amazon Nova Sonic and Amazon Bedrock AgentCore.md"]
created: 2026-04-21
updated: 2026-04-21
---

# Strands Agents

## Overview

Strands Agents는 LLM 기반 에이전트를 Python으로 작성하기 위한 프레임워크다. 에이전트의 핵심 구성 요소(모델, 시스템 프롬프트, 도구 집합)를 명시적으로 조립하는 API를 제공하며, Bedrock·다른 모델 제공자를 모델 백엔드로 선택할 수 있다. [[entities/amazon-bedrock-agentcore|Amazon Bedrock AgentCore]]의 서브 에이전트를 작성할 때 AWS 공식 예시에서 자주 쓰이는 선택지다.

## Key Facts

- **언어**: Python.
- **핵심 API**: `Agent`, `@tool` 데코레이터, `strands.models.BedrockModel` 등.
- **모델 백엔드**: `BedrockModel(model_id="amazon.nova-lite-v1:0")`처럼 Bedrock 모델을 연결하는 방식이 AWS 샘플의 기본 형태.
- **공식 문서**: https://strandsagents.com/latest/documentation/docs/

## Notes

### 에이전트 정의 패턴

```python
from strands import Agent, tool
from strands.models import BedrockModel

@tool
def get_account_balance(account_id) -> str:
    """Get account balance for given account Id"""
    return {"result": result}

bedrock_model = BedrockModel(model_id="amazon.nova-lite-v1:0")
agent = Agent(tools=[get_account_balance], model=bedrock_model, system_prompt=...)
```

Python 함수에 `@tool` 데코레이터를 붙이면 LLM이 호출 가능한 도구로 등록된다. docstring과 시그니처가 tool 스펙으로 활용된다.

### 배포 패턴

Strands로 쓴 `Agent`를 AgentCore의 `BedrockAgentCoreApp.entrypoint`에 감싸면 서버리스 HTTP 서비스가 된다. 즉, Strands는 에이전트 **정의**를, AgentCore는 **호스팅**을 책임지는 역할 분담이다.

## Related

- [[entities/amazon-bedrock-agentcore|Amazon Bedrock AgentCore]]
- [[entities/amazon-nova-sonic|Amazon Nova Sonic]]
- [[concepts/multi-agent-voice-architecture|멀티 에이전트 보이스 아키텍처]]
- [[summaries/2026-04-21-nova-sonic-multi-agent-voice-assistant|Nova Sonic + AgentCore 멀티 에이전트 요약]]
