---
title: "Claude Conversation"
source: "https://claude.ai/share/eb7c40c0-8f48-4bd2-8806-fc3c86ecc795"
author:
published:
created: 2026-04-15
description: "Claude conversation with 10 messages"
tags:
  - "clippings"
---
**You**

[https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html#GUID-DC4306FC-D6C1-4BCC-AECE-48C32C1A8DAA](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html#GUID-DC4306FC-D6C1-4BCC-AECE-48C32C1A8DAA) 이 글 통해 jvm virtual thread 설명해줘

---

**Claude**

문서 내용을 바탕으로 JVM Virtual Thread에 대해 설명드리겠습니다.

## Virtual Thread란?

Java 21에서 정식 도입된 Virtual Thread는 JVM이 직접 관리하는 경량 스레드입니다. 핵심을 이해하려면 먼저 기존 Platform Thread와 비교해야 합니다.

\*\*Platform Thread(기존 스레드)\*\*는 OS 스레드를 1:1로 감싸는 래퍼입니다. 플랫폼 스레드는 자신의 OS 스레드를 전체 생명주기 동안 점유하기 때문에, 사용 가능한 플랫폼 스레드 수는 OS 스레드 수에 의해 제한됩니다. 보통 수천 개가 한계죠.

**Virtual Thread** 는 이와 근본적으로 다릅니다. 가상 스레드도 `java.lang.Thread` 의 인스턴스이지만, 특정 OS 스레드에 묶이지 않습니다. 가상 스레드에서 블로킹 I/O 작업을 호출하면, Java 런타임이 해당 가상 스레드를 일시 중단하고, 그 OS 스레드는 다른 가상 스레드의 작업을 수행할 수 있게 됩니다.

문서에서는 가상 메모리에 비유합니다. OS가 제한된 RAM에 큰 가상 주소 공간을 매핑하듯, Java 런타임은 대량의 가상 스레드를 소수의 OS 스레드에 매핑합니다. 덕분에 수백만 개의 가상 스레드도 가능합니다.

---

## 스케줄링: Mount/Unmount 모델

Java 런타임이 가상 스레드를 스케줄링할 때, 가상 스레드를 플랫폼 스레드에 할당(mount)하고, OS는 그 플랫폼 스레드를 평소처럼 스케줄링합니다. 이 플랫폼 스레드를 \*\*캐리어(carrier)\*\*라고 부릅니다. 코드를 실행한 후, 가상 스레드는 캐리어에서 분리(unmount)될 수 있으며, 이는 보통 블로킹 I/O 시 발생합니다.

즉, 가상 스레드가 I/O를 기다리는 동안 캐리어 스레드는 놀지 않고 다른 가상 스레드를 실행합니다.

---

## Pinning 문제

가상 스레드가 캐리어에서 분리되지 **못하는** 경우가 있는데, 이를 **pinning** 이라 합니다. `synchronized` 블록/메서드 내부에서 블로킹 작업을 수행하거나, native 메서드 또는 외부 함수를 실행할 때 pinning이 발생합니다.

Pinning이 애플리케이션을 잘못 동작하게 만들지는 않지만, 확장성을 저해할 수 있습니다. 해결책은 빈번하고 오래 걸리는 `synchronized` 를 `ReentrantLock` 으로 교체하는 것입니다.

---

## 사용법

생성 방법은 매우 간단합니다:

```java
// 1. 직접 생성
Thread.ofVirtual().start(() -> System.out.println("Hello"));

// 2. ExecutorService 활용 (권장 패턴)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var future1 = executor.submit(() -> fetchURL(url1));
    var future2 = executor.submit(() -> fetchURL(url2));
    response.send(future1.get() + future2.get());
}
```

---

## 핵심 가이드라인

문서에서 강조하는 채택 지침을 정리하면 다음과 같습니다.

**1\. 동기 블로킹 코드를 작성하라.** 가상 스레드는 풍부하므로 블로킹이 저렴하고 권장됩니다. 따라서 간단한 동기 스타일로 코드를 작성하고 블로킹 I/O API를 사용해야 합니다. `CompletableFuture` 체인 같은 비동기 스타일은 가상 스레드의 이점을 살리지 못합니다.

**2\. 가상 스레드를 풀링하지 말라.** 가상 스레드는 풍부하므로 풀링된 공유 자원이 아니라 각 작업(task)을 대표해야 합니다. 가상 스레드의 수는 항상 애플리케이션의 동시 작업 수와 같습니다. `newFixedThreadPool` 대신 `newVirtualThreadPerTaskExecutor` 를 사용하세요.

**3\. 동시성 제한에는 Semaphore를 사용하라.** 외부 서비스 호출 수를 제한할 때, 스레드 풀 크기로 조절하는 관행 대신 `Semaphore` 를 사용해야 합니다. DB 커넥션 풀은 그 자체가 세마포어 역할을 하므로 추가 세마포어가 불필요합니다.

**4\. ThreadLocal 캐싱을 피하라.** 가상 스레드는 풀링되지 않고 관련 없는 작업에 의해 재사용되지 않으므로, 모든 작업마다 새 인스턴스가 생성되어 캐싱의 의도와 정반대의 결과를 초래합니다. 대신 불변 객체를 공유하거나 Scoped Values를 활용하세요.

---

## 핵심 요약

가상 스레드의 본질은 \*\*"속도가 아니라 규모"\*\*입니다. 가상 스레드는 더 빠른 스레드가 아닙니다. 플랫폼 스레드보다 코드를 빠르게 실행하지 않습니다. 가상 스레드가 존재하는 이유는 속도(낮은 지연)가 아니라 규모(높은 처리량)를 제공하기 위해서입니다.

I/O 대기가 많은 서버 애플리케이션에서, 기존에는 스레드 수가 제한되어 동시 요청 처리에 한계가 있었다면, 가상 스레드는 수백만 개의 동시 작업을 간단한 동기 코드로 처리할 수 있게 해줍니다.

---

**You**

1. virtual thread 이전의 스레드는 모두 플랫폼 스레드이며, os 스레드와 1:1인가?
2. 가상 스레드란 cpu core/ os 스레드간의 멀티 플렉싱을 os 스레드 / 애플리케이션 스레드의 멀티 플렉싱으로 확장한 구조라고 할 수 있는가?
3. 기존 플랫폼 스레드 + 블로킹 자바 프로그래밍에서는 io 블로킹 시 스레드가 어떻게 동작하고 있었나?
4. sync 내부에서는 i/o 시에 가상 스레드의 이점을 활용하지 못한다 라고 할 수 잇는 것인가?
5. ReentrantLock은 무엇인가?
6. CompletableFuture과 동기 프로그래밍은 어떻게 다른가?
7. 가상 스레드는 무한히 생성해도 되는가?
8. concurrency limit을 거는 경우 가상 스레드 생성은 어떻게 제한되느낙? - semaphore 예시 알려줘

---

**Claude**

좋은 질문들입니다. 하나씩 답변드리겠습니다.

---

## 1\. Virtual Thread 이전의 스레드는 모두 플랫폼 스레드이며, OS 스레드와 1:1인가?

네, 맞습니다. Java 21 이전에는 `new Thread()` 로 만드는 모든 스레드가 플랫폼 스레드였습니다. 플랫폼 스레드는 OS 스레드를 얇게 감싸는 래퍼로 구현되며, 전체 생명주기 동안 OS 스레드를 점유합니다.

즉, Java 스레드 1개 = OS 스레드(커널 스레드) 1개였고, 이것이 스레드 수의 물리적 한계를 만들었습니다. 일반적으로 수천 개가 실질적 상한이었죠. 스레드 하나당 기본 스택 메모리가 약 1MB이므로, 1만 개만 만들어도 ~10GB 메모리가 필요했습니다.

---

## 2\. 가상 스레드는 OS 스레드/애플리케이션 스레드 간 멀티플렉싱으로 확장한 구조인가?

정확한 이해입니다. 계층을 정리하면 이렇습니다:

```
[기존]
CPU Core ←──M:N──→ OS 스레드 ←──1:1──→ Java 플랫폼 스레드

[Virtual Thread 도입 후]
CPU Core ←──M:N──→ OS 스레드 ←──1:1──→ 캐리어(플랫폼 스레드) ←──M:N──→ Virtual Thread
```

문서에서도 가상 메모리 비유를 사용합니다. OS가 제한된 RAM에 큰 가상 주소 공간을 매핑하여 많은 메모리를 시뮬레이션하듯, Java 런타임은 대량의 가상 스레드를 소수의 OS 스레드에 매핑하여 많은 스레드를 시뮬레이션합니다.

OS가 물리 메모리 위에 가상 메모리를 올린 것처럼, JVM이 OS 스레드 위에 가상 스레드라는 추상 계층을 하나 더 얹은 겁니다. 멀티플렉싱이 한 단계 더 추가된 구조라고 보시면 정확합니다.

---

## 3\. 기존 플랫폼 스레드 + 블로킹 프로그래밍에서 I/O 블로킹 시 스레드는 어떻게 동작했나?

기존 구조에서는 다음과 같이 동작합니다:

```
Thread-1: [요청 처리] → [DB 쿼리 호출] → [OS 스레드째 WAITING] → [응답 받음] → [처리 계속]
                                              ↑
                                    이 동안 OS 스레드가 아무 일도 못 함
                                    CPU는 놀지만 스레드는 점유된 상태
```

플랫폼 스레드가 `socket.read()` 나 JDBC 쿼리 같은 블로킹 I/O를 호출하면, OS 커널이 해당 스레드를 **WAITING/BLOCKED 상태** 로 전환합니다. I/O가 완료될 때까지 그 OS 스레드는 다른 일을 할 수 없습니다. 스레드 자체는 살아있지만 CPU를 쓰지도, 반납하지도 않는 "비싼 대기" 상태가 됩니다.

플랫폼 스레드를 블로킹하는 것은 비용이 큽니다. 왜냐하면 의미 있는 작업을 거의 하지 않으면서도 상대적으로 희소한 자원인 스레드를 붙잡고 있기 때문입니다.

**Virtual Thread의 차이점**: 가상 스레드가 블로킹 I/O를 호출하면, JVM이 해당 가상 스레드를 캐리어에서 **unmount** 합니다. 캐리어(OS 스레드)는 즉시 다른 가상 스레드를 mount해서 실행할 수 있습니다. I/O가 완료되면 가상 스레드는 다시 (같거나 다른) 캐리어에 mount되어 이어서 실행됩니다.

---

## 4\. synchronized 내부에서는 I/O 시 가상 스레드의 이점을 활용하지 못하는가?

네, 그렇습니다. 이것이 **pinning** 문제입니다.

가상 스레드가 `synchronized` 블록이나 메서드 내부에서 블로킹 작업을 수행하면, JVM의 가상 스레드 스케줄러가 귀중한 OS 스레드를 블로킹하게 됩니다. 반면 `synchronized` 외부에서 블로킹하면 그렇지 않습니다.

이유를 좀 더 설명하면, `synchronized` 는 모니터 락을 사용하는데, 이 모니터는 특정 OS 스레드에 귀속됩니다. 따라서 가상 스레드가 `synchronized` 안에 있으면 캐리어에서 분리(unmount)할 수 없고, I/O 대기 중에도 OS 스레드를 그대로 점유합니다. 즉, 이 순간만큼은 플랫폼 스레드와 다를 바 없는 상태가 됩니다.

단, 짧은 인메모리 연산이나 빈번하지 않은 작업을 `synchronized` 로 보호하는 것은 부정적 영향이 없습니다. 문제가 되는 건 **오래 걸리는 I/O + 빈번한 호출** 조합입니다.

---

## 5\. ReentrantLock은 무엇인가?

`ReentrantLock` 은 `java.util.concurrent.locks` 패키지에 있는 명시적 락입니다. `synchronized` 와 같은 상호 배제(mutual exclusion) 기능을 제공하지만, 몇 가지 중요한 차이가 있습니다:

```java
// synchronized - 모니터 락 (OS 스레드에 귀속 → pinning 유발)
synchronized(lockObj) {
    frequentIO();
}

// ReentrantLock - java.util.concurrent 락 (pinning 없음)
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    frequentIO();
} finally {
    lock.unlock();
}
```

핵심 차이는 **구현 수준** 입니다. `synchronized` 는 JVM의 모니터 메커니즘(OS 수준)을 사용하므로 가상 스레드가 캐리어에서 분리될 수 없습니다. 반면 `ReentrantLock` 은 `java.util.concurrent` 의 `AbstractQueuedSynchronizer` (AQS) 기반으로, Java 레벨에서 대기 큐를 관리합니다. 가상 스레드가 이 락에서 블로킹되면 캐리어에서 정상적으로 unmount될 수 있어 pinning이 발생하지 않습니다.

추가로 `ReentrantLock` 은 `tryLock(timeout)`, 공정성(fairness) 설정, `Condition` 객체 등 `synchronized` 에는 없는 기능도 제공합니다.

---

## 6\. CompletableFuture와 동기 프로그래밍은 어떻게 다른가?

문서에서 제시한 두 스타일을 비교하면:

**비동기 스타일 (CompletableFuture)**

```java
CompletableFuture.supplyAsync(info::getUrl, pool)
   .thenCompose(url -> getBodyAsync(url, HttpResponse.BodyHandlers.ofString()))
   .thenApply(info::findImage)
   .thenCompose(url -> getBodyAsync(url, HttpResponse.BodyHandlers.ofByteArray()))
   .thenApply(info::setImageData)
   .thenAccept(this::process)
   .exceptionally(t -> { t.printStackTrace(); return null; });
```

이 코드는 스레드를 블로킹하지 않기 위해 콜백 체인으로 작업을 연결합니다. 스레드 풀의 소수 스레드가 작업 조각들을 번갈아 실행하는 구조입니다. 문제점은 코드가 복잡하고, 디버깅 시 스택 트레이스가 파편화되며, 예외 처리가 어렵다는 것입니다.

**동기 스타일 (블로킹 I/O)**

```java
try {
   String page = getBody(info.getUrl(), HttpResponse.BodyHandlers.ofString());
   String imageUrl = info.findImage(page);
   byte[] data = getBody(imageUrl, HttpResponse.BodyHandlers.ofByteArray());
   info.setImageData(data);
   process(info);
} catch (Exception ex) {
   ex.printStackTrace();
}
```

위에서 아래로 읽히는 자연스러운 코드입니다. 각 `getBody()` 호출에서 스레드가 블로킹되지만, 가상 스레드에서는 이 블로킹이 캐리어 unmount로 처리되므로 비용이 거의 없습니다.

핵심 차이를 정리하면:

|  | CompletableFuture (비동기) | 동기 + Virtual Thread |
| --- | --- | --- |
| 스레드 사용 | 소수 스레드를 콜백으로 공유 | 작업당 가상 스레드 1개 |
| 코드 흐름 | 콜백 체인, 비선형 | 순차적, 위→아래 |
| 디버깅 | 스택 트레이스 파편화 | 정상적인 스택 트레이스 |
| 에러 처리 | `.exceptionally()` 등 별도 패턴 | `try-catch` |

가상 스레드는 간단한 동기 스타일로 코드를 작성하고 블로킹 I/O API를 사용하도록 권장합니다. 비동기 프레임워크의 복잡성 없이 동일한 처리량을 달성할 수 있기 때문입니다.

---

## 7\. 가상 스레드는 무한히 생성해도 되는가?

"무한"은 아닙니다. 가상 스레드도 메모리를 소비하지만, 플랫폼 스레드보다 훨씬 가볍습니다.

가상 스레드는 일반적으로 얕은 호출 스택을 가지며, 단일 HTTP 클라이언트 호출이나 단일 JDBC 쿼리 정도만 수행합니다. 스택이 얕으면 스레드당 메모리가 수 KB 수준이므로 수백만 개도 가능합니다.

하지만 실질적 병목은 다른 곳에 있습니다. 가상 스레드 100만 개가 동시에 DB 커넥션을 요청하면 DB가 죽겠죠. 가상 스레드의 수 자체보다는, 그 스레드들이 접근하는 **하위 자원** (DB 커넥션, 외부 API rate limit, 파일 디스크립터 등)이 실질적 한계가 됩니다. 이것이 바로 다음 질문인 Semaphore가 필요한 이유입니다.

---

## 8\. Semaphore로 동시성을 어떻게 제한하는가?

문서에서 제시하는 패턴입니다. 예를 들어, 외부 API가 동시 10개 요청만 허용한다고 가정하면:

```java
// ❌ 기존 방식: 스레드 풀 크기로 제한 (가상 스레드에서는 안티패턴)
ExecutorService es = Executors.newFixedThreadPool(10);
Result foo() {
    var fut = es.submit(() -> callLimitedService());
    return fut.get();
}

// ✅ 가상 스레드 방식: Semaphore로 제한
Semaphore sem = new Semaphore(10);  // permit 10개

Result foo() {
    sem.acquire();   // permit이 없으면 여기서 블로킹 (가상 스레드이므로 unmount됨)
    try {
        return callLimitedService();
    } finally {
        sem.release(); // permit 반환
    }
}
```

실제 서버 코드로 확장하면 이런 형태가 됩니다:

```java
Semaphore apiLimit = new Semaphore(10);

void handle(Request request, Response response) {
    // 가상 스레드는 제한 없이 생성
    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        // 1000개의 작업을 제출해도 OK
        List<Future<String>> futures = requests.stream()
            .map(req -> executor.submit(() -> {
                apiLimit.acquire();       // 11번째부터는 여기서 대기 (unmount)
                try {
                    return callExternalAPI(req);  // 동시에 최대 10개만 실행
                } finally {
                    apiLimit.release();
                }
            }))
            .toList();

        // 결과 수집
        for (var future : futures) {
            results.add(future.get());
        }
    }
}
```

세마포어에서 가상 스레드를 단순히 블로킹하는 것이 고정 스레드 풀에 작업을 제출하는 것과 크게 달라 보일 수 있지만, 실제로는 그렇지 않습니다. 가상 스레드가 곧 작업이므로 결과 구조는 동등합니다.

차이점은 Semaphore는 **특정 자원의 접근만** 제한하고, 나머지 코드는 자유롭게 실행된다는 것입니다. 스레드 풀은 모든 작업을 풀 크기로 제한하지만, Semaphore는 필요한 지점에만 선택적으로 병목을 걸 수 있습니다.

---

**You**

1. 톰캣서버는 가상 스레드를 지원하는가?
2. JDBC 드라이버 증 가상 스레드를 지원하는 드라이버?
3. 동기 프로그래밍 태스크를 가상 스레드에 제출하는 예시 하나 알려줘
4. JVM 모니터 메커니즘에 대해 좀 더 자세한 설명을 해줘. 하드웨어 레벨까지 가도 괜찮아

---

**Claude**

좋은 질문들입니다. 하나씩 깊이 있게 답변드리겠습니다.

---

## 1\. 톰캣 서버의 가상 스레드 지원

네, 지원합니다. Tomcat은 Virtual Thread Implementation을 제공하며, 이 구현은 Executor에 제출된 각 작업마다 새 가상 스레드를 생성합니다. Tomcat 10.1.x, 11.0, 심지어 9.0에서도 사용 가능합니다.

**Spring Boot에서의 설정** 은 매우 간단합니다. `application.properties` 에 `spring.threads.virtual.enabled=true` 를 추가하면 내장 Tomcat이 각 요청을 가상 스레드에서 처리합니다. [GitHub](https://github.com/linux-china/spring-boot-virtual-thread-demo)

다만 주의점이 있습니다. 블로킹을 경험하는 전통적인 Spring MVC on Tomcat 애플리케이션이 서블릿 비동기 API, 리액티브 프로그래밍 등으로 전환하지 않은 경우, 가상 스레드 기반 executor로 전환하면 일정 수준의 확장성 개선을 볼 수 있습니다. [Spring](https://spring.io/blog/2023/02/27/web-applications-and-project-loom/) 반면, 이미 리액티브로 전환한 애플리케이션은 체감 차이가 크지 않습니다.

---

## 2\. JDBC 드라이버의 가상 스레드 지원 현황

JDBC 드라이버에서 가상 스레드의 핵심 이슈는 **pinning** 입니다. 드라이버 내부에 `synchronized` 블록이 있으면, 그 안에서 I/O할 때 캐리어 스레드가 점유됩니다. 드라이버별 현황을 정리하면:

**PostgreSQL (pgjdbc)**: 42.6.0 버전에서 가상 스레드 친화적으로 개선되었습니다. 42.4.3 이전 버전은 캐리어 스레드를 완전히 pinning했습니다. [OpenJDK](https://mail.openjdk.org/pipermail/loom-dev/2023-June/005817.html) 거의 모든 synchronized 메서드가 ReentrantLock으로 교체되었습니다. [GitHub](https://github.com/quarkusio/quarkus/issues/33337) 다만 최신 버전(42.7.x)에서도 일부 pinning 이슈가 보고되고 있어 완전하지는 않습니다.

**MariaDB**: MariaDB Java connector 3.3.0은 pinning을 유발하는 I/O 케이스를 회피하도록 개선되어 가상 스레드와 호환됩니다. [MariaDB](https://mariadb.com/resources/blog/benchmark-jdbc-connectors-and-java-21-virtual-threads/) MariaDB JDBC 드라이버(3.1.4)는 가상 스레드 친화적인 것으로 확인되었습니다. [OpenJDK](https://mail.openjdk.org/pipermail/loom-dev/2023-June/005817.html)

**Oracle**: Oracle은 JDBC Driver 21c 버전부터 가상 스레드를 이용한 동기식 JDBC 접근을 지원하도록 구축했습니다. [Medium](https://medium.com/oracledevs/introduction-to-oracle-jdbc-21c-driver-support-for-virtual-threads-189b918c56f4)

**MySQL**: MySQL 드라이버에서 pinning을 방지하기 위해 `synchronized` 를 `ReentrantLock` 으로 교체하는 PR이 제출되었지만, `ConnectionImpl` 의 `connectionMutex` 등 복잡한 락 구조 때문에 완전한 적용이 지연되고 있습니다. [MySQL](https://bugs.mysql.com/bug.php?id=109346) MySQL은 상대적으로 가장 대응이 느린 편입니다.

**MS SQL**: MS SQL JDBC 드라이버(12.2.0)는 가상 스레드 친화적인 것으로 확인되었습니다. [OpenJDK](https://mail.openjdk.org/pipermail/loom-dev/2023-June/005817.html)

**Java 24+ 참고**: Java 24 이상에서는 synchronized 개선으로 인해 대부분의 JDBC 드라이버가 더 이상 캐리어 스레드를 pinning하지 않습니다. [Quarkus](https://quarkus.io/guides/virtual-threads) JDK 자체에서 `synchronized` 내부의 블로킹 시에도 가상 스레드를 unmount할 수 있도록 개선이 진행 중이므로, 장기적으로 pinning 문제는 해소될 전망입니다.

---

## 3\. 동기 프로그래밍 태스크를 가상 스레드에 제출하는 예시

여러 외부 API를 동시에 호출하고 결과를 합치는 실용적인 예시입니다:

```java
// 주문 처리: 사용자 정보, 재고, 결제를 동시에 조회
record OrderInfo(User user, Stock stock, Payment payment) {}

OrderInfo processOrder(long userId, long productId) 
        throws InterruptedException, ExecutionException {
    
    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        
        // 3개의 동기 블로킹 태스크를 각각 가상 스레드에 제출
        Future<User> userFuture = executor.submit(() -> {
            // 동기 블로킹 HTTP 호출 — 가상 스레드이므로 캐리어 unmount
            return restClient.get("http://user-service/users/" + userId, User.class);
        });
        
        Future<Stock> stockFuture = executor.submit(() -> {
            // 동기 블로킹 JDBC 호출
            return jdbcTemplate.queryForObject(
                "SELECT * FROM stock WHERE product_id = ?", 
                stockRowMapper, productId);
        });
        
        Future<Payment> paymentFuture = executor.submit(() -> {
            // 동기 블로킹 외부 API 호출
            return paymentClient.getPaymentMethods(userId);
        });
        
        // 3개 작업이 병렬 실행되고, .get()은 결과가 올 때까지 블로킹
        // (이것도 가상 스레드 위에서 실행되면 unmount됨)
        return new OrderInfo(
            userFuture.get(),
            stockFuture.get(),
            paymentFuture.get()
        );
    }
    // try-with-resources가 닫히면서 모든 가상 스레드 종료를 자동 대기
}
```

핵심 포인트는: 각 `submit()` 은 새 가상 스레드를 만들고, 내부의 블로킹 I/O(`restClient.get`, `jdbcTemplate.queryForObject`)에서 캐리어가 해방됩니다. 3개 작업이 물리적으로 동시 실행되면서도 코드는 완전히 동기적이고 선형적입니다.

---

## 4\. JVM 모니터 메커니즘 (하드웨어 레벨까지)

이 부분이 가장 깊은 주제입니다. `synchronized` 가 어떻게 작동하는지 JVM 내부부터 CPU 명령어까지 내려가 보겠습니다.

### 4-1. Java 레벨: synchronized와 모니터

Java의 모든 객체는 \*\*모니터(monitor)\*\*를 가집니다. `synchronized(obj)` 는 obj의 모니터를 획득(enter)하고, 블록이 끝나면 해제(exit)합니다. 바이트코드로는 `monitorenter` 와 `monitorexit` 명령어로 컴파일됩니다.

```java
synchronized(obj) {   // monitorenter
    doWork();
}                      // monitorexit
```

### 4-2. JVM 레벨: Object Header와 락 상태

모니터 정보는 객체의 \*\*헤더(Mark Word)\*\*에 저장됩니다. HotSpot JVM에서 64비트 기준으로 Mark Word는 8바이트이며, 락 상태에 따라 내용이 바뀝니다:

```
┌──────────────────────────────────────────────────────┐
│                    Mark Word (64 bits)                │
├──────────────────────────────────────────────────────┤
│ Unlocked (Biased 아닌 경우):                          │
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

JVM은 경합 정도에 따라 락을 **점진적으로 에스컬레이션** 합니다:

### 4-3. Biased Locking → Thin Lock → Fat Lock 에스컬레이션

**① Biased Locking (편향 락)** — JDK 15에서 deprecated, JDK 18+에서 제거

대부분의 락은 실제로 하나의 스레드만 사용합니다. 이 경우, Mark Word에 해당 스레드 ID를 기록하고, 이후 같은 스레드가 진입할 때는 아무런 동기화 비용 없이 통과합니다. CAS 조차 필요 없습니다.

**② Lightweight Lock (Thin Lock, 경량 락)** — 경합 없는 경우

다른 스레드가 진입을 시도하면 biased가 풀리고 경량 락으로 전환됩니다. 이때 하드웨어 수준의 동기화가 시작됩니다:

```
스레드의 스택 프레임에 Lock Record 생성
     ↓
Lock Record에 Mark Word 복사 (Displaced Mark Word)
     ↓
CAS(Compare-And-Swap)로 Mark Word를 Lock Record 포인터로 교체 시도
     ↓
성공 → 락 획득 (여전히 저비용)
실패 → 다른 스레드가 이미 보유 → 스핀(spin) 시도
```

**③ Heavyweight Lock (Fat Lock, 중량 락)** — 실제 경합 발생 시

스핀으로도 획득 실패하면 JVM이 OS 수준의 동기화 메커니즘을 사용합니다:

```
JVM이 ObjectMonitor 구조체 할당 (C++ 객체)
     ↓
Mark Word가 ObjectMonitor 포인터로 교체 (태그 비트: 10)
     ↓
ObjectMonitor 내부 구조:
  ┌─────────────────────────────────┐
  │ _owner     : 현재 락 보유 스레드  │
  │ _EntryList : 진입 대기 스레드 큐  │
  │ _WaitSet   : wait() 호출 스레드  │
  │ _recursions: 재진입 카운트       │
  │ _count     : 경합 스레드 수       │
  └─────────────────────────────────┘
     ↓
대기 스레드는 OS의 mutex/futex를 통해 BLOCKED 상태로 전환
     ↓
커널 스케줄러가 스레드를 sleep 시킴 (컨텍스트 스위치 발생)
```

### 4-4. 하드웨어 레벨: CAS와 메모리 배리어

경량 락의 핵심인 \*\*CAS(Compare-And-Swap)\*\*는 CPU 명령어로 직접 매핑됩니다:

```
// x86/x64에서의 실제 명령어
lock cmpxchg [memory_address], new_value
```

`lock` prefix는 다음을 보장합니다:

- 해당 캐시 라인을 **배타적으로 소유** (MESI 프로토콜의 Exclusive/Modified 상태)
- **메모리 배리어(memory fence)** 역할 — 이 명령어 전후의 load/store 재배열 방지
- 버스 레벨 또는 캐시 레벨에서 **원자적** 실행 보장

좀 더 구체적으로 CPU 캐시 프로토콜 수준에서 보면:

```
CPU Core 0 (CAS 시도)              CPU Core 1 (락 보유 중)
     │                                    │
     ├─ 캐시 라인 읽기 (Shared)            │
     │                                    │
     ├─ lock cmpxchg 실행                 │
     │   → 캐시 라인 Exclusive 요청        │
     │   → MESI Invalidate 메시지 전송  ──→ │
     │                                    ├─ 캐시 라인 무효화 (Invalid)
     │   ← 다른 코어 ACK               ←──│
     │                                    │
     ├─ 비교: 메모리 값 == expected?        │
     │   YES → 새 값 쓰기 (Modified)       │
     │   NO  → 실패 반환                   │
```

중량 락에서 OS가 개입하면 더 깊어집니다. Linux 기준:

```
JVM ObjectMonitor → pthread_mutex_lock() → futex(FUTEX_WAIT) 시스템콜
     ↓
커널: 
  - 스레드를 실행 큐에서 제거
  - wait queue에 추가
  - 컨텍스트 스위치 (레지스터 저장/복원, TLB flush 가능)
  - 비용: ~수 마이크로초
```

### 4-5. 이것이 가상 스레드와 어떤 관련이 있나?

이제 왜 `synchronized` 가 가상 스레드에서 문제인지 명확해집니다:

**`synchronized` (모니터 기반)**: ObjectMonitor의 `_owner` 는 \*\*OS 스레드(캐리어)\*\*에 묶입니다. 가상 스레드가 `synchronized` 안에서 I/O를 기다릴 때, 모니터가 캐리어를 참조하고 있으므로 JVM이 가상 스레드를 캐리어에서 분리할 수 없습니다. 분리하면 모니터의 소유권 관계가 깨지기 때문입니다.

**`ReentrantLock` (AQS 기반)**: `AbstractQueuedSynchronizer` 의 대기 큐는 `LockSupport.park()` 를 사용합니다. JVM은 가상 스레드의 `park()` 를 감지하면 캐리어에서 unmount하고, 가상 스레드의 상태를 힙에 저장합니다. OS 스레드가 전혀 블로킹되지 않습니다.

```
synchronized + I/O:
  가상스레드 → monitorenter → OS mutex 보유 → I/O 블로킹
  → 캐리어 OS 스레드도 함께 블로킹 (pinning)

ReentrantLock + I/O:
  가상스레드 → lock.lock() (AQS/park) → I/O 블로킹
  → JVM이 가상스레드를 힙에 저장, 캐리어 해방 → 다른 가상스레드 실행
```

이것이 문서에서 빈번하고 오래 걸리는 `synchronized` 를 `ReentrantLock` 으로 교체하라고 권장하는 근본 이유입니다.

---

**You**

1. 가상 스레드를 풀링하는 것과 동시성을 제한하는 것은 어떤 점에서 다른가?
2. jdbc 드라이버들은 락을 어디서 쓰는가? mongoDB 드라이버도 가상 스레드 호환이 되는가?
3. jdk 24에서는 피닝이 어떻게 개선되었나?
4. java monitor = CS의 critical section인가? 오브젝트마다 각각 뮤텍스를 가지고 있는 것인가? - 락 대기시 는 플랫폼 스레드는 wating 상대로 스피닝 락에 빠지나?
5. 가상스레드 피닝 문제는 요약하자면 object의 뮤텍스 정보를 OS 스레드가 가지고 있으므로 독립적으로 실행할 수 없다 인가?

---

**Claude**

좋은 질문들입니다. 하나씩 답변드리겠습니다.

---

## 1\. 가상 스레드를 풀링하는 것 vs 동시성을 제한하는 것

이 둘은 **제한 범위** 가 근본적으로 다릅니다.

**스레드 풀링 (풀 크기 = 10)**

```java
ExecutorService pool = Executors.newFixedThreadPool(10);
pool.submit(() -> {
    callServiceA();   // ← 이것도 10개로 제한됨
    callServiceB();   // ← 이것도 10개로 제한됨
    compute();         // ← 이것도 10개로 제한됨
});
```

풀의 10개 스레드가 **모든 작업의 모든 단계** 를 제한합니다. ServiceA가 느려서 스레드 10개를 다 잡으면, ServiceB 호출이나 compute 같은 전혀 무관한 작업도 전부 멈춥니다. 풀은 "자원 관리" 도구이지 "동시성 제어" 도구가 아닌데, 부수 효과로 동시성이 제한되는 것입니다.

**Semaphore (permit = 10)**

```java
Semaphore semA = new Semaphore(10);

void handle() {
    // 이 구간은 가상 스레드 100만 개가 동시에 실행 가능
    var data = prepareData();
    
    semA.acquire();
    try {
        callServiceA();  // ← 여기만 동시 10개로 제한
    } finally {
        semA.release();
    }
    
    // 이 구간도 자유롭게 실행
    callServiceB();     // ← 제한 없음
    compute();           // ← 제한 없음
}
```

Semaphore는 **특정 자원의 접근 지점만** 선택적으로 제한합니다. 나머지 코드는 가상 스레드의 이점(무제한 동시성)을 온전히 누립니다.

정리하면:

|  | 스레드 풀 | Semaphore |
| --- | --- | --- |
| 제한 대상 | 전체 작업의 모든 단계 | 특정 자원 접근 지점만 |
| 가상 스레드 의미 | 가상 스레드를 소수로 제한 → 존재 의미 상실 | 가상 스레드는 무제한, 병목만 선택 제어 |
| 비유 | 식당 입구에서 10명만 입장 허용 | 식당은 자유 입장, 화장실만 10칸 |

---

## 2\. JDBC 드라이버는 락을 어디서 쓰는가? MongoDB 드라이버의 가상 스레드 호환성

### JDBC 드라이버에서 락이 사용되는 곳

JDBC 드라이버 내부에는 `synchronized` 가 광범위하게 사용됩니다. 주요 지점은:

**커넥션 레벨**: `Connection` 객체의 상태(autoCommit, transaction isolation 등)를 변경할 때 동기화가 필요합니다. 예를 들어 MySQL 드라이버의 `ConnectionImpl` 은 `synchronized(connectionMutex)` 를 사용하며, 이 프록시의 구현체들도 synchronized 메서드를 가지고 있습니다. [MySQL](https://bugs.mysql.com/bug.php?id=109346) 커넥션 하나에 여러 스레드가 접근하는 것을 방지합니다.

**Statement/ResultSet 레벨**: PostgreSQL 드라이버의 `PgStatement` 은 `synchronized(this)` 와 `synchronized(connection)` 을 사용합니다. [GitHub](https://github.com/pgjdbc/pgjdbc/issues/1951) Statement 실행 중 다른 스레드의 간섭을 막습니다.

**소켓 I/O 레벨**: DB 서버와의 TCP 소켓 통신에서 요청/응답을 직렬화하기 위해 락을 걸합니다. 이것이 가장 치명적인 pinning 지점인데, 네트워크 I/O라 대기 시간이 길기 때문입니다.

**프로토콜 파싱**: 서버 응답을 읽고 파싱하는 과정에서도 버퍼 접근 동기화가 필요합니다.

### MongoDB 드라이버의 가상 스레드 호환성

네, MongoDB Java 드라이버는 가상 스레드를 공식 지원합니다. 드라이버 내부가 불필요한 가상 스레드 pinning을 방지하고, 구조화된 동시성에서 취소에 사용되는 스레드의 인터럽트 상태를 보존하도록 업데이트되었습니다. [MongoDB](https://www.mongodb.com/docs/drivers/java/sync/v5.0/reference/whats-new/) 이 지원은 v4.11(2023년 10월)에 추가되었고, 현재 v5.x에서도 유지되고 있습니다.

---

## 3\. JDK 24에서 pinning이 어떻게 개선되었나

JDK 24는 **JEP 491: Synchronize Virtual Threads without Pinning** 을 통해 근본적인 변화를 도입했습니다.

### 핵심 변경 사항

`synchronized` 키워드는 객체와 연관된 모니터에 의존합니다. Java 24 이전에는 가상 스레드가 `synchronized` 를 통해 모니터를 획득하면, JVM이 캐리어 플랫폼 스레드를 모니터 보유자로 인식했습니다. 가상 스레드는 플랫폼 스레드 간에 이동할 수 있으므로, 모니터를 보유하는 동안 이동을 방지하기 위해 pinning이 필요했습니다. [Mike my bytes](https://mikemybytes.com/2025/04/09/java24-thread-pinning-revisited/)

JVM의 `synchronized` 키워드 구현을 변경하여, 가상 스레드가 캐리어와 독립적으로 모니터를 획득, 보유, 해제할 수 있게 했습니다. mount/unmount 연산이 `synchronized` 메서드나 블록 내부에서도 가상 스레드의 unmount와 재mount를 허용하는 데 필요한 부기(bookkeeping)를 수행합니다. [OpenJDK](https://openjdk.org/jeps/491)

구체적으로 말하면, JDK 24에서 모니터 메커니즘이 가상 스레드를 인식하고, 가상 스레드의 ID를 사용하여 누가 어떤 모니터를 보유하는지 추적합니다. 따라서 `synchronized` 도 `Object::wait` 도 더 이상 pinning이 필요하지 않습니다. [nipafx](https://nipafx.dev/inside-java-newscast-80/)

### 벤치마크 결과

JDK 21에서 5,000개 가상 스레드로 실행 시 총 31.791초가 걸렸던 것이, JDK 24에서는 동일 테스트에서 극적으로 개선되었습니다. [GitHub](https://github.com/danvega/pinning) (캐리어 스레드 1개로 제한한 극단적 테스트 조건에서)

### 남은 pinning 케이스

모든 pinning 시나리오가 사라진 것은 아닙니다. native 메서드나 Foreign Function & Memory API를 사용하여 네이티브 코드를 호출하는 가상 스레드는 여전히 pinning될 수 있습니다. 하지만 이는 대부분의 프로덕션 Java 앱에서 해당하지 않습니다. [Mike my bytes](https://mikemybytes.com/2025/04/09/java24-thread-pinning-revisited/)

또한 클래스 로딩이 네이티브 코드를 거치므로, 가상 스레드가 클래스를 로드할 때 pinning됩니다. 이는 클래스 이니셜라이저에 블로킹 연산이 포함된 경우 가장 관련이 있지만, 극히 드문 경우입니다. [nipafx](https://nipafx.dev/inside-java-newscast-80/)

### 실무적 의미

이전에는 빈번하고 오래 지속되는 pinning 문제를 `synchronized` 에서 `ReentrantLock` 으로 마이그레이션하여 해결할 것을 권장했습니다. `synchronized` 키워드가 더 이상 가상 스레드를 pinning하지 않으면, 그러한 마이그레이션은 더 이상 필요하지 않습니다. [OpenJDK](https://openjdk.org/jeps/491) 즉, JDBC 드라이버들이 `synchronized` 를 `ReentrantLock` 으로 바꾸느라 고생한 것이 JDK 24부터는 불필요해진 셈입니다.

---

## 4\. Java Monitor = Critical Section인가? 오브젝트마다 뮤텍스를 가지고 있는가?

### Monitor ≈ Critical Section + Condition Variable

Java의 모니터는 CS(Computer Science)의 **모니터 개념** (Hoare/Mesa 모니터)을 구현한 것으로, 단순한 critical section(상호 배제 영역)보다 넓은 개념입니다:

```
Java Monitor = Mutual Exclusion (critical section) + Condition Variable (wait/notify)
```

`synchronized` 블록은 모니터의 상호 배제 부분(critical section 진입/퇴장)을, `wait()` / `notify()` / `notifyAll()` 은 조건 변수 부분을 담당합니다.

### 오브젝트마다 뮤텍스를 가지고 있는가?

**논리적으로는 그렇습니다.** 모든 Java 객체는 잠재적으로 모니터(뮤텍스)로 사용될 수 있습니다. 하지만 **물리적으로는 lazy allocation** 입니다.

평소에는 객체 헤더의 Mark Word에 락 상태 비트만 존재하고, 실제 `ObjectMonitor` C++ 구조체(뮤텍스 포함)는 경합이 발생하여 heavyweight lock으로 에스컬레이션될 때만 할당됩니다:

```
모든 객체: Mark Word (8바이트) ← 항상 존재
     │
     ├─ 경합 없음: Lock Record (스택에 할당, 저비용)
     │
     └─ 경합 발생 → ObjectMonitor 할당 (힙에 할당)
                     ├─ _owner (뮤텍스 보유 스레드)
                     ├─ _EntryList (진입 대기 큐)
                     ├─ _WaitSet (wait() 대기 큐)
                     └─ OS mutex/futex (실제 동기화 프리미티브)
```

### 락 대기 시 플랫폼 스레드는 spinning인가 waiting인가?

**둘 다입니다.** JVM은 적응적 전략을 사용합니다:

```
락 획득 시도 (CAS 실패)
     │
     ├─ 1단계: Spinning (적응적 스핀)
     │   - 짧은 시간 busy-wait (CPU 사이클 소비)
     │   - 락 보유자가 곧 해제할 것으로 예상될 때 유리
     │   - JVM이 과거 이력을 보고 스핀 횟수를 동적 조정
     │   - 성공 → 락 획득, 컨텍스트 스위치 회피
     │
     └─ 2단계: OS-level Blocking (스핀 실패 시)
         - pthread_mutex_lock() → futex(FUTEX_WAIT) 시스템콜
         - 스레드가 커널의 wait queue로 이동
         - 상태: BLOCKED (Java) / SLEEPING (OS)
         - CPU를 소비하지 않음 (진짜 잠듦)
         - 락 해제 시 커널이 깨움 → 컨텍스트 스위치 비용 발생
```

적응적 스핀(Adaptive Spinning)이 핵심인데, 이전에 같은 락에서 스핀이 성공했으면 더 오래 스핀하고, 실패 이력이 많으면 바로 OS 블로킹으로 넘어갑니다. `-XX:+UseSpinning` (구 버전) 또는 JVM 내부 휴리스틱으로 제어됩니다.

정리하면, spinning은 "곧 풀릴 것 같은 짧은 경합"에 대한 최적화이고, 오래 걸리면 OS에 위임하여 CPU를 낭비하지 않습니다.

---

## 5\. 가상 스레드 pinning 문제의 본질

네, 그 요약이 핵심을 정확히 짚고 있습니다. 좀 더 정밀하게 표현하면:

### JDK 21에서의 문제

```
ObjectMonitor._owner = 캐리어(플랫폼/OS) 스레드의 ID
```

모니터의 소유권이 **OS 스레드** 에 기록됩니다. 가상 스레드 A가 `synchronized` 블록에 진입하면:

```
가상스레드 A  ──mount──→  캐리어 스레드 #3
                              │
                    ObjectMonitor._owner = 캐리어 #3
```

이 상태에서 가상 스레드 A를 unmount하면 두 가지 문제가 발생합니다:

**문제 1: 소유권 불일치.** 캐리어 #3에 다른 가상 스레드 B가 mount되면, B가 모니터의 "소유자"인 캐리어 #3 위에서 실행됩니다. 모니터 입장에서 B가 락을 보유한 것처럼 보이게 되어 상호 배제가 깨집니다.

**문제 2: 재진입 실패.** 가상 스레드 A가 나중에 캐리어 #7에 재mount되어 같은 synchronized 블록에 재진입하려 하면, `_owner` 가 캐리어 #3이므로 자기 자신의 락인데도 진입이 거부됩니다.

따라서 JVM은 **가상 스레드를 캐리어에 고정(pin)하는 것 외에 선택지가 없었습니다.**

### JDK 24에서의 해결

JDK 24에서 모니터 메커니즘이 가상 스레드를 인식하고, 가상 스레드의 ID를 사용하여 누가 어떤 모니터를 보유하는지 추적합니다. [nipafx](https://nipafx.dev/inside-java-newscast-80/)

```
// JDK 21
ObjectMonitor._owner = carrier_thread_id   // OS 스레드에 묶임

// JDK 24
ObjectMonitor._owner = virtual_thread_id   // 가상 스레드에 묶임
```

모니터 소유권이 가상 스레드 ID로 추적되므로, 가상 스레드가 어떤 캐리어에 mount되든 상관없이 자신의 모니터를 올바르게 인식합니다. 따라서 `synchronized` 안에서 블로킹되더라도 안전하게 unmount할 수 있게 된 것입니다.

한 줄로 요약하면: **JDK 21의 pinning은 "모니터가 OS 스레드를 추적하는데, 가상 스레드는 OS 스레드 간에 이동하는 존재"라는 근본적 불일치에서 비롯된 것이고, JDK 24는 모니터가 가상 스레드를 직접 추적하도록 바꿔서 해결한 것입니다.**

---

**You**

1. Semaphore를 제공하는 java virtual thread api가 있을 것 같은데 그런 예시 한 번 찾아줘
2. MongoDB virtutal thread 드라이버 사용 예시 코드 찾아줘.

---

**Claude**

두 가지 모두 찾아봤습니다.

---

## 1\. Virtual Thread + Semaphore 예시

Semaphore는 별도의 "virtual thread용 API"가 아니라, `java.util.concurrent.Semaphore` 를 그대로 사용합니다. 가상 스레드를 사용할 때 외부 서비스가 너무 많은 호출로 압도되는 것을 방지하는 방법은 `java.util.concurrent.Semaphore` 를 사용하는 것입니다. Semaphore는 풀처럼 작동하지만, 커넥션이나 스레드 같은 희소 자원을 풀링하는 대신 permit(단순 카운터)을 풀링합니다. [Inside.java](https://inside.java/2024/02/04/sip094/)

핵심은 가상 스레드가 `semaphore.acquire()` 에서 블로킹되면 캐리어에서 unmount되므로 OS 스레드를 소비하지 않는다는 것입니다. 이것이 플랫폼 스레드의 블로킹과 근본적으로 다른 점입니다.

Oracle의 Inside Java에서 제시하는 공식 패턴은 이렇습니다:

```java
// 외부 서비스의 rate limit에 맞춰 permit 수 설정
private static final Semaphore RATE_LIMITER = new Semaphore(10);

String callExternalService(String request) throws InterruptedException {
    RATE_LIMITER.acquire();  // permit 없으면 가상 스레드가 여기서 블로킹 (unmount됨)
    try {
        return httpClient.send(request);  // 동시 최대 10개만 실행
    } finally {
        RATE_LIMITER.release();  // 반드시 finally에서 release (permit 누수 방지)
    }
}
```

`release()` 는 반드시 `finally` 블록 안에 배치하여 permit 누수를 방지해야 합니다. [Inside.java](https://inside.java/2024/02/04/sip094/)

이를 실제 서버 애플리케이션에 통합한 전체 예시는 다음과 같습니다:

```java
import java.util.concurrent.*;
import java.net.http.*;
import java.net.URI;

public class VirtualThreadSemaphoreExample {

    // 외부 결제 API: 동시 5개 제한
    private static final Semaphore PAYMENT_LIMITER = new Semaphore(5);
    // 외부 알림 API: 동시 20개 제한
    private static final Semaphore NOTIFICATION_LIMITER = new Semaphore(20);

    private static final HttpClient httpClient = HttpClient.newHttpClient();

    public static void main(String[] args) throws Exception {

        // 가상 스레드는 제한 없이 생성 — Semaphore가 외부 자원만 선택적으로 제한
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {

            // 10,000건의 주문을 동시에 처리
            List<Future<String>> futures = new ArrayList<>();
            for (int i = 0; i < 10_000; i++) {
                final int orderId = i;
                futures.add(executor.submit(() -> processOrder(orderId)));
            }

            // 결과 수집
            for (var future : futures) {
                System.out.println(future.get());
            }
        }
    }

    static String processOrder(int orderId) throws Exception {
        // 1단계: 주문 검증 — 제한 없음, 가상 스레드 10,000개가 동시에 가능
        validateOrder(orderId);

        // 2단계: 결제 — Semaphore로 동시 5개 제한
        PAYMENT_LIMITER.acquire();
        String paymentResult;
        try {
            paymentResult = callPaymentAPI(orderId);
        } finally {
            PAYMENT_LIMITER.release();
        }

        // 3단계: 알림 — Semaphore로 동시 20개 제한
        NOTIFICATION_LIMITER.acquire();
        try {
            sendNotification(orderId);
        } finally {
            NOTIFICATION_LIMITER.release();
        }

        return "Order " + orderId + ": " + paymentResult;
    }

    static void validateOrder(int orderId) {
        // 인메모리 검증 — 제한 불필요
    }

    static String callPaymentAPI(int orderId) throws Exception {
        // 동기 블로킹 HTTP 호출 — 가상 스레드이므로 캐리어 unmount
        var request = HttpRequest.newBuilder()
            .uri(URI.create("https://payment-api.example.com/charge/" + orderId))
            .build();
        var response = httpClient.send(request, HttpResponse.BodyHandlers.ofString());
        return response.body();
    }

    static void sendNotification(int orderId) throws Exception {
        // 동기 블로킹 HTTP 호출
        var request = HttpRequest.newBuilder()
            .uri(URI.create("https://notification-api.example.com/send/" + orderId))
            .build();
        httpClient.send(request, HttpResponse.BodyHandlers.ofString());
    }
}
```

주의사항으로, 가상 스레드를 풀링하면 안 됩니다. 가상 스레드는 희소 자원이 아닙니다. 이미 커넥션 풀을 사용하여 서비스 연결을 관리하고 있다면, Semaphore 사용을 피해야 합니다. [Inside.java](https://inside.java/2024/02/04/sip094/) DB 커넥션 풀(HikariCP 등)이 이미 있는 경우 추가 Semaphore는 불필요합니다. 커넥션 풀 자체가 세마포어 역할을 하기 때문입니다.

---

## 2\. MongoDB + Virtual Thread 사용 예시

MongoDB는 두 가지 접근 방식으로 가상 스레드를 활용할 수 있습니다.

### 방식 A: Spring Boot + Spring Data MongoDB (가장 간단)

`spring.threads.virtual.enabled=true` 를 `application.properties` 에 추가하면 됩니다. [GitHub](https://github.com/mongodb-developer/java-spring-boot-mongodb-starter)

```
# application.properties
spring.data.mongodb.uri=mongodb://localhost:27017/mydb
spring.threads.virtual.enabled=true
```

```java
// Entity
@Document("products")
public record Product(
    @Id String id,
    String name,
    double price,
    int stock
) {}

// Repository
public interface ProductRepository extends MongoRepository<Product, String> {
    List<Product> findByPriceLessThan(double price);
}

// Controller — 각 요청이 자동으로 가상 스레드에서 실행됨
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductRepository productRepository;

    public ProductController(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @GetMapping("/{id}")
    public Product findProduct(@PathVariable String id) {
        // 동기 블로킹 호출이지만, 가상 스레드 위에서 실행되므로
        // 캐리어가 unmount되어 다른 요청을 처리할 수 있음
        return productRepository.findById(id)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
    }

    @GetMapping("/cheap")
    public List<Product> findCheapProducts(@RequestParam double maxPrice) {
        return productRepository.findByPriceLessThan(maxPrice);
    }

    @PostMapping
    public Product createProduct(@RequestBody Product product) {
        return productRepository.save(product);
    }
}
```

이 방식에서는 **코드 변경 없이** 설정 한 줄로 가상 스레드가 활성화됩니다. Tomcat이 들어오는 HTTP 요청마다 가상 스레드를 생성하고, 그 안에서 `productRepository.findById()` 가 동기적으로 실행됩니다.

### 방식 B: MongoDB Java Driver 직접 사용 + Virtual Thread Executor

Spring 없이 드라이버를 직접 사용하면서 가상 스레드로 대량 동시 작업을 수행하는 예시입니다:

```java
import com.mongodb.client.*;
import com.mongodb.client.model.*;
import org.bson.Document;
import java.util.concurrent.*;
import java.util.List;
import java.util.ArrayList;

public class MongoVirtualThreadExample {

    public static void main(String[] args) throws Exception {
        
        // MongoClient는 스레드 안전하며 내부 커넥션 풀 보유
        try (MongoClient mongoClient = MongoClients.create("mongodb://localhost:27017")) {
            
            MongoDatabase db = mongoClient.getDatabase("mydb");
            MongoCollection<Document> collection = db.getCollection("sensors");

            // 가상 스레드로 10,000개 센서 데이터를 동시에 삽입
            try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
                
                List<Future<Void>> futures = new ArrayList<>();
                
                for (int i = 0; i < 10_000; i++) {
                    final int sensorId = i;
                    futures.add(executor.submit(() -> {
                        // 각 가상 스레드에서 동기 블로킹 MongoDB 호출
                        Document doc = new Document()
                            .append("sensorId", sensorId)
                            .append("temperature", Math.random() * 100)
                            .append("timestamp", System.currentTimeMillis());
                        
                        collection.insertOne(doc);  // 블로킹 I/O → 캐리어 unmount
                        return null;
                    }));
                }

                // 모든 작업 완료 대기
                for (var future : futures) {
                    future.get();
                }
            }

            // 가상 스레드로 여러 쿼리를 동시에 실행하고 결과 합치기
            try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
                
                var hotSensors = executor.submit(() ->
                    collection.find(Filters.gt("temperature", 90))
                        .into(new ArrayList<>())
                );
                
                var coldSensors = executor.submit(() ->
                    collection.find(Filters.lt("temperature", 10))
                        .into(new ArrayList<>())
                );
                
                var totalCount = executor.submit(() ->
                    collection.countDocuments()
                );

                // 3개 쿼리가 병렬 실행됨
                System.out.println("Hot sensors: " + hotSensors.get().size());
                System.out.println("Cold sensors: " + coldSensors.get().size());
                System.out.println("Total documents: " + totalCount.get());
            }
        }
    }
}
```

### 방식 C: Quarkus + Eclipse JNoSQL (MongoDB 공식 튜토리얼)

Quarkus에서는 `@VirtualThreads` 어노테이션이 붙은 `ExecutorService` 를 주입받아, 가상 스레드를 통해 MongoDB 레코드를 비동기적으로 삽입할 수 있습니다. [DEV Community](https://dev.to/mongodb/java-virtual-threads-in-action-optimizing-mongodb-operation-4l5k)

```java
@ApplicationScoped
public class CameraService {

    @Inject
    DocumentTemplate template;

    @Inject
    @VirtualThreads
    ExecutorService vThreads;

    public void generateCameras(int count) {
        for (int i = 0; i < count; i++) {
            final int id = i;
            vThreads.submit(() -> {
                Camera camera = new Camera(
                    "camera-" + id,
                    "City-" + (id % 100),
                    Math.random() * 180 - 90,   // latitude
                    Math.random() * 360 - 180    // longitude
                );
                template.insert(camera);  // 동기 블로킹 — 가상 스레드에서 실행
            });
        }
    }
}
```

### 핵심 포인트

세 방식 모두 공통적으로, MongoDB 드라이버 v4.11+ 또는 v5.x를 사용해야 합니다. 드라이버 내부가 불필요한 가상 스레드 pinning을 방지하도록 업데이트되었습니다. [MongoDB](https://www.mongodb.com/docs/drivers/java/sync/v5.0/reference/whats-new/) 드라이버 버전만 맞으면 기존 동기 코드를 그대로 가상 스레드 위에서 실행하는 것으로 충분합니다. 별도의 "virtual thread 전용 API"를 배울 필요가 없다는 것이 가상 스레드의 설계 철학입니다.