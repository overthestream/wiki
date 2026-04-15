---
title: "Monitor"
type: concept
tags: [concurrency, os, synchronization]
sources: [raw/articles/Monitors.md]
created: 2026-04-15
updated: 2026-04-15
---

# Monitor

## Definition

모니터(Monitor)는 **공유 데이터 + 그 데이터에 접근하는 프로시저 + 조건 변수**를 하나의 캡슐로 묶은 동기화 추상화다. Tony Hoare(1974)와 Per Brinch Hansen(1973)이 [[concepts/semaphore|세마포어]]의 한계를 극복하기 위해 제안했다.

핵심 규칙: **모니터 내부에는 한 시점에 최대 하나의 스레드만 실행될 수 있다.** 이것이 컴파일러/런타임에 의해 자동으로 보장되므로, 프로그래머가 락을 직접 관리할 필요가 없다.

## Key Ideas

### 세 가지 구성 요소

| 구성 요소 | 역할 | OOP 대응 |
|-----------|------|----------|
| 공유 데이터 | 모니터가 보호하는 상태. 외부 직접 접근 불가 | `private` 필드 |
| 프로시저 | 공유 데이터를 조작하는 함수. 모니터 진입 = 프로시저 호출 | `synchronized` 메서드 |
| 조건 변수 | 특정 조건 충족까지 스레드를 대기/재개시키는 메커니즘 | `wait()`/`notify()` |

### 조건 변수 (Condition Variables)

상호 배제만으로는 "버퍼가 비었으면 기다려라"같은 조건 동기화를 표현할 수 없다. 조건 변수가 이를 해결한다:

- **`wait()`**: 조건 불충족 시 호출. 스레드는 조건 변수의 대기 큐에 들어가고, 모니터 잠금이 풀림
- **`signal()`**: 조건 변경 후 호출. 대기 큐에서 하나의 스레드를 깨움

### Hoare Monitor (Signal-and-Wait)

`signal()` 호출자가 **즉시 모니터를 양보**한다. 깨어난 스레드는 signal 직후에 실행되므로 조건이 보장된다.

- 조건 체크: **`if`로 충분** — signal 후 아무도 상태를 변경하지 못함
- 내부 큐 3개: Entry Queue, Condition Queue, Signaler Queue
- 장점: 조건 보장이 강력하여 추론이 쉬움
- 단점: signal 시 즉시 컨텍스트 스위치, 구현 복잡

### Mesa Monitor (Signal-and-Continue)

`signal()` 호출자가 **모니터를 계속 사용**한다. 깨어난 스레드는 "실행 가능" 상태가 될 뿐, 실제 실행까지 시간 간격이 있다.

- 조건 체크: **`while` 필수** — signal과 깨어남 사이에 다른 스레드가 상태를 변경할 수 있음
- 내부 큐 2개: Entry Queue, Condition Queue
- 장점: 구현 단순, 불필요한 컨텍스트 스위치 없음, `notifyAll()`(broadcast) 자연스럽게 지원
- 단점: `while`을 빠뜨리면 버그

거의 모든 실용 시스템(Java, C#, POSIX pthread)이 Mesa Monitor를 채택했다.

### `while`이 필수인 이유 (Mesa)

```
소비자 C1: buffer 비었음 → wait()으로 잠듦
생산자 P:  item 추가 → signal() → 계속 실행
소비자 C2: P 퇴장 후 먼저 진입 → buffer에서 item 꺼냄
소비자 C1: 깨어남 → buffer가 다시 비어있음!
           if였으면 빈 buffer에서 remove() → 에러
           while이면 재확인 → 다시 wait()
```

### 세마포어와의 비교

| 항목 | [[concepts/semaphore\|세마포어]] | 모니터 |
|------|------------|--------|
| 추상화 수준 | 저수준 (카운터 + P/V) | 고수준 (캡슐화된 구조체) |
| 상호 배제 | 프로그래머가 직접 관리 | 자동 보장 |
| 조건 동기화 | 카운터로 간접 표현 | 조건 변수로 명시적 표현 |
| 실수 가능성 | 높음 (순서, 누락) | 낮음 (구조가 강제) |

## Examples

### 생산자-소비자 (Mesa Monitor)

```
monitor BoundedBuffer {
    Item[] buffer;
    Condition notFull, notEmpty;

    void produce(Item item) {
        while (buffer.isFull())       // while 필수
            notFull.wait();
        buffer.add(item);
        notEmpty.signal();
    }

    Item consume() {
        while (buffer.isEmpty())      // while 필수
            notEmpty.wait();
        Item item = buffer.remove();
        notFull.signal();
        return item;
    }
}
```

### Java 매핑

| 모니터 구성 요소 | Java 대응 |
|-----------------|-----------|
| 공유 데이터 | 객체의 `private` 필드 |
| 모니터 프로시저 | `synchronized` 메서드 |
| 조건 변수 wait | `Object.wait()` |
| 조건 변수 signal | `Object.notify()` |
| 조건 변수 broadcast | `Object.notifyAll()` |
| 모니터 진입/퇴장 | `monitorenter`/`monitorexit` 바이트코드 |

## Related

- [[concepts/jvm-monitor|JVM Monitor]] — Java에서의 모니터 구현 (Object Header, 락 에스컬레이션, CAS/MESI)
- [[concepts/semaphore|Semaphore]] — 모니터 이전의 저수준 동기화 프리미티브
- [[concepts/virtual-thread|Virtual Thread]] — 모니터(synchronized)가 pinning을 유발하는 맥락
- [[summaries/2026-04-15-monitors|모니터 개념 정리]] — 원본 자료 요약
