---
title: LiveKit Agents 프레임워크 공식 문서 요약
type: summary
tags: [ai, tool, realtime, voice-ai]
sources: ["raw/articles/LiveKit Documentation.md", "https://docs.livekit.io/agents/"]
created: 2026-04-21
updated: 2026-04-21
---

# LiveKit Agents 프레임워크 공식 문서 요약

## Source Info
- **출처**: https://docs.livekit.io/agents/
- **저자**: LiveKit (공식 문서)
- **날짜**: 2026-04-21 수집

## Summary

LiveKit Agents는 Python 또는 Node.js 프로그램을 **LiveKit Room의 풀 리얼타임 참여자**로 연결하는 프레임워크다. AI 파이프라인을 통해 실시간 미디어/데이터를 주고받고, 결과를 다시 Room에 publish하는 것을 목적으로 한다. 코드로 직접 구축하거나 **LiveKit Agent Builder**(노코드 브라우저 도구)로 프로토타이핑할 수 있다. 실행 환경은 LiveKit Cloud(매니지드 배포, 옵저버빌리티 내장, LiveKit Inference로 키리스 모델 사용 가능) 또는 커스텀 환경 양쪽을 지원한다.

프레임워크의 본질은 **AI 모델과 사용자 사이의 상태ful 리얼타임 브리지**다. AI 모델은 안정된 데이터센터에 있지만 사용자는 품질이 일정치 않은 모바일 네트워크에서 접속하기 때문에, WebRTC를 활용해 불안정한 연결에서도 원활한 통신을 보장한다. 프론트엔드 ↔ 에이전트는 LiveKit WebRTC로, 에이전트 ↔ 백엔드는 HTTP와 WebSocket으로 통신하는 구조다. 에이전트 SDK는 STT-LLM-TTS 파이프라인, 턴 감지, 인터럽션 처리, LLM 오케스트레이션 등 리얼타임 보이스 AI의 핵심 과제를 해결하는 컴포넌트를 포함한다.

에이전트의 연결 수명 주기는 다음과 같다: 에이전트 코드가 시작되면 먼저 LiveKit 서버(셀프호스팅 또는 Cloud)에 "agent server" 프로세스로 등록된다. 이 서버는 디스패치 요청을 대기하다가 요청을 받으면 "job" 서브프로세스를 부트하고 Room에 참가시킨다. 기본 동작은 **새 Room이 만들어질 때마다 에이전트가 자동 디스패치**되는 것이다.

프레임워크의 핵심 특징은 **음성·영상·텍스트 멀티모달 처리**, **LLM 중립적 도구 사용**, **멀티 에이전트 핸드오프**, **광범위한 프로바이더 플러그인**, **자체 턴 감지 모델**, **코드 중심 빌드**, **프로덕션 준비(로드밸런싱, Kubernetes 호환)**, **Apache 2.0 오픈소스**다. 주요 사용 사례로는 멀티모달 어시스턴트, 원격의료, 콜센터, 실시간 번역, 게임 NPC, 로보틱스(클라우드 뇌)가 제시된다.

핵심 개념은 네 축으로 정리된다: **Multimodality**(음성/텍스트/비전 통합), **Logic & structure**(세션, 태스크, 워크플로우, 도구, 파이프라인 노드, 핸드오프), **Agent server**(수명주기, 디스패치, 스케일링), **Models**(LLM/STT/TTS/Realtime API/아바타 — LiveKit Inference 또는 플러그인).

## Key Takeaways

- LiveKit Agents의 정체성은 **"LiveKit Room의 리얼타임 참여자로서의 AI"**다. 일반적인 HTTP API 기반 챗봇과 달리 WebRTC 세션 안에 에이전트가 join한다.
- 아키텍처 분리: **프론트엔드 ↔ 에이전트는 WebRTC**, **에이전트 ↔ 백엔드는 HTTP/WebSocket**. 네트워크 불안정성은 WebRTC가, 비즈니스 로직은 기존 HTTP 스택이 담당.
- 에이전트 서버 모델: **"agent server → dispatch → job subprocess"** 구조. 디폴트는 Room 생성 시마다 자동 디스패치. 이는 프로세스 격리와 수평 확장을 함께 풀어내는 설계.
- STT-LLM-TTS 파이프라인을 SDK 레벨에서 추상화하고, **턴 감지 전용 모델**을 자체 제공한다는 점이 경쟁 프레임워크 대비 차별점.
- **LiveKit Inference**는 "API 키 없이 Cloud 환경 내에서 모델 사용"을 제공해 벤더 로크인과 키 관리를 함께 해결하려는 움직임.
- 텔레포니(SIP) 통합이 1급 시민으로 취급됨 — 콜센터·회사 안내 같은 전통적 음성 UX를 AI로 대체하는 경로가 열려있음.
- 활용 범위가 **"로보틱스: 로봇의 뇌를 클라우드에"**까지 확장됨. 즉 프레임워크 타겟이 단순 보이스봇을 넘어 물리 AI까지 포괄.

## Quotes & References

> "Your agent code operates as a stateful, realtime bridge between powerful AI models and your users."

> "When your agent code starts, it first registers with a LiveKit server ... to run as an 'agent server' process. The agent server waits until it receives a dispatch request. To fulfill this request, the agent server boots a 'job' subprocess which joins the room."

주요 레퍼런스:
- GitHub: https://github.com/livekit/agents
- Python SDK reference: https://docs.livekit.io/reference/python/livekit/agents/index.html
- 대표 예제: Medical Office Triage, Restaurant Agent, Company Directory, Pipeline Translator

라이선스: Apache 2.0

## Related

- [[entities/livekit|LiveKit]]
- [[concepts/voice-ai-agent|Voice AI Agent]]
- [[concepts/stt-llm-tts-pipeline|STT-LLM-TTS Pipeline]]
