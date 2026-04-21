---
title: Using realtime models (OpenAI Realtime API) 요약
type: summary
tags: [ai, voice-ai, openai, prompting]
sources: ["raw/articles/Using realtime models  OpenAI API.md", "https://developers.openai.com/api/docs/guides/realtime-models-prompting"]
created: 2026-04-21
updated: 2026-04-21
---

# Using realtime models (OpenAI Realtime API) 요약

## Source Info
- **출처**: OpenAI Developers Docs (`/api/docs/guides/realtime-models-prompting`)
- **저자**: OpenAI (공식 가이드)
- **날짜**: 2026-04-21 clipping (발행일 미표기)

## Summary

이 문서는 OpenAI의 주력 speech-to-speech 모델인 `gpt-realtime`을 **프롬프팅으로 어떻게 잘 쓸지**를 다룬다. 모델이 이전 세대와 다르게 동작한다는 전제하에, 세션을 연 뒤 `session.update` client event로 서버 저장 prompt를 주입하는 방식을 설명한다. prompt id·version·variables·instructions를 어떻게 조합하는지, 그리고 mid-call 스왑이 어떻게 동작하는지(변수 값·instruction override까지) 코드 예시로 보인다.

본론은 `gpt-realtime` 전용의 **10가지 프롬프팅 팁**이다. 핵심 아이디어는 "작은 문구 변화가 큰 행동 변화를 만든다"는 instruction following 감도를 인정하고, 그에 맞춰 **불릿 중심 구조, 섹션화된 시스템 프롬프트(Role/Tone/Context/...), 샘플 문구 제공, 언어 고정, 반복 회피 규칙, 대문자 강조, tool preamble, LLM으로 프롬프트 리뷰, 속도·에스컬레이션 규칙**까지 실전 패턴을 제시한다. 자세한 쿡북 링크로 확장된다.

마지막은 관련 가이드 목록(Inputs/outputs, Managing conversations, Webhooks, Costs, Function calling, MCP, Transcription, Voice agents SDK)으로 이어진다. 이 문서는 **"모델이 어떻게 동작하는지"보다 "무엇을 써야 모델이 말을 잘 듣는지"에 초점**을 둔 프롬프팅 가이드라는 점이 구별되는 포인트.

## Key Takeaways

- `gpt-realtime`은 **instruction following에 매우 민감**. "inaudible" → "unintelligible" 같은 미묘한 단어 교체가 노이즈 처리 품질을 바꾼다.
- **섹션화된 프롬프트 템플릿** (Role/Tone/Context/Pronunciations/Tools/Rules/Flow/Safety)이 공식 권장 구조. 도메인별 섹션 추가 가능.
- **짧은 bullet > 긴 문단**이 구어 모델에는 특히 유효. 수치 조건도 "IF MORE THAN THREE FAILURES"처럼 텍스트로 번역.
- 언어 드리프트 방지용 전용 `## Language` 섹션, 방언까지 고정 가능. 언어 교습처럼 역할별로 Explanations/Conversation 분리도 가능.
- 로봇 같은 반복은 `## Variety` 섹션으로 완화. **sample phrases는 다양하게 제공**해야 복제 과정에서 자연스러움이 유지.
- Tool 사용 시 **preamble 발화**("I'm checking that now.")가 음성 UX에서 필수. 무음은 통화에서 죽음.
- Prompt는 다른 LLM에게 **비평받는 워크플로우**가 기본. OpenAI가 제공하는 "Prompt-Critique Expert" 역할 프롬프트가 템플릿.
- Pacing과 에스컬레이션은 프롬프트 레벨에서 해결. 2–3 문장 길이, 자동 에스컬레이션 트리거 조건을 못 박아둔다.
- **`session.prompt.id` + `variables`** 구조 덕분에 prompt 버전 관리와 변수 주입을 API 레벨에서 할 수 있다. 코드 재배포 없이 A/B 실험 가능.

## Quotes & References

- *"Swapping 'inaudible' → 'unintelligible' improved noisy input handling."* — instruction following 감도를 상징.
- *"IF [func.return_value] IS BIGGER THAN 0, RESPOND 1 TO THE USER."* — 수치→텍스트 번역 + 대문자 강조 예.
- 에스컬레이션 룰 예: *"2 failed tool attempts on the same task OR 3 consecutive no-match/no-input events."*
- 쿡북: https://developers.openai.com/cookbook/examples/realtime_prompting_guide

## Related

- [[entities/openai-realtime-api|OpenAI Realtime API]]
- [[concepts/voice-agent-prompting|Voice Agent Prompting]]
- [[concepts/voice-ai-agent|Voice AI Agent]]
- [[concepts/realtime-session-event-model|Realtime Session/Event Model]]
