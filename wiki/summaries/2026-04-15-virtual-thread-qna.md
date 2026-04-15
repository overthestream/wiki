---
title: "Virtual Thread 심층 Q&A"
type: summary
tags: [java, concurrency, jvm, virtual-thread, jdbc, mongodb]
sources: [raw/articles/virtual-thread-QnA.md]
created: 2026-04-15
updated: 2026-04-15
---

# Virtual Thread 심층 Q&A

## Source Info
- **출처**: https://claude.ai/share/eb7c40c0-8f48-4bd2-8806-fc3c86ecc795
- **저자**: Claude 대화
- **날짜**: 2026-04-15

## Summary

Java [[concepts/virtual-thread|Virtual Thread]]에 대한 심층 Q&A로, 기본 개념부터 하드웨어 레벨까지 확장한다. Oracle 공식 문서를 기반으로 하되 실무 적용 관점에서 더 깊이 있는 내용을 다룬다.

### 기본 개념 및 스케줄링
Java 21 이전에는 모든 `Thread`가 OS 스레드와 1:1인 Platform Thread였다. Virtual Thread 도입으로 `CPU Core ↔ OS 스레드 ↔ Carrier(Platform Thread) ↔ Virtual Thread`의 M:N:M:N 멀티플렉싱 구조가 되었다. 기존 Platform Thread + 블로킹 프로그래밍에서는 I/O 시 OS 스레드가 WAITING/BLOCKED 상태로 전환되어 "비싼 대기"가 발생했으나, Virtual Thread에서는 JVM이 캐리어에서 unmount하여 다른 Virtual Thread가 사용할 수 있게 한다.

### JVM 모니터와 Pinning
[[concepts/jvm-monitor|JVM Monitor]] 메커니즘을 하드웨어 레벨까지 분석한다. Java 객체의 Mark Word(64비트)에 락 상태가 기록되며, 경합 정도에 따라 Biased → Lightweight(CAS) → Heavyweight(ObjectMonitor/OS mutex) 순으로 에스컬레이션된다. Lightweight Lock의 CAS는 x86의 `lock cmpxchg` 명령어로 매핑되며, MESI 프로토콜로 캐시 일관성을 보장한다.

[[concepts/thread-pinning|Thread Pinning]]의 본질은 ObjectMonitor의 `_owner`가 OS 스레드(캐리어)에 묶이는 것이다. JDK 24(JEP 491)에서는 모니터 소유권을 Virtual Thread ID로 추적하도록 변경하여 `synchronized` 내부에서도 unmount가 가능해졌다.

### 실무 적용
- **Tomcat**: `spring.threads.virtual.enabled=true` 설정으로 Virtual Thread 활성화 가능
- **Semaphore**: 외부 자원 접근만 선택적으로 제한. 스레드 풀과 달리 나머지 코드는 자유롭게 실행
- **JDBC 드라이버 호환성**: PostgreSQL(42.6.0+), MariaDB(3.3.0+), Oracle(21c+), MS SQL(12.2.0+) 지원. MySQL은 대응이 가장 느림
- **MongoDB**: Java 드라이버 v4.11+에서 공식 지원. pinning 방지를 위한 내부 업데이트 완료

## Key Takeaways

- Virtual Thread는 OS 스레드/애플리케이션 스레드 간 멀티플렉싱을 한 계층 더 추가한 구조
- `synchronized`가 Virtual Thread에서 문제인 이유: ObjectMonitor._owner가 OS 스레드에 귀속 → 캐리어 분리 불가
- `ReentrantLock`은 AQS 기반으로 `LockSupport.park()`를 사용하여 pinning 없이 블로킹 가능
- JDK 24(JEP 491)에서 `synchronized` pinning 근본적 해결 — 모니터가 Virtual Thread ID를 직접 추적
- Semaphore vs Thread Pool: Semaphore는 특정 자원 접근 지점만 선택적으로 제한, Pool은 모든 단계를 제한
- `CompletableFuture` 비동기 스타일 대신 동기 블로킹 + Virtual Thread가 더 간단하고 디버깅이 쉬움
- JVM 락 에스컬레이션: Biased Lock → Lightweight Lock(CAS/스핀) → Heavyweight Lock(OS mutex/futex)

## Quotes & References

멀티플렉싱 계층 구조:
```
[기존]     CPU Core ←─M:N─→ OS 스레드 ←─1:1─→ Java Platform Thread
[VT 도입]  CPU Core ←─M:N─→ OS 스레드 ←─1:1─→ Carrier ←─M:N─→ Virtual Thread
```

JDK 21 vs JDK 24 모니터 소유권:
```
// JDK 21: ObjectMonitor._owner = carrier_thread_id  (OS 스레드에 묶임)
// JDK 24: ObjectMonitor._owner = virtual_thread_id  (가상 스레드에 묶임)
```

## Related

- [[concepts/virtual-thread|Virtual Thread]]
- [[concepts/thread-pinning|Thread Pinning]]
- [[concepts/jvm-monitor|JVM Monitor]]
- [[summaries/2026-04-15-java-virtual-threads|Java Virtual Threads 공식 문서 요약]]
