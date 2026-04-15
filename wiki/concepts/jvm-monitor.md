---
title: "JVM Monitor"
type: concept
tags: [java, concurrency, jvm, synchronization, hardware]
sources: [raw/articles/virtual-thread-QnA.md]
created: 2026-04-15
updated: 2026-04-15
---

# JVM Monitor

## Definition

Java의 모니터는 CS의 **Hoare/Mesa 모니터** 개념을 구현한 것으로, 상호 배제(Mutual Exclusion)와 조건 변수(Condition Variable)를 결합한 동기화 메커니즘이다.

```
Java Monitor = Mutual Exclusion (synchronized) + Condition Variable (wait/notify)
```

모든 Java 객체는 잠재적으로 모니터로 사용될 수 있다. `synchronized`는 바이트코드의 `monitorenter`/`monitorexit`로 컴파일된다.

## Key Ideas

### Object Header와 Mark Word

모니터 정보는 객체 헤더의 **Mark Word**(64비트)에 저장된다. 락 상태에 따라 Mark Word의 내용이 변한다:

```
┌──────────────────────────────────────────────────────┐
│                    Mark Word (64 bits)                │
├──────────────────────────────────────────────────────┤
│ Unlocked:                                            │
│  [unused:25 | hashcode:31 | unused:1 | age:4 | 0|01] │
│                                                      │
│ Biased Locking:                                      │
│  [thread_id:54 | epoch:2 | unused:1 | age:4 | 1|01] │
│                                                      │
│ Lightweight Lock:                                    │
│  [ptr_to_lock_record:62                         |00] │
│                                                      │
│ Heavyweight Lock:                                    │
│  [ptr_to_ObjectMonitor:62                       |10] │
│                                                      │
│ GC Marked:                                           │
│  [forwarding_address:62                         |11] │
└──────────────────────────────────────────────────────┘
 (마지막 2비트가 락 상태 태그)
```

논리적으로 모든 객체가 뮤텍스를 가지지만, 물리적으로는 **lazy allocation**이다. 실제 `ObjectMonitor` C++ 구조체는 경합 시에만 할당된다.

### 락 에스컬레이션

JVM은 경합 정도에 따라 락을 점진적으로 에스컬레이션한다:

#### 1단계: Biased Locking (편향 락)
- JDK 15에서 deprecated, JDK 18+에서 제거
- 대부분의 락이 하나의 스레드만 사용하는 경우를 최적화
- Mark Word에 스레드 ID만 기록, CAS 없이 통과

#### 2단계: Lightweight Lock (경량 락)
- 다른 스레드가 진입 시도 시 전환
- 스레드의 스택 프레임에 Lock Record 생성
- **CAS(Compare-And-Swap)**로 Mark Word를 Lock Record 포인터로 교체 시도
- 실패 시 스핀(spin) → 재시도

#### 3단계: Heavyweight Lock (중량 락)
- 스핀 실패 시 OS 수준 동기화로 전환
- `ObjectMonitor` 구조체 할당:
  ```
  ObjectMonitor
  ├─ _owner      : 현재 락 보유 스레드
  ├─ _EntryList  : 진입 대기 스레드 큐
  ├─ _WaitSet    : wait() 호출 스레드
  ├─ _recursions : 재진입 카운트
  └─ _count      : 경합 스레드 수
  ```
- 대기 스레드는 OS의 mutex/futex를 통해 BLOCKED 상태로 전환

### 하드웨어 레벨: CAS와 메모리 배리어

경량 락의 CAS는 CPU 명령어로 직접 매핑:

```asm
; x86/x64
lock cmpxchg [memory_address], new_value
```

`lock` prefix의 보장:
- 해당 캐시 라인을 **배타적으로 소유** (MESI 프로토콜의 Exclusive/Modified 상태)
- **메모리 배리어(memory fence)** 역할 — load/store 재배열 방지
- 버스/캐시 레벨에서 **원자적** 실행 보장

MESI 프로토콜 수준의 동작:
```
CPU Core 0 (CAS 시도)              CPU Core 1 (락 보유 중)
     │                                    │
     ├─ 캐시 라인 읽기 (Shared)            │
     ├─ lock cmpxchg 실행                 │
     │   → Exclusive 요청                  │
     │   → MESI Invalidate 전송 ────────→  │
     │                                    ├─ 캐시 라인 무효화 (Invalid)
     │   ← ACK ←───────────────────────── │
     ├─ 비교: 메모리 값 == expected?        │
     │   YES → 새 값 쓰기 (Modified)       │
     │   NO  → 실패 반환                   │
```

### 중량 락의 OS 개입 (Linux)

```
JVM ObjectMonitor → pthread_mutex_lock() → futex(FUTEX_WAIT) 시스템콜
     ↓
커널:
  - 스레드를 실행 큐에서 제거
  - wait queue에 추가
  - 컨텍스트 스위치 (레지스터 저장/복원, TLB flush 가능)
  - 비용: ~수 마이크로초
```

### 적응적 스핀 (Adaptive Spinning)

락 대기 시 JVM은 적응적 전략을 사용:

1. **Spinning**: 짧은 busy-wait. 이전에 같은 락에서 스핀 성공 이력이 있으면 더 오래 스핀. 성공 시 컨텍스트 스위치 회피
2. **OS Blocking**: 스핀 실패 시 `futex(FUTEX_WAIT)` 시스템콜로 진짜 sleep. CPU를 소비하지 않음

### Virtual Thread와의 관계

`synchronized` 내부 블로킹이 [[concepts/thread-pinning|pinning]]을 유발하는 이유:
- ObjectMonitor._owner가 OS 스레드(캐리어)에 묶임
- Virtual Thread를 unmount하면 모니터 소유권 관계가 깨짐
- JDK 24에서 모니터가 Virtual Thread ID를 직접 추적하도록 변경하여 해결

`ReentrantLock`(AQS 기반)은 `LockSupport.park()`를 사용하며, JVM이 Virtual Thread의 `park()`를 감지하면 캐리어에서 unmount하고 상태를 힙에 저장한다. OS 스레드가 전혀 블로킹되지 않는다.

```
synchronized + I/O:
  VT → monitorenter → OS mutex 보유 → I/O 블로킹 → 캐리어도 블로킹 (pinning)

ReentrantLock + I/O:
  VT → lock.lock() (AQS/park) → I/O 블로킹 → JVM이 VT를 힙에 저장, 캐리어 해방
```

## Examples

```java
// synchronized — 바이트코드: monitorenter/monitorexit
synchronized(obj) {   // monitorenter: Mark Word 변경 시도
    doWork();
}                     // monitorexit: Mark Word 복원

// ReentrantLock — AQS 기반, pinning 없음
ReentrantLock lock = new ReentrantLock();
lock.lock();          // AQS CAS → 실패 시 LockSupport.park()
try {
    doWork();
} finally {
    lock.unlock();    // AQS 상태 변경 → LockSupport.unpark()
}
```

## Related

- [[concepts/virtual-thread|Virtual Thread]] — 모니터 메커니즘이 영향을 미치는 대상
- [[concepts/thread-pinning|Thread Pinning]] — 모니터가 원인인 pinning 문제
- [[summaries/2026-04-15-virtual-thread-qna|Virtual Thread Q&A]] — 하드웨어 레벨까지의 상세 분석 원본
