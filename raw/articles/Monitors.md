# 모니터(Monitor) 개념 완전 정리

## 1. 모니터가 해결하려는 문제

동시성 프로그래밍의 두 가지 핵심 난제가 있다.

**① 상호 배제 (Mutual Exclusion)** 여러 스레드가 공유 데이터를 동시에 수정하면 데이터가 깨진다. 한 번에 하나의 스레드만 접근해야 한다.

**② 조건 동기화 (Condition Synchronization)** "버퍼가 비었으면 기다려라", "버퍼가 차면 기다려라"처럼, 특정 조건이 만족될 때까지 스레드를 멈추고 조건이 충족되면 깨워야 한다.

### 세마포어의 한계

모니터 이전에는 이 두 문제를 **세마포어**로 해결했다. 하지만 세마포어는 저수준 프리미티브여서 프로그래머가 직접 `acquire()`/`release()` 쌍을 정확히 배치해야 한다. 순서를 잘못 쓰거나 하나를 빠뜨리면 데드락이나 레이스 컨디션이 발생하고, 코드가 복잡해질수록 실수가 기하급수적으로 늘어난다.

```
// 세마포어로 생산자-소비자 구현 — 실수하기 쉬움
P(mutex);         // 상호 배제 획득
P(emptySlots);    // 빈 슬롯 대기  ← 이 순서가 바뀌면 데드락!
buffer.add(item);
V(mutex);
V(fullSlots);
```

Tony Hoare(1974)와 Per Brinch Hansen(1973)은 **상호 배제와 조건 동기화를 언어 수준에서 자동으로 보장하는 구조체**를 제안했다. 그것이 **모니터**다.

---

## 2. 모니터란 무엇인가

모니터는 **공유 데이터 + 그 데이터에 접근하는 프로시저 + 조건 변수**를 하나의 캡슐로 묶은 동기화 추상화다. 객체지향의 클래스와 비슷하되, **동시성 보장이 내장**되어 있다.

```
┌─────────────────────────────────────────────┐
│                  Monitor                     │
│                                             │
│   ┌─────────────────────────────────────┐   │
│   │         공유 데이터 (state)          │   │
│   │   buffer[], count, head, tail ...   │   │
│   └─────────────────────────────────────┘   │
│                                             │
│   ┌──────────────┐  ┌──────────────┐       │
│   │ produce(item)│  │  consume()   │       │  ← 프로시저 (진입점)
│   └──────────────┘  └──────────────┘       │
│                                             │
│   조건 변수: notFull, notEmpty              │  ← wait() / signal()
│                                             │
├─────────────────────────────────────────────┤
│   진입 큐 (Entry Queue)                     │
│   [ Thread-C ][ Thread-D ][ Thread-E ]      │
└─────────────────────────────────────────────┘

규칙: 모니터 안에는 한 번에 하나의 스레드만 존재할 수 있다.
```

> **핵심 규칙**: 모니터 내부에는 한 시점에 최대 하나의 스레드만 실행될 수 있다. 이것이 컴파일러/런타임에 의해 자동으로 보장된다. 프로그래머가 락을 직접 걸거나 풀 필요가 없다.

---

## 3. 모니터의 세 가지 구성 요소

### 3.1. 공유 데이터 (Shared State)

모니터가 보호하는 데이터. 외부에서 직접 접근할 수 없고, 반드시 모니터의 프로시저를 통해서만 접근한다. 객체지향의 `private` 필드와 같은 개념이다.

### 3.2. 프로시저 (Monitor Procedures)

공유 데이터를 조작하는 함수들. **모니터에 진입한다 = 이 프로시저 중 하나를 호출한다**는 의미이며, 한 스레드가 프로시저를 실행 중이면 다른 스레드는 진입 큐에서 대기한다.

### 3.3. 조건 변수 (Condition Variables)

상호 배제만으로는 부족하다. "버퍼가 비었으면 소비자가 기다려야 한다"는 논리를 표현해야 하는데, 이때 조건 변수를 사용한다.

조건 변수에는 두 가지 연산이 있다:

- **`wait()`**: "지금 조건이 안 맞으니 나는 잠들겠다. 모니터를 양보할 테니 다른 스레드가 들어와라." 호출한 스레드는 조건 변수의 대기 큐에 들어가고, 모니터의 잠금이 풀린다.
- **`signal()`** (또는 `notify()`): "조건이 바뀌었으니 기다리는 스레드를 깨워라." 대기 큐에서 하나의 스레드를 깨운다.

