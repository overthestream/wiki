---
title: "Virtual Thread"
type: concept
tags: [java, concurrency, jvm, virtual-thread]
sources: [raw/articles/java-virtual-threads.md, raw/articles/virtual-thread-QnA.md, "raw/articles/SimpleAsyncTaskExecutor (Spring Framework 7.0.6 API).md"]
created: 2026-04-15
updated: 2026-04-15
---

# Virtual Thread

## Definition

Java 21(JEP 444)에서 정식 도입된 경량 스레드. `java.lang.Thread`의 인스턴스이지만 특정 OS 스레드에 바인딩되지 않는다. 블로킹 I/O 시 Java 런타임이 Virtual Thread를 carrier(Platform Thread)에서 unmount하고, 해당 OS 스레드를 다른 Virtual Thread에 재할당한다.

**핵심**: Virtual Thread는 더 빠른 스레드가 아니다. 속도(낮은 지연)가 아닌 **확장성(높은 처리량)**을 제공한다.

## Key Ideas

### Platform Thread vs Virtual Thread

| 항목 | Platform Thread | Virtual Thread |
|------|-----------------|----------------|
| 구현 | OS 스레드의 얇은 래퍼 | Java 런타임 구현 |
| OS 스레드와의 관계 | 1:1 (전체 생명주기 동안 점유) | M:N (다수의 VT가 소수의 OS 스레드에 매핑) |
| 수량 | 제한적 (일반적으로 수천 개, ~1MB/스레드) | 풍부함 (수백만 가능, ~수 KB/스레드) |
| Pooling | 필요 | **금지** |
| I/O 차단 비용 | 높음 (OS 스레드 점유) | 낮음 (carrier에서 unmount) |
| 적합 코드 스타일 | 비동기 콜백 | 동기 블로킹 |
| 적합 작업 | 모든 유형 | I/O 대기가 대부분인 작업 |
| 부적합 작업 | — | CPU 집약적 장시간 작업 |

### 스케줄링: Mount/Unmount 모델

```
Virtual Thread ──mount──→ Carrier (Platform Thread) ──→ OS가 스케줄링
                                    │
                         블로킹 I/O 발생
                                    │
Virtual Thread ←──unmount──┘  Carrier는 다른 VT에 재할당
```

가상 메모리가 제한된 RAM에 큰 가상 주소 공간을 매핑하듯, Java 런타임이 대량의 Virtual Thread를 소수의 OS 스레드에 매핑한다.

### 생성 방법

```java
// 1. Thread.Builder
Thread thread = Thread.ofVirtual().start(() -> System.out.println("Hello"));

// 2. ExecutorService (권장 패턴)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<?> future = executor.submit(() -> doWork());
    future.get();
}
```

### 도입 가이드라인

1. **동기 블로킹 코드 작성** — `CompletableFuture` 체인 대신 단순한 try-catch 블로킹 스타일. Virtual Thread에서는 블로킹이 저비용
2. **Thread Pooling 금지** — Virtual Thread는 공유 자원이 아닌 개별 작업. `newVirtualThreadPerTaskExecutor` 사용
3. **동시성 제한은 Semaphore로** — 스레드 풀 크기가 아닌 `Semaphore`로 외부 자원 접근만 선택적 제한
4. **ThreadLocal 캐싱 금지** — VT는 풀링되지 않아 매번 새 인스턴스 생성. 불변 공유 인스턴스 또는 Scoped Values 사용
5. **빈번하고 긴 synchronized 피하기** — `ReentrantLock`으로 대체 (JDK 24 이전). [[concepts/thread-pinning|Thread Pinning]] 참조

### 적용 기준

> 10,000개 이상의 Virtual Thread가 없으면 기대 이점이 적을 가능성이 높다.

### 실무 적용

- **Spring Boot**: `spring.threads.virtual.enabled=true`로 Tomcat 요청별 Virtual Thread 활성화
  - 내부적으로 `SimpleAsyncTaskExecutor.setVirtualThreads(true)` 사용 — 태스크당 새 Virtual Thread를 생성하는 방식으로, 스레드 재사용 없이 동작하므로 Virtual Thread의 "풀링 금지" 원칙과 자연스럽게 부합
  - `setConcurrencyLimit()`으로 동시 실행 수 제한 가능 (Semaphore와 유사한 효과)
  - 상세: [[summaries/2026-04-15-simple-async-task-executor|SimpleAsyncTaskExecutor API 요약]]
- **JDBC**: PostgreSQL(42.6.0+), MariaDB(3.3.0+), Oracle(21c+), MS SQL(12.2.0+) 지원. MySQL은 대응 지연
- **MongoDB**: Java 드라이버 v4.11+에서 공식 지원

## Examples

### Semaphore를 통한 동시성 제한

```java
Semaphore apiLimit = new Semaphore(10);

try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10_000; i++) {
        executor.submit(() -> {
            apiLimit.acquire();       // 11번째부터 여기서 대기 (unmount)
            try {
                return callExternalAPI();  // 동시 최대 10개만 실행
            } finally {
                apiLimit.release();
            }
        });
    }
}
```

### 동기 블로킹 병렬 호출

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<User> user = executor.submit(() -> restClient.get("/users/" + id));
    Future<Stock> stock = executor.submit(() -> jdbcTemplate.queryForObject(sql, mapper, productId));
    
    // 두 작업이 병렬 실행되고, .get()에서 결과 대기 (unmount됨)
    return new OrderInfo(user.get(), stock.get());
}
```

## Related

- [[concepts/thread-pinning|Thread Pinning]] — Virtual Thread가 carrier에 고정되는 문제
- [[concepts/monitor|Monitor]] — 동기화 추상화 이론 (Hoare vs Mesa)
- [[concepts/jvm-monitor|JVM Monitor]] — synchronized의 내부 동작과 pinning의 근본 원인
- [[summaries/2026-04-15-java-virtual-threads|Java Virtual Threads 공식 문서 요약]]
- [[summaries/2026-04-15-virtual-thread-qna|Virtual Thread Q&A]]
- [[summaries/2026-04-15-simple-async-task-executor|SimpleAsyncTaskExecutor API 요약]] — Spring의 Virtual Thread 통합 구현체
