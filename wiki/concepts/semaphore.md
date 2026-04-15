---
title: "Semaphore"
type: concept
tags: [concurrency, os, synchronization]
sources: [raw/articles/Monitors.md]
created: 2026-04-15
updated: 2026-04-15
---

# Semaphore

## Definition

세마포어(Semaphore)는 정수 카운터와 두 가지 원자적 연산(`P`/`V`, 또는 `acquire`/`release`)으로 구성된 저수준 동기화 프리미티브다. Edsger Dijkstra가 고안했으며, [[concepts/monitor|모니터]]가 등장하기 전까지 상호 배제와 조건 동기화를 해결하는 주된 방법이었다.

## Key Ideas

### 동작 원리

- **`P(S)` / `acquire()`**: 카운터 > 0이면 1 감소하고 통과. 0이면 양수가 될 때까지 스레드 블로킹
- **`V(S)` / `release()`**: 카운터를 1 증가. 대기 중인 스레드가 있으면 하나를 깨움

카운터 초기값에 따라 용도가 달라진다:
- **이진 세마포어** (초기값 1): 상호 배제 (뮤텍스와 유사)
- **카운팅 세마포어** (초기값 N): 자원 풀 관리, 동시 접근 수 제한

### 한계

세마포어는 저수준 프리미티브이므로 프로그래머가 `acquire()`/`release()` 쌍을 정확히 배치해야 한다:

- **순서 실수 → 데드락**: `P(mutex)` 후 `P(emptySlots)` 순서가 바뀌면 데드락
- **누락 → 레이스 컨디션**: `release()`를 빠뜨리면 영구 블로킹
- **코드 복잡도 증가**: 세마포어 수가 늘어날수록 정확한 순서 관리가 기하급수적으로 어려워짐

이러한 한계가 [[concepts/monitor|모니터]]의 탄생 동기가 되었다.

### 현대적 활용

모니터가 상호 배제의 주된 수단이 된 이후에도, 카운팅 세마포어는 **동시 접근 수 제한** 용도로 여전히 유용하다. 특히 [[concepts/virtual-thread|Virtual Thread]] 환경에서 스레드 풀 크기 대신 `java.util.concurrent.Semaphore`로 외부 자원 접근을 제한하는 패턴이 권장된다.

## Examples

### 생산자-소비자 (세마포어)

```java
Semaphore mutex = new Semaphore(1);      // 상호 배제
Semaphore emptySlots = new Semaphore(N); // 빈 슬롯 수
Semaphore fullSlots = new Semaphore(0);  // 찬 슬롯 수

void produce(Item item) {
    emptySlots.acquire();    // 빈 슬롯 대기
    mutex.acquire();         // 상호 배제 획득 ← 순서 중요!
    buffer.add(item);
    mutex.release();
    fullSlots.release();
}

Item consume() {
    fullSlots.acquire();     // 찬 슬롯 대기
    mutex.acquire();
    Item item = buffer.remove();
    mutex.release();
    emptySlots.release();
    return item;
}
```

### Virtual Thread에서의 동시성 제한

```java
Semaphore apiLimit = new Semaphore(10);

try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10_000; i++) {
        executor.submit(() -> {
            apiLimit.acquire();
            try {
                return callExternalAPI();  // 동시 최대 10개
            } finally {
                apiLimit.release();
            }
        });
    }
}
```

## Related

- [[concepts/monitor|Monitor]] — 세마포어의 한계를 극복한 고수준 동기화 추상화
- [[concepts/virtual-thread|Virtual Thread]] — Semaphore 패턴으로 동시성 제한
- [[summaries/2026-04-15-monitors|모니터 개념 정리]] — 세마포어와 모니터 비교 원본 자료
