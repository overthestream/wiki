# Virtual Threads - Java 21 Core Libraries

> 출처: https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html

## 개요

Virtual threads는 고처리량 동시성 애플리케이션을 작성, 유지, 디버깅하는 노력을 줄이는 경량 스레드입니다.

**참고**: [JEP 444](https://openjdk.java.net/jeps/444)에서 배경 정보 확인 가능

## 기본 개념

### Platform Thread란?

- OS 스레드 주변의 얇은 래퍼로 구현
- Java 코드를 기본 OS 스레드에서 실행
- Platform thread는 전체 생명주기 동안 OS 스레드를 캡처
- **제한사항**: 사용 가능한 platform thread 수 = OS 스레드 수
- 큰 스택과 OS가 유지하는 다른 리소스 포함
- 모든 유형의 작업에 적합하지만 리소스가 제한적

### Virtual Thread란?

- `java.lang.Thread`의 인스턴스
- 특정 OS 스레드에 바인딩되지 않음
- OS 스레드에서 코드 실행하지만, 블로킹 I/O 작업 호출 시 일시 중단
- 일시 중단된 가상 스레드의 OS 스레드는 다른 가상 스레드에 자유롭게 사용 가능

#### Virtual Memory와의 유사성
가상 메모리가 큰 가상 주소 공간을 제한된 RAM에 매핑하듯이, Java 런타임은 많은 수의 가상 스레드를 적은 수의 OS 스레드에 매핑합니다.

#### 특징
- **얕은 호출 스택**: 일반적으로 단일 HTTP 클라이언트 호출이나 단일 JDBC 쿼리 수행
- **Thread-local 변수 지원**: 신중하게 고려해야 함 (수백만 개의 가상 스레드 지원 가능)
- **적합한 작업**: I/O 작업 완료를 기다리는 데 대부분 시간을 소비하는 작업
- **부적합한 작업**: 오래 실행되는 CPU 집약적 작업

### Virtual Thread를 사용하는 이유

**사용 시나리오**:
- 많은 수의 동시 작업으로 구성된 고처리량 애플리케이션
- 대부분의 시간을 기다리는 작업들
- 서버 애플리케이션 (여러 클라이언트 요청 처리, 블로킹 I/O 작업)

**중요**: Virtual threads는 더 빠른 스레드가 아님 - 코드를 더 빠르게 실행하지 않음
- **목적**: 확장성(높은 처리량) 제공, 속도(낮은 지연시간) 아님

---

## Virtual Thread 생성 및 실행

### 1. Thread.Builder를 사용한 생성

#### 기본 예시
```java
Thread thread = Thread.ofVirtual().start(() -> System.out.println("Hello"));
thread.join();
```

#### Thread.Builder로 스레드 이름 지정
```java
Thread.Builder builder = Thread.ofVirtual().name("MyThread");
Runnable task = () -> {
    System.out.println("Running thread");
};
Thread t = builder.start(task);
System.out.println("Thread t name: " + t.getName());
t.join();
```

#### 여러 Virtual Thread 생성
```java
Thread.Builder builder = Thread.ofVirtual().name("worker-", 0);
Runnable task = () -> {
    System.out.println("Thread ID: " + Thread.currentThread().threadId());
};

// name "worker-0"
Thread t1 = builder.start(task);   
t1.join();
System.out.println(t1.getName() + " terminated");

// name "worker-1"
Thread t2 = builder.start(task);   
t2.join();  
System.out.println(t2.getName() + " terminated");
```

**출력 예시**:
```
Thread ID: 21
worker-0 terminated
Thread ID: 24
worker-1 terminated
```

### 2. Executors를 사용한 생성

```java
try (ExecutorService myExecutor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<?> future = myExecutor.submit(() -> System.out.println("Running thread"));
    future.get();
    System.out.println("Task completed");
    // ...
}
```

**특징**:
- `ExecutorService` 분리로 스레드 관리와 생성을 애플리케이션과 분리
- `submit()` 호출 시 각 작업마다 새로운 가상 스레드 생성 및 시작
- `Future.get()`은 스레드 작업 완료까지 대기

### 3. Echo Server/Client 예시

#### EchoServer
```java
public class EchoServer {
    
    public static void main(String[] args) throws IOException {
         
        if (args.length != 1) {
            System.err.println("Usage: java EchoServer <port>");
            System.exit(1);
        }
         
        int portNumber = Integer.parseInt(args[0]);
        try (
            ServerSocket serverSocket =
                new ServerSocket(Integer.parseInt(args[0]));
        ) {                
            while (true) {
                Socket clientSocket = serverSocket.accept();
                // Accept incoming connections
                // Start a service thread
                Thread.ofVirtual().start(() -> {
                    try (
                        PrintWriter out =
                            new PrintWriter(clientSocket.getOutputStream(), true);
                        BufferedReader in = new BufferedReader(
                            new InputStreamReader(clientSocket.getInputStream()));
                    ) {
                        String inputLine;
                        while ((inputLine = in.readLine()) != null) {
                            System.out.println(inputLine);
                            out.println(inputLine);
                        }
                    
                    } catch (IOException e) { 
                        e.printStackTrace();
                    }
                });
            }
        } catch (IOException e) {
            System.out.println("Exception caught when trying to listen on port "
                + portNumber + " or listening for a connection");
            System.out.println(e.getMessage());
        }
    }
}
```

#### EchoClient
```java
public class EchoClient {
    public static void main(String[] args) throws IOException {
        if (args.length != 2) {
            System.err.println(
                "Usage: java EchoClient <hostname> <port>");
            System.exit(1);
        }
        String hostName = args[0];
        int portNumber = Integer.parseInt(args[1]);
        try (
            Socket echoSocket = new Socket(hostName, portNumber);
            PrintWriter out =
                new PrintWriter(echoSocket.getOutputStream(), true);
            BufferedReader in =
                new BufferedReader(
                    new InputStreamReader(echoSocket.getInputStream()));
        ) {
            BufferedReader stdIn =
                new BufferedReader(
                    new InputStreamReader(System.in));
            String userInput;
            while ((userInput = stdIn.readLine()) != null) {
                out.println(userInput);
                System.out.println("echo: " + in.readLine());
                if (userInput.equals("bye")) break;
            }
        } catch (UnknownHostException e) {
            System.err.println("Don't know about host " + hostName);
            System.exit(1);
        } catch (IOException e) {
            System.err.println("Couldn't get I/O for the connection to " +
                hostName);
            System.exit(1);
        } 
    }
}
```

---

## Virtual Thread 스케줄링

### 스케줄링 메커니즘

1. **OS 스케줄링**: Platform thread는 OS가 스케줄링
2. **Java 런타임 스케줄링**: Virtual thread는 Java 런타임이 스케줄링
3. **마운팅(Mount)**: 런타임이 virtual thread를 platform thread에 할당/마운팅
4. **Carrier**: Platform thread가 virtual thread를 지정하는 용어
5. **언마운팅(Unmount)**: 
   - Virtual thread가 블로킹 I/O 작업 수행 시 일반적으로 발생
   - 언마운팅 후 Carrier는 다른 virtual thread 마운팅 가능

### Pinned Virtual Threads

Virtual thread가 carrier에 고정되어 블로킹 작업 중 언마운팅될 수 없는 상황:

#### 1. Synchronized 블록/메서드
```java
synchronized(lockObj) {
    // 블로킹 I/O 작업
    blockingIO();
}
```

#### 2. Native 메서드 또는 Foreign Function

### Pinning 영향

- **문제**: 애플리케이션 정확성에 문제 없지만 확장성 저해 가능
- **해결책**: 
  - 자주 실행되고 장시간 유지되는 synchronized 블록/메서드 피하기
  - `java.util.concurrent.locks.ReentrantLock` 사용

### ReentrantLock 예시
```java
// 문제 코드
synchronized(lockObj) {
    frequentIO();
}

// 개선된 코드
lock.lock();
try {
    frequentIO();
} finally {
    lock.unlock();
}
```

---

## Virtual Thread 디버깅

### 1. JDK Flight Recorder (JFR) 이벤트

| 이벤트 | 설명 | 기본값 |
|--------|------|--------|
| `jdk.VirtualThreadStart` | Virtual thread 시작 | 비활성 |
| `jdk.VirtualThreadEnd` | Virtual thread 종료 | 비활성 |
| `jdk.VirtualThreadPinned` | Virtual thread pinned (임계값보다 오래) | 활성 (임계값: 20ms) |
| `jdk.VirtualThreadSubmitFailed` | Virtual thread 시작/언파킹 실패 | 활성 |

#### 이벤트 출력
```bash
jfr print --events jdk.VirtualThreadStart,jdk.VirtualThreadEnd,jdk.VirtualThreadPinned,jdk.VirtualThreadSubmitFailed recording.jfr
```

### 2. jcmd 스레드 덤프

```bash
# 텍스트 형식
jcmd <PID> Thread.dump_to_file -format=text <file>

# JSON 형식
jcmd <PID> Thread.dump_to_file -format=json <file>
```

### 3. Pinning 감지

```bash
# 전체 스택 트레이스
java -Djdk.tracePinnedThreads=full MyApplication

# 간략한 정보만
java -Djdk.tracePinnedThreads=short MyApplication
```

---

## Virtual Threads 도입 가이드

### 권장 사항

#### 1. 간단한 동기식 코드 작성 (Thread-Per-Request 스타일)

**비추천 - 비블로킹 비동기 스타일**:
```java
CompletableFuture.supplyAsync(info::getUrl, pool)
   .thenCompose(url -> getBodyAsync(url, HttpResponse.BodyHandlers.ofString()))
   .thenApply(info::findImage)
   .thenCompose(url -> getBodyAsync(url, HttpResponse.BodyHandlers.ofByteArray()))
   .thenApply(info::setImageData)
   .thenAccept(this::process)
   .exceptionally(t -> { t.printStackTrace(); return null; });
```

**추천 - 동기식 블로킹 스타일**:
```java
try {
   String page = getBody(info.getUrl(), HttpResponse.BodyHandlers.ofString());
   String imageUrl = info.findImage(page);
   byte[] data = getBody(imageUrl, HttpResponse.BodyHandlers.ofByteArray());   
   info.setImageData(data);
   process(info);
} catch (Exception ex) {
   t.printStackTrace();
}
```

#### 2. 모든 동시 작업을 Virtual Thread로 표현 (Thread Pooling 금지)

**추천 - Virtual Thread Executor**:
```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
   Future<ResultA> f1 = executor.submit(task1);
   Future<ResultB> f2 = executor.submit(task2);
   // ... use futures
}
```

#### 3. Semaphore로 동시성 제한

```java
Semaphore sem = new Semaphore(10);

Result foo() {
    sem.acquire();
    try {
        return callLimitedService();
    } finally {
        sem.release();
    }
}
```

#### 4. Thread-Local 변수에 비싼 객체 캐싱 금지

**문제**: Virtual thread는 풀링되지 않으므로, ThreadLocal 캐싱 시 매번 새 인스턴스 생성 → 메모리 낭비

**해결**: Immutable 공유 인스턴스 사용 (예: `DateTimeFormatter`)

#### 5. 긴 및 빈번한 Pinning 피하기

Synchronized를 ReentrantLock으로 대체

### 도입 기준

> 10,000개 이상의 virtual thread가 없으면 기대 이점이 적을 가능성 높음

---

## 요약 비교

| 항목 | Platform Thread | Virtual Thread |
|------|-----------------|-----------------|
| 구현 | OS 스레드 래퍼 | Java 런타임 구현 |
| 수량 | 제한적 (OS 스레드 수) | 풍부함 (수백만 가능) |
| 사용 목적 | 공유 리소스 | 개별 작업 |
| Pooling | 필요 | 금지 |
| I/O 차단 | 비용 높음 | 비용 저함 |
| 용도 | 모든 유형 | I/O 대기 작업 |
| 적합 코드 스타일 | 비동기 콜백 | 동기 블로킹 |