---

## 4. 모니터 동작 흐름 예시 (생산자-소비자)

```
monitor BoundedBuffer {
    Item[] buffer = new Item[N];
    int count = 0;
    Condition notFull, notEmpty;

    void produce(Item item) {
        while (count == N)
            notFull.wait();       // 버퍼가 가득 차면 기다림
        buffer.add(item);
        count++;
        notEmpty.signal();        // 소비자를 깨움
    }

    Item consume() {
        while (count == 0)
            notEmpty.wait();      // 버퍼가 비었으면 기다림
        Item item = buffer.remove();
        count--;
        notFull.signal();         // 생산자를 깨움
        return item;
    }
}
```

### 실행 과정

```
[시점 1] 소비자 C가 consume() 호출
         → 모니터 진입 (현재 모니터 안: C)
         → count == 0이므로 notEmpty.wait()
         → C는 notEmpty 대기 큐로 이동, 모니터 잠금 해제
         → 모니터 안: 비어있음

[시점 2] 생산자 P가 produce(item) 호출
         → 모니터 진입 (현재 모니터 안: P)
         → count != N이므로 대기 안 함
         → buffer에 item 추가, count = 1
         → notEmpty.signal() → C를 깨움
         → P가 모니터 퇴장

[시점 3] 소비자 C가 깨어남
         → 모니터 재진입
         → while 조건 재확인: count != 0 → 통과
         → buffer에서 item 꺼냄, count = 0
         → notFull.signal()
         → C가 모니터 퇴장
```

프로그래머가 한 것은 **"언제 기다릴지(wait)"와 "언제 깨울지(signal)"만 명시**한 것이다. 상호 배제(한 번에 하나만 들어감)는 모니터가 자동으로 보장한다.

---

## 5. 세마포어와의 비교

같은 생산자-소비자를 세마포어로 구현하면:

```
Semaphore mutex = new Semaphore(1);      // 상호 배제
Semaphore emptySlots = new Semaphore(N); // 빈 슬롯 수
Semaphore fullSlots = new Semaphore(0);  // 찬 슬롯 수

void produce(Item item) {
    emptySlots.acquire();    // 빈 슬롯 대기
    mutex.acquire();         // 상호 배제 획득 ← 순서 중요! 바꾸면 데드락
    buffer.add(item);
    mutex.release();         // 상호 배제 해제 ← 빠뜨리면 데드락
    fullSlots.release();     // 찬 슬롯 알림
}

Item consume() {
    fullSlots.acquire();     // 찬 슬롯 대기
    mutex.acquire();         // 상호 배제 획득
    Item item = buffer.remove();
    mutex.release();         // 상호 배제 해제
    emptySlots.release();    // 빈 슬롯 알림
    return item;
}
```

|항목|세마포어|모니터|
|---|---|---|
|추상화 수준|저수준 (카운터 + P/V)|고수준 (캡슐화된 구조체)|
|상호 배제|프로그래머가 직접 관리|자동 보장|
|조건 동기화|세마포어 카운터로 간접 표현|조건 변수로 명시적 표현|
|실수 가능성|높음 (순서, 누락)|낮음 (구조가 강제)|
|비유|신호등을 직접 수동 제어|자동 교통 관제 시스템|

---

## 6. 모니터의 핵심 질문: signal() 후 누가 실행되는가?

스레드 A가 조건 변수에 `signal()`을 보내서 대기 중인 스레드 B를 깨울 때, **누가 모니터를 계속 사용하는가?**

이 질문에 대한 답이 **Hoare 모니터**와 **Mesa 모니터**를 나눈다.

---

## 7. Hoare Monitor (Signal-and-Wait)

Hoare의 원래 제안(1974)에서는 `signal()`을 호출한 스레드가 **즉시 모니터를 양보**한다.

```
시간 흐름 →

스레드 A (signal 호출자)          스레드 B (wait에서 대기 중)
────────────────────          ─────────────────────────
모니터 안에서 실행 중
condition.signal() 호출
  ├─ A는 즉시 중단 (양보)  ──→   B가 즉시 모니터를 획득하고 실행 재개
  │   A는 대기 큐로 이동          B는 조건이 참임을 보장받음
  │                              B 실행 완료 또는 다시 wait()
  │                         ←──  모니터 반환
  A가 다시 모니터 획득
  나머지 코드 실행
```

