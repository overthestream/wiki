---
title: "SimpleAsyncTaskExecutor — Spring Framework API"
type: summary
tags: [java, spring, concurrency, virtual-thread, api-doc]
sources: ["raw/articles/SimpleAsyncTaskExecutor (Spring Framework 7.0.6 API).md"]
created: 2026-04-15
updated: 2026-04-15
---

# SimpleAsyncTaskExecutor — Spring Framework API

## Source Info
- **출처**: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/SimpleAsyncTaskExecutor.html
- **저자**: Juergen Hoeller
- **날짜**: Spring Framework 7.0.6 (Spring 2.0부터 존재)

## Summary

`SimpleAsyncTaskExecutor`는 Spring Framework의 `TaskExecutor` 구현체로, 제출된 각 태스크마다 새로운 스레드를 생성하여 실행한다. `CustomizableThreadCreator`를 상속하며 `AsyncTaskExecutor`, `Serializable`, `AutoCloseable`을 구현한다.

### Virtual Thread 지원 (Spring 6.1+, JDK 21+)
`setVirtualThreads(true)` 메서드로 [[concepts/virtual-thread|Virtual Thread]] 모드를 활성화할 수 있다. 활성화하면 Platform Thread 대신 Virtual Thread를 생성한다. 이는 `spring.threads.virtual.enabled=true` 설정의 내부 구현체에 해당한다.

### 동시성 제한 (Concurrency Throttle)
`setConcurrencyLimit(int)` 메서드로 최대 병렬 실행 수를 제한할 수 있다. 스레드 풀의 최대 풀 크기와 동등한 효과이지만, 스레드 풀과 달리 **제한에 도달하면 제출자를 블로킹**한다(큐 기반이 아님). `setRejectTasksWhenLimitReached(true)`로 블로킹 대신 `TaskRejectedException`을 던지게 할 수도 있다.

상수:
- `UNBOUNDED_CONCURRENCY` (-1): 동시성 제한 없음 (기본값)
- `NO_CONCURRENCY` (0): 동시 실행 불허

### Graceful Shutdown
`setTaskTerminationTimeout(long)` 메서드로 `close()` 시 활성 스레드의 종료 대기 시간을 설정할 수 있다. 단, 태스크 추적 오버헤드가 발생하므로 대량의 짧은 태스크에는 부적합하다. `setCancelRemainingTasksOnClose(true)`로 `close()` 시 즉시 인터럽트도 가능하다 (Spring 6.2.11+).

### 주요 제한사항
- **스레드를 재사용하지 않는다** — 대량의 짧은 태스크에는 `ThreadPoolTaskExecutor`가 더 적합. 다만 JDK 21에서는 Virtual Thread 활성화가 대안
- **컨텍스트 수준 생명주기 관리에 참여하지 않는다** — 넘겨진 스레드의 태스크를 중앙에서 중지/재시작 불가. 엄격한 생명주기 관리가 필요하면 `ThreadPoolTaskExecutor` 사용

### RetryTask 통합
Spring 7.0에서 `RetryTask`와 결합하여 실패 시 재시도 정책에 따라 태스크를 재실행할 수 있다.

## Key Takeaways

- `SimpleAsyncTaskExecutor`는 태스크당 새 스레드를 생성하는 가장 단순한 `TaskExecutor` 구현체
- JDK 21+에서 `setVirtualThreads(true)`로 Virtual Thread 모드 활성화 가능 — 스레드 재사용 없이도 대량 동시 태스크 처리 가능
- `setConcurrencyLimit()`은 스레드 풀의 풀 크기와 유사하지만, 큐 없이 제출자를 블로킹하는 방식으로 동작
- 스레드 풀이 아니므로 Virtual Thread의 "풀링 금지" 원칙과 자연스럽게 부합
- Graceful shutdown은 `taskTerminationTimeout` 설정 시에만 동작하며, 태스크 추적 오버헤드 수반
- `TaskDecorator`로 태스크 실행 전후에 컨텍스트 설정/모니터링 가능

## Quotes & References

> This implementation does not reuse threads! Consider a thread-pooling TaskExecutor implementation instead, in particular for executing a large number of short-lived tasks. Alternatively, on JDK 21, consider setting `setVirtualThreads(boolean)` to `true`.

주요 메서드:

| 메서드 | 설명 | Since |
|--------|------|-------|
| `setVirtualThreads(boolean)` | Virtual Thread 모드 전환 | 6.1 |
| `setConcurrencyLimit(int)` | 최대 병렬 실행 수 제한 | 2.0 |
| `setTaskTerminationTimeout(long)` | Graceful shutdown 타임아웃 설정 | 6.1 |
| `setCancelRemainingTasksOnClose(boolean)` | close() 시 즉시 인터럽트 여부 | 6.2.11 |
| `setRejectTasksWhenLimitReached(boolean)` | 제한 도달 시 거부 vs 블로킹 | 6.2.6 |
| `setTaskDecorator(TaskDecorator)` | 태스크 실행 전후 데코레이터 | 4.3 |

## Related

- [[concepts/virtual-thread|Virtual Thread]] — SimpleAsyncTaskExecutor의 Virtual Thread 모드가 활용하는 핵심 기술
- [[summaries/2026-04-15-java-virtual-threads|Java Virtual Threads 공식 문서 요약]]
- [[summaries/2026-04-15-virtual-thread-qna|Virtual Thread Q&A]]
