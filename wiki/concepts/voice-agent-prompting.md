---
title: Voice Agent Prompting
type: concept
tags: [ai, theory, voice-ai, prompting, ux]
sources: ["raw/articles/Using realtime models  OpenAI API.md"]
created: 2026-04-21
updated: 2026-04-21
---

# Voice Agent Prompting

## Definition

**Voice Agent Prompting**은 speech-to-speech 모델(예: `gpt-realtime`)에 대한 시스템 프롬프트 설계 기법이다. 텍스트 챗봇 프롬프팅과 공통 원칙을 공유하지만, **오디오 출력의 자연스러움·지연·인터럽션·음성 특유의 실패 모드**에 맞춘 구체 규칙이 추가된다. OpenAI Realtime 가이드의 10가지 팁을 기반으로 정리한다.

## Key Ideas

### 구조화된 섹션 프롬프트 템플릿

OpenAI 공식 추천 구조:

```markdown
# Role & Objective
# Personality & Tone
# Context
# Reference Pronunciations
# Tools
# Instructions / Rules
# Conversation Flow
# Safety & Escalation
```

섹션을 라벨링해 두면 모델이 참조하기 쉽고 **사람이 문제 구간만 수술하듯 고치기 쉬움**. 도메인별 섹션(Compliance, Brand Policy) 추가는 권장.

### 10가지 실전 팁

1. **정확히, 충돌 제거**: 문구 하나로 행동이 크게 흔들린다. 예: "inaudible" → "unintelligible"로 바꿔 노이즈 처리가 크게 개선. 초안 작성 후 LLM으로 ambiguity/conflict 리뷰.
2. **불릿 > 문단**: 장문보다 짧고 명확한 bullet이 따라가기 쉽다.
3. **불명확한 오디오 처리 명시**: 언제 되묻을지, 어떤 샘플 문구를 쓸지, 기본 언어는 뭔지. `Sample clarification phrases` 섹션을 둬서 다양한 문구 제공.
4. **언어 고정**: 언어 드리프트가 있으면 전용 `## Language` 섹션. 예: "Respond in the same language as the user unless directed otherwise"에서부터 방언 고정까지.
5. **샘플 문구와 대화 흐름 제공**: `Greeting → Discover → Verify → Diagnose → Resolve → Confirm/Close` 같은 phase graph를 주고, 각 phase의 goal·sample phrases·exit 조건 명시. 모델이 샘플 스타일을 강하게 복제한다.
6. **로봇 같은 반복 방지**: `## Variety` 섹션에 "Do not repeat the same sentence twice" 명시.
7. **대문자로 강조**: 중요한 규칙은 ALL CAPS. 수치 조건은 텍스트로 번역 — "IF MORE THAN THREE FAILURES THEN ESCALATE" 쪽이 "IF x > 3"보다 낫다.
8. **도구 사용 안내(preamble)**: tool 호출 직전에 "I'm checking that now." 같은 짧은 한 줄을 뱉고 바로 호출. 사용자가 무음에서 "모델이 죽었나" 오해하지 않게 함.
9. **LLM으로 프롬프트 비평**: 전용 "Prompt-Critique Expert" 프롬프트로 ambiguity/lacking definitions/conflicts/unstated assumptions를 찾게 함. 출력 포맷 고정(Issues/Improvements/Revised Prompt)이 유용.
10. **속도와 에스컬레이션**: `Personality & Tone`에 pacing 규칙(2–3 sentences, 빠르게 말하되 rushed처럼 들리지 않게)과 `Safety & Escalation`에 에스컬레이션 트리거(자해 위험, 명시적 요청, 2회 tool 실패, 3회 no-match 등)와 MANDATORY 발화문을 고정.

### 음성 UX 특유의 제약

- **TTS 친화 출력**: 짧은 문장, 구어체, 숫자·약어 읽기 방식(pronunciation guide)이 중요.
- **지연 민감성**: 긴 서문은 체감 지연을 키운다. pacing·길이 제한이 기능이다.
- **에스컬레이션 경로 필수**: 텍스트 봇보다 "사람 연결"의 긴급성이 높다. 실패 누적 기반 자동 에스컬레이션이 기본값.

### 서버 저장 prompt + variables

Realtime API는 `session.prompt.id`로 서버 저장 prompt를 참조하고 `variables`로 파라미터 주입 가능. 중간에 `session.update`로 버전/변수 스왑 → **A/B 실험과 페르소나 전환이 코드 배포 없이 가능**.

## Examples

- 언어 교습 앱: `## Language`에 Explanations(영어)/Conversation(프랑스어) 분리.
- 인터넷 고객지원 봇: Greeting 섹션 샘플 문구 3개 + "Caller states an initial goal" exit 조건.
- 에스컬레이션 규칙: "2 failed tool attempts on the same task OR 3 consecutive no-match/no-input events" → `escalate_to_human` 도구 + 고정 발화 "Thanks for your patience—I'm connecting you with a specialist now."

## Related

- [[entities/openai-realtime-api|OpenAI Realtime API]]
- [[concepts/voice-ai-agent|Voice AI Agent]]
- [[concepts/realtime-session-event-model|Realtime Session/Event Model]]
- [[summaries/2026-04-21-openai-realtime-using-models|Using realtime models 요약]]