### 특성

- **`signal()` 후 조건이 보장됨**: B가 깨어나는 시점에 A가 signal을 보낸 직후이므로, 그 사이에 아무도 공유 상태를 변경하지 못한다. B는 자신이 기다린 조건이 여전히 참임을 확신할 수 있다.
- **`if`로 조건을 한 번만 체크하면 충분**하다.

```java
// Hoare 스타일 — if로 충분
if (buffer.isEmpty()) {
    notEmpty.wait();
}
// 여기서 buffer는 반드시 비어있지 않음이 보장됨
item = buffer.remove();
```

### 내부 큐 구조

Hoare 모니터는 세 가지 큐가 필요하다:

```
┌─────────────────────────────────────┐
│           Hoare Monitor             │
│                                     │
│   [1] Entry Queue                   │  모니터에 들어오려는 스레드
│   [2] Condition Queue               │  wait()한 스레드
│   [3] Signaler Queue                │  signal()한 후 양보한 스레드
│                                     │
│   우선순위: Signaler > Condition     │
│             > Entry                 │
└─────────────────────────────────────┘
```

### 장단점

- **장점**: 조건 보장이 강력하여 추론이 쉽다.
- **단점**: signal 시 즉시 컨텍스트 스위치 발생, 구현 복잡, 큐 3개 관리.

---

## 8. Mesa Monitor (Signal-and-Continue)

Mesa 모니터(Xerox PARC, 1980)에서는 `signal()`을 호출한 스레드가 **모니터를 계속 사용**한다. 깨어난 스레드는 단지 "실행 가능" 상태가 될 뿐, 즉시 실행되지 않는다.

```
시간 흐름 →

스레드 A (signal 호출자)          스레드 B (wait에서 대기 중)
────────────────────          ─────────────────────────
모니터 안에서 실행 중
condition.signal() 호출
  ├─ B를 "실행 가능"으로 표시       B는 아직 실행 안 됨
  ├─ A는 계속 실행                 (진입 큐에서 대기)
  ├─ 더 많은 코드 실행 가능
  ├─ 모니터 퇴장              ──→  B가 모니터 획득 시도
                                   조건이 여전히 참인지 다시 확인 필요!
                                   (A나 다른 스레드가 상태를 바꿨을 수 있음)
```

### 특성

- **`signal()` 후 조건이 보장되지 않음**: B가 실제로 실행을 재개하기까지 시간 간격이 있고, 그 사이에 A나 다른 스레드가 공유 상태를 변경할 수 있다.
- **반드시 `while` 루프로 조건을 재확인**해야 한다.

```java
// Mesa 스타일 — while 필수
while (buffer.isEmpty()) {    // if가 아니라 while!
    notEmpty.wait();
}
// 깨어났지만 다른 스레드가 먼저 buffer를 비웠을 수 있음
// while이 다시 체크해줌
item = buffer.remove();
```

### 내부 큐 구조

Mesa 모니터는 두 가지 큐면 충분하다:

```
┌─────────────────────────────────────┐
│           Mesa Monitor              │
│                                     │
│   [1] Entry Queue                   │  모니터에 들어오려는 스레드
│   [2] Condition Queue               │  wait()한 스레드                 
│                                     │
│   (Signaler Queue 불필요            │
│    — signal 호출자가 계속 실행)       │
└─────────────────────────────────────┘
```

### while이 필수인 이유 — 구체적 시나리오

```
소비자 C1: buffer 비었음 → wait()으로 잠듦
생산자 P:  item 추가 → signal() → 계속 실행
소비자 C2: (P가 아직 모니터 안에 있는 동안 진입 큐에서 대기)
           P가 모니터 퇴장 → C2가 먼저 진입 → buffer에서 item 꺼냄
소비자 C1: 이제야 깨어남 → buffer가 다시 비어있음!
           if였으면 빈 buffer에서 remove() 시도 → 에러
           while이면 다시 체크 → 비었으므로 다시 wait()
```

### 장단점

- **장점**: 구현 단순, 불필요한 컨텍스트 스위치 없음, 현실적으로 더 효율적.
- **단점**: 프로그래머가 반드시 `while`을 써야 함 (안 쓰면 버그).

