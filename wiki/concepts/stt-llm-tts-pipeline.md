---
title: STT-LLM-TTS Pipeline
type: concept
tags: [ai, theory, voice-ai, pipeline]
sources: ["raw/articles/LiveKit Documentation.md"]
created: 2026-04-21
updated: 2026-04-21
---

# STT-LLM-TTS Pipeline

## Definition

STT-LLM-TTS 파이프라인은 **음성 입력을 텍스트로 변환(STT) → 언어 모델로 처리(LLM) → 텍스트를 음성으로 합성(TTS)** 하는 3단계 직렬 처리 구조다. 현대 [[concepts/voice-ai-agent|Voice AI Agent]]의 가장 일반적인 레퍼런스 아키텍처로, 각 단계를 독립 프로바이더로 교체할 수 있다는 유연성 때문에 "cascade(캐스케이드) 파이프라인"이라고도 불린다.

## Key Ideas

### 3단계 구조

1. **STT (Speech-to-Text)**: 사용자의 오디오 스트림을 텍스트로 전사. 스트리밍 STT는 완결된 문장이 아닌 부분 전사(partial transcripts)를 실시간으로 방출한다.
2. **LLM (Large Language Model)**: 전사 텍스트 + 대화 히스토리 + 시스템 프롬프트 + 도구 정의를 받아 응답 텍스트(또는 tool call)를 생성. 스트리밍 토큰 출력이 일반적.
3. **TTS (Text-to-Speech)**: LLM이 생성한 텍스트를 음성 오디오로 합성. 스트리밍 TTS는 LLM 토큰이 도착하는 대로 합성을 시작해 end-to-end 지연을 줄인다.

### 왜 파이프라인인가 — Realtime API와의 비교

OpenAI Realtime API 같은 "음성→음성 직접" 모델이 등장했지만, cascade 파이프라인은 여전히 다음 이유로 주류다:

- **프로바이더 선택의 자유**: STT는 Deepgram, LLM은 Claude/GPT, TTS는 ElevenLabs처럼 각 단계의 최적 모델을 조합할 수 있다.
- **관찰 가능성**: 중간 단계의 텍스트가 그대로 로그·전사로 남아 디버깅과 평가가 쉽다.
- **비용 통제**: 각 단계의 가격이 독립적이라 모델 다운그레이드가 용이.
- **도구 사용 친화**: LLM 출력이 텍스트/JSON이므로 전통적 function calling 생태계와 즉시 호환.

단점은 **누적 지연**이다. STT 끝 → LLM 첫 토큰 → TTS 첫 오디오까지의 chain 지연이 사용자 체감 응답성을 결정한다. 따라서 각 단계를 **스트리밍으로 맞물리게 하는 것**이 구현의 핵심.

### 턴 감지 (Turn Detection)가 파이프라인의 선행 관문

파이프라인이 언제 "한 턴이 끝났다"고 판단해야 STT 결과를 LLM에 flush할 수 있다. 단순 VAD(묵음 감지)로는 말 중간 숨 쉬는 구간을 오인하기 쉬워, LiveKit Agents는 **전용 턴 감지 모델**을 별도로 제공한다. 이는 cascade 파이프라인의 품질이 STT/LLM/TTS 자체보다 "언제 파이프라인을 trigger하는가"에서 결정된다는 실무적 교훈을 반영한다.

### 인터럽션 처리

사용자가 에이전트 TTS 재생 중에 말하기 시작하면:
1. TTS 재생 즉시 중단
2. 진행 중인 LLM 스트리밍도 취소 또는 무시
3. 새 STT 입력으로 파이프라인 재시작
4. 대화 히스토리에는 중단된 지점까지만 기록 (모델이 "내가 어디서 끊겼는지" 인지해야 자연스러움)

### 도구 사용과 프론트엔드 포워딩

LLM 단계에서 tool call이 발생하면 백엔드에서 실행할 수도 있지만, **프론트엔드로 포워딩**하는 패턴이 있다. 예: 화면에 폼을 띄우거나 로컬 센서 데이터를 읽는 경우. LiveKit Agents는 이 패턴을 1급으로 지원한다.

## Examples

- **가장 단순한 조합**: Deepgram(STT) + GPT-4o-mini(LLM) + Cartesia(TTS).
- **프로덕션 조합**: LiveKit Inference로 다중 프로바이더를 키리스로 조합 + 자체 턴 감지 모델.
- **멀티 에이전트 확장**: 트리아지 에이전트 → 전문 에이전트로 handoff 시, 파이프라인의 LLM 단계만 교체되고 STT/TTS는 공유.

## Related

- [[concepts/voice-ai-agent|Voice AI Agent]]
- [[entities/livekit|LiveKit]]
- [[summaries/2026-04-21-livekit-documentation|LiveKit Agents 프레임워크 공식 문서 요약]]
