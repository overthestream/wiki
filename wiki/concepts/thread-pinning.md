---
title: "Thread Pinning"
type: concept
tags: [java, concurrency, jvm, virtual-thread, pinning]
sources: [raw/articles/java-virtual-threads.md, raw/articles/virtual-thread-QnA.md]
created: 2026-04-15
updated: 2026-04-15
---

# Thread Pinning

## Definition

[[concepts/virtual-thread|Virtual Thread]]가 carrier(Platform Thread)에 고정되어 블로킹 작업 중에도 unmount될 수 없는 상태. 애플리케이션의 정확성에는 문제가 없지만 확장성을 저해한다. Virtual Thread의 핵심 이점(캐리어 재활용)이 무효화되어, 해당 순간에는 Platform Thread와 다를 바 없는 상태가 된다.

## Key Ideas

### 발생 조건

1. **`synchronized` 블록/메서드 내부에서 블로킹 I/O** (JDK 21~23)
2. **Native 메서드 또는 Foreign Function 실행** (JDK 24+에서도 여전히 발생)

### 근본 원인 (JDK 21)

[[concepts/jvm-monitor|JVM Monitor]]의 `ObjectMonitor._owner`가 OS 스레드(캐리어) ID로 기록된다:

```
가상스레드 A ──mount──→ 캐리어 #3
                            │
                  ObjectMonitor._owner = 캐리어 #3
```

이 상태에서 unmount하면 두 가지 문제 발생:
- **소유권 불일치**: 캐리어 #3에 다른 Virtual Thread B가 mount되면, B가 락의 소유자인 것처럼 보임 → 상호 배제 깨짐
- **재진입 실패**: Virtual Thread A가 다른 캐리어 #7에 재mount되면, `_owner`가 캐리어 #3이므로 자신의 락에 재진입 불가

따라서 JVM은 Virtual Thread를 캐리어에 고정(pin)하는 것 외에 선택지가 없었다.

### JDK 24의 해결 (JEP 491)

**Synchronize Virtual Threads without Pinning** — 모니터 소유권을 Virtual Thread ID로 추적하도록 변경:

```
// JDK 21: ObjectMonitor._owner = carrier_thread_id  → pinning 필수
// JDK 24: ObjectMonitor._owner = virtual_thread_id  → pinning 불필요
```

Virtual Thread가 어떤 캐리어에 mount되든 자신의 모니터를 올바르게 인식하므로, `synchronized` 안에서 블로킹되더라도 안전하게 unmount 가능.

**남은 pinning 케이스** (JDK 24+):
- Native 메서드 / Foreign Function & Memory API
- 클래스 로딩 시 (네이티브 코드 경유) — 극히 드문 경우

### 해결 방법 (JDK 21~23)

`synchronized`를 `ReentrantLock`으로 대체:

```java
// 문제: synchronized 내부 블로킹 → pinning
synchronized(lockObj) {
    frequentIO();  // 캐리어 점유
}

// 해결: ReentrantLock 사용 → pinning 없음
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    frequentIO();  // 캐리어에서 unmount 가능
} finally {
    lock.unlock();
}
```

`ReentrantLock`은 `AbstractQueuedSynchronizer`(AQS) 기반으로 `LockSupport.park()`를 사용한다. JVM은 Virtual Thread의 `park()`를 감지하면 캐리어에서 unmount하고 Virtual Thread 상태를 힙에 저장한다.

### 감지 방법

```bash
# JDK Flight Recorder
jfr print --events jdk.VirtualThreadPinned recording.jfr

# 시스템 프로퍼티 (전체 스택 트레이스)
java -Djdk.tracePinnedThreads=full MyApplication

# 시스템 프로퍼티 (간략)
java -Djdk.tracePinnedThreads=short MyApplication
```

JFR의 `jdk.VirtualThreadPinned` 이벤트는 기본 활성화(임계값 20ms).

### JDBC 드라이버 호환성

JDBC 드라이버 내부의 `synchronized`가 pinning의 주요 원인이었다. 락이 사용되는 지점:
- **커넥션 레벨**: `Connection` 상태 변경 동기화
- **Statement/ResultSet 레벨**: 실행 중 간섭 방지
- **소켓 I/O 레벨**: TCP 요청/응답 직렬화 (가장 치명적 — 네트워크 I/O 대기 시간이 김)
- **프로토콜 파싱**: 버퍼 접근 동기화

드라이버별 대응 현황:
| 드라이버 | 지원 버전 | 비고 |
|----------|-----------|------|
| PostgreSQL (pgjdbc) | 42.6.0+ | synchronized → ReentrantLock 교체. 일부 pinning 잔존 |
| MariaDB | 3.3.0+ | pinning 유발 I/O 케이스 회피 |
| Oracle | 21c+ | Virtual Thread 지원 설계 |
| MS SQL | 12.2.0+ | Virtual Thread 친화적 확인 |
| MySQL | 진행 중 | `ConnectionImpl.connectionMutex` 등 복잡한 구조로 지연 |
| MongoDB | 4.11+ | 공식 지원. pinning 방지 내부 업데이트 완료 |

JDK 24+에서는 `synchronized` 개선으로 대부분의 드라이버 pinning 문제가 해소된다.

## Examples

짧은 인메모리 연산이나 빈번하지 않은 작업의 `synchronized`는 문제없다:
```java
// OK — 짧은 인메모리 연산
synchronized(obj) {
    counter++;
}

// 문제 — 오래 걸리는 I/O + 빈번한 호출
synchronized(obj) {
    resultSet = statement.executeQuery(sql);  // 네트워크 I/O 대기
}
```

## Related

- [[concepts/virtual-thread|Virtual Thread]] — Pinning이 영향을 미치는 대상
- [[concepts/jvm-monitor|JVM Monitor]] — Pinning의 근본 원인인 모니터 메커니즘
- [[summaries/2026-04-15-java-virtual-threads|Java Virtual Threads 공식 문서 요약]]
- [[summaries/2026-04-15-virtual-thread-qna|Virtual Thread Q&A]]