---

## 9. 생산자-소비자 코드 비교

```java
// ═══════════════ Hoare Monitor ═══════════════
monitor BoundedBuffer {
    Item[] buffer;
    Condition notFull, notEmpty;

    void produce(Item item) {
        if (buffer.isFull())          // ← if로 충분
            notFull.wait();
        buffer.add(item);
        notEmpty.signal();            // signal 후 즉시 양보 → 소비자 실행
        // 소비자가 끝난 후에야 여기로 돌아옴
    }

    Item consume() {
        if (buffer.isEmpty())         // ← if로 충분
            notEmpty.wait();
        Item item = buffer.remove();
        notFull.signal();
        return item;
    }
}

// ═══════════════ Mesa Monitor ═══════════════
monitor BoundedBuffer {
    Item[] buffer;
    Condition notFull, notEmpty;

    void produce(Item item) {
        while (buffer.isFull())       // ← while 필수!
            notFull.wait();
        buffer.add(item);
        notEmpty.signal();            // signal 후 계속 실행
        // 바로 다음 줄로 진행
    }

    Item consume() {
        while (buffer.isEmpty())      // ← while 필수!
            notEmpty.wait();
        Item item = buffer.remove();
        notFull.signal();
        return item;
    }
}
```

---

## 10. 전체 비교표

|항목|Hoare Monitor|Mesa Monitor|
|---|---|---|
|signal 후 동작|호출자가 즉시 양보 (Signal-and-Wait)|호출자가 계속 실행 (Signal-and-Continue)|
|깨어난 스레드|즉시 실행, 조건 보장됨|나중에 실행, 조건 재확인 필요|
|조건 체크|`if` 가능|`while` 필수|
|컨텍스트 스위치|추가 발생 (signal 시 양보)|최소화|
|내부 큐|3개 (Entry, Condition, Signaler)|2개 (Entry, Condition)|
|구현 복잡도|높음|낮음|
|broadcast (notifyAll)|의미론적으로 어색함|자연스럽게 지원|
|채택 사례|이론/교육용 위주|**Java**, C# Monitor, POSIX pthread, 거의 모든 실용 시스템|

---

## 11. Java는 Mesa Monitor

Java의 `synchronized` + `wait()`/`notify()`는 **Mesa 모니터**다. 따라서 Java에서는 항상 다음 패턴을 따라야 한다:

```java
synchronized(lock) {
    while (!condition) {    // 절대 if 쓰지 말 것
        lock.wait();
    }
    // 조건 충족 후 작업 수행
}
```

### Java가 Mesa를 선택한 이유

1. Hoare 모니터는 signal 시 즉시 컨텍스트 스위치가 필요한데, JVM에서 이를 효율적으로 구현하기 어렵고 불필요한 오버헤드를 만든다.
2. `notify()`가 특정 스레드를 지정할 수 없고 아무나 깨우기 때문에, `while` 재확인이 어차피 필요하다.
3. `notifyAll()`(broadcast)이 자연스럽게 동작한다 — 깨어난 스레드들이 하나씩 모니터를 획득하고 `while`로 조건을 재확인하므로, 조건을 만족하는 스레드만 진행하고 나머지는 다시 잠든다. Hoare에서는 이런 패턴이 불가능하다.

### Java 객체 = 모니터 매핑

|모니터 구성 요소|Java 대응|
|---|---|
|공유 데이터|객체의 `private` 필드|
|모니터 프로시저|`synchronized` 메서드|
|조건 변수 wait|`Object.wait()`|
|조건 변수 signal|`Object.notify()`|
|조건 변수 broadcast|`Object.notifyAll()`|
|모니터 진입|`synchronized` 블록 진입 / `monitorenter` 바이트코드|
|모니터 퇴장|`synchronized` 블록 퇴장 / `monitorexit` 바이트코드|

> **한 줄 요약**: 모니터는 "공유 데이터와 그 데이터에 접근하는 코드를 하나로 묶고, 한 번에 하나의 스레드만 그 안에서 실행되도록 보장하는 동기화 추상화"이며, Java `synchronized`의 이론적 기반이다. Java는 Mesa 모니터 방식을 채택하여 `while` 루프 안의 `wait()`이 표준 패턴이다.