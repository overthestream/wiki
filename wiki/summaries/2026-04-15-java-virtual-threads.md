---
title: "Java Virtual Threads — Oracle 공식 문서 요약"
type: summary
tags: [ai, java, concurrency, jvm, virtual-thread]
sources: [raw/articles/java-virtual-threads.md]
created: 2026-04-15
updated: 2026-04-15
---

# Java Virtual Threads — Oracle 공식 문서 요약

## Source Info
- **출처**: https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html
- **저자**: Oracle
- **날짜**: Java 21 (2023-09)

## Summary

Java 21에서 정식 도입된 [[concepts/virtual-thread|Virtual Thread]]는 고처리량 동시성 애플리케이션의 작성, 유지, 디버깅 비용을 줄이기 위한 경량 스레드이다. JEP 444로 제안되었다.

기존 Platform Thread는 OS 스레드와 1:1로 매핑되어 전체 생명주기 동안 OS 스레드를 점유한다. 이 때문에 사용 가능한 스레드 수가 OS 스레드 수에 제한되며, 큰 스택 메모리를 소비한다. 반면 Virtual Thread는 `java.lang.Thread`의 인스턴스이면서도 특정 OS 스레드에 바인딩되지 않는다. 블로킹 I/O 시 Java 런타임이 Virtual Thread를 일시 중단(unmount)하고 해당 OS 스레드(carrier)를 다른 Virtual Thread에 할당한다.

Virtual Thread의 스케줄링은 mount/unmount 모델로 동작한다. Java 런타임이 Virtual Thread를 Platform Thread(carrier)에 할당하면 OS가 해당 carrier를 스케줄링한다. 블로킹 I/O 시 Virtual Thread는 carrier에서 분리되고, carrier는 다른 Virtual Thread를 실행할 수 있다.

단, `synchronized` 블록/메서드 또는 native 메서드 내부에서는 Virtual Thread가 carrier에서 분리될 수 없는 [[concepts/thread-pinning|pinning]] 상태가 된다. 이는 확장성을 저해할 수 있으며, `ReentrantLock`으로 대체하는 것이 권장된다.

디버깅에는 JDK Flight Recorder(JFR) 이벤트, `jcmd` 스레드 덤프, `-Djdk.tracePinnedThreads` 시스템 프로퍼티를 활용할 수 있다.

## Key Takeaways

- Virtual Thread는 **더 빠른 스레드가 아니다** — 속도(낮은 지연)가 아닌 확장성(높은 처리량)을 위한 것
- 가상 메모리가 제한된 RAM에 큰 주소 공간을 매핑하듯, Java 런타임이 대량의 Virtual Thread를 소수의 OS 스레드에 매핑
- I/O 대기가 대부분인 작업에 적합하고, CPU 집약적 작업에는 부적합
- **동기 블로킹 스타일 코드 작성 권장** — `CompletableFuture` 체인 대신 단순한 try-catch 블로킹 코드
- **Thread Pooling 금지** — `newFixedThreadPool` 대신 `newVirtualThreadPerTaskExecutor` 사용
- **동시성 제한은 Semaphore로** — 스레드 풀 크기가 아닌 `Semaphore`로 외부 자원 접근 제한
- **ThreadLocal 캐싱 금지** — Virtual Thread는 풀링되지 않으므로 매번 새 인스턴스 생성됨. 불변 공유 인스턴스 사용
- 10,000개 이상의 Virtual Thread가 아니면 기대 이점이 적을 가능성이 높음

## Quotes & References

> Virtual threads are not faster threads; they do not run code any faster than platform threads. They exist to provide scale (higher throughput), not speed (lower latency).

| 항목 | Platform Thread | Virtual Thread |
|------|-----------------|----------------|
| 구현 | OS 스레드 래퍼 | Java 런타임 구현 |
| 수량 | 제한적 (OS 스레드 수) | 풍부함 (수백만 가능) |
| Pooling | 필요 | 금지 |
| I/O 차단 비용 | 높음 | 낮음 |
| 적합 코드 스타일 | 비동기 콜백 | 동기 블로킹 |

## Related

- [[concepts/virtual-thread|Virtual Thread]]
- [[concepts/thread-pinning|Thread Pinning]]
- [[concepts/jvm-monitor|JVM Monitor]]
- [[summaries/2026-04-15-virtual-thread-qna|Virtual Thread Q&A]]
