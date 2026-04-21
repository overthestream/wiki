---
title: LiveKit
type: entity
tags: [tool, ai, realtime, webrtc, open-source]
sources: ["raw/articles/LiveKit Documentation.md", "https://docs.livekit.io/agents/"]
created: 2026-04-21
updated: 2026-04-21
---

# LiveKit

## Overview

LiveKit은 음성/영상/데이터 리얼타임 통신 인프라와 그 위에 올라가는 AI 에이전트 프레임워크를 제공하는 오픈소스 플랫폼이다. WebRTC 기반 미디어 전송을 1급 시민으로 삼고, 여기에 STT-LLM-TTS 파이프라인·턴 감지·멀티모달 처리 같은 보이스 AI용 추상화를 얹은 것이 특징이다. 셀프호스팅과 매니지드(LiveKit Cloud) 양쪽을 지원한다.

## Key Facts

- **라이선스**: Apache 2.0 (프레임워크 및 전체 LiveKit 생태계).
- **핵심 제품**: LiveKit (WebRTC 인프라), LiveKit Agents (AI 에이전트 프레임워크), LiveKit Cloud (매니지드 호스팅), LiveKit Inference (키리스 모델 호출), LiveKit Agent Builder (노코드 브라우저 도구), SIP 기반 텔레포니 통합.
- **SDK 언어**: Python, Node.js (에이전트 작성용).
- **통신 모델**: 프론트엔드 ↔ 에이전트는 WebRTC, 에이전트 ↔ 백엔드는 HTTP/WebSocket.
- **배포 모델**: LiveKit Cloud(글로벌 인프라, 옵저버빌리티·전사·트레이스 내장) 또는 Kubernetes 호환 커스텀 환경.
- **GitHub**: https://github.com/livekit/agents
- **공식 문서**: https://docs.livekit.io/

## Notes

### Agents 프레임워크의 구조적 포지셔닝

일반적인 챗봇 프레임워크가 HTTP 기반 요청/응답을 가정하는 반면, LiveKit Agents는 **에이전트를 "Room의 참여자"로 모델링**한다. 사용자와 에이전트 모두 같은 LiveKit Room에 join하고, 미디어·데이터를 양방향으로 publish/subscribe한다. 이는 음성 인터럽션, 턴 감지, 멀티모달 입력(음성+화면공유) 같은 요구사항이 프로토콜 레벨에서 자연스럽게 풀린다는 이점을 준다.

### 수명주기 모델

에이전트 코드는 기동 시 LiveKit 서버에 "agent server"로 등록하고 디스패치를 대기한다. 요청이 오면 "job" 서브프로세스가 스폰되어 Room에 참가하는 프로세스-격리형 구조다. 기본값은 **Room 생성 시마다 자동 디스패치**. 로드밸런싱과 graceful shutdown이 내장되어 있다.

### 적용 도메인

LiveKit이 문서에서 전면에 내세우는 활용처는 일반 보이스봇을 넘어선다:

- 멀티모달 어시스턴트 (음성/텍스트/화면공유 동시)
- 원격의료 (휴먼 인 더 루프 포함)
- 콜센터 (인바운드/아웃바운드)
- 실시간 번역
- 게임 NPC (스크립트가 아닌 LLM 기반)
- 로보틱스 ("로봇의 뇌를 클라우드에")

### 대표 예제 (Recipes)

- **Medical Office Triage**: 증상·병력 기반 환자 트리아지.
- **Restaurant Agent**: 주문 접수, 공유 카트, 결제까지 진행하는 프론트 오피스 에이전트.
- **Company Directory**: DTMF 톤과 음성 프롬프트에 반응해 호 전환.
- **Pipeline Translator**: 처리 파이프라인 내 번역 구현.

## Related

- [[concepts/voice-ai-agent|Voice AI Agent]]
- [[concepts/stt-llm-tts-pipeline|STT-LLM-TTS Pipeline]]
- [[summaries/2026-04-21-livekit-documentation|LiveKit Agents 프레임워크 공식 문서 요약]]
