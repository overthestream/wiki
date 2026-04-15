---
title: Activity Log
type: log
updated: 2026-04-15
---

# Activity Log

위키 활동의 시간순 기록. 각 항목은 `## [날짜] action | 제목` 형식.

---

## [2026-04-15] init | Wiki 초기 설정
- LLM Wiki 패턴 기반 Obsidian vault 구성
- 디렉토리 구조, 스키마(CLAUDE.md), 인덱스, 로그, 템플릿 생성

## [2026-04-15] ingest | Java Virtual Threads 공식 문서
- 원본: raw/articles/java-virtual-threads.md
- 요약: Oracle Java 21 공식 문서 기반 Virtual Thread 개요 — 생성, 스케줄링, pinning, 디버깅, 도입 가이드라인
- 새 페이지: [[summaries/2026-04-15-java-virtual-threads]], [[concepts/virtual-thread]], [[concepts/thread-pinning]], [[concepts/jvm-monitor]]

## [2026-04-15] ingest | Virtual Thread 심층 Q&A
- 원본: raw/articles/virtual-thread-QnA.md
- 요약: JVM 모니터 하드웨어 레벨 분석, JDK 24 pinning 해결, Semaphore 패턴, JDBC/MongoDB 드라이버 호환성
- 새 페이지: [[summaries/2026-04-15-virtual-thread-qna]]
- 업데이트: [[concepts/virtual-thread]], [[concepts/thread-pinning]], [[concepts/jvm-monitor]]

## [2026-04-15] ingest | SimpleAsyncTaskExecutor API
- 원본: raw/articles/SimpleAsyncTaskExecutor (Spring Framework 7.0.6 API).md
- 요약: Spring의 TaskExecutor 구현체 — Virtual Thread 모드, 동시성 제한, graceful shutdown
- 새 페이지: [[summaries/2026-04-15-simple-async-task-executor]]
- 업데이트: [[concepts/virtual-thread]]

## [2026-04-15] ingest | 모니터 개념 정리
- 원본: raw/articles/Monitors.md
- 요약: 동시성 프로그래밍의 모니터 개념 — 세마포어 한계, 모니터 구조, Hoare vs Mesa, Java synchronized와의 관계
- 새 페이지: [[summaries/2026-04-15-monitors]], [[concepts/monitor]], [[concepts/semaphore]]
- 업데이트: [[concepts/jvm-monitor]], [[concepts/virtual-thread]]
