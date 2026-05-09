# Tomcat + Virtual Threads

> 내장 서블릿 컨테이너 + Java 21의 가상 스레드.

## Tomcat (내장)

Spring Boot의 `spring-boot-starter-web` 가 기본으로 내장하는 서블릿 컨테이너. 외부 WAS 없이 `java -jar app.jar` 로 실행.

특징:
- HTTP/1.1 + HTTP/2 + WebSocket
- NIO 기반 커넥터 (다수 커넥션을 적은 스레드로 처리)
- `server.tomcat.threads.max` 로 최대 스레드 수 조정

대안: Jetty, Undertow, Netty(WebFlux). `spring-boot-starter-undertow` 로 교체 가능. 이 프로젝트는 Tomcat 기본.

## Virtual Threads (Java 21, JEP 444)

### 기존(Platform Threads)의 한계
- OS 스레드 1:1 매핑 → 하나당 ~1MB 스택 + 컨텍스트 스위칭 비용
- 동시 200 요청 → 스레드 200개 → 메모리 ~200MB + OS 스케줄러 부담
- 블로킹 I/O(`Thread.sleep`, JDBC, HTTP 호출) 동안 OS 스레드 점유

### Virtual Thread의 동작
- JVM이 스케줄링하는 경량 스레드. 기본 ForkJoinPool 위에서 동작.
- 스택을 힙에 저장 → 한 스레드당 ~1KB.
- 블로킹 호출이 들어오면 **carrier 스레드(OS 스레드)에서 떼어내고(unmount)**, 다른 가상 스레드를 올림(mount). I/O 완료 시 다시 올라옴.
- 결과: **동기 코드 그대로 작성하되, 블로킹이 OS 스레드를 잡지 않음**.

### 활성화
```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```
이 한 줄로:
- Tomcat의 요청 처리 스레드가 가상 스레드
- `@Async` 의 기본 executor 가 가상 스레드
- 스케줄링 작업(`@Scheduled`) 가상 스레드

### 이 프로젝트에서의 효과

`FakePaymentGateway.process()`:
```java
Thread.sleep(300);  // 네트워크 지연 시뮬레이션
```
Platform Thread 모델: 300ms 동안 OS 스레드 점유. 동시 1000 결제 = OS 스레드 1000 필요.
Virtual Thread 모델: 300ms 동안 가상 스레드만 대기, OS 스레드는 다른 일. CPU 코어 수 정도(8개)면 충분.

### 주의사항

1. **synchronized 안의 블로킹**은 unmount 안 됨 (pinning).
   - JDK 21에서는 일부 보완, 24+에서는 완전 해소.
   - JDBC 드라이버, 일부 라이브러리는 내부 `synchronized` 가 있을 수 있음.
   - `-Djdk.tracePinnedThreads=full` 로 진단.

2. **DB 커넥션이 새 병목**.
   - Virtual Thread는 무한정 만들어도 되지만, HikariCP 풀은 유한.
   - `maximum-pool-size: 20` 인데 동시 100 요청이면 80개는 커넥션 대기.
   - 풀 크기를 적정선까지 올리거나, async-friendly 드라이버 검토.

3. **CPU 바운드 작업에는 효과 없음**.
   - 가상 스레드의 이득은 I/O 대기 시간에서 옴. CPU 작업은 어차피 코어 수가 한계.

4. **ThreadLocal 비싸짐**.
   - 가상 스레드는 수만 개 생성될 수 있어 ThreadLocal 메모리가 누적.
   - `ScopedValue` (Java 21 incubator) 가 대안.

## FastAPI 대응

| FastAPI (asyncio) | Spring + Virtual Threads |
|-------------------|---------------------------|
| `async def` | 동기 메서드 그대로 |
| `await` | (없음 — 자동 unmount) |
| `asyncio.gather()` | `Thread.startVirtualThread` 또는 `StructuredTaskScope` |
| 색깔 함수 문제 | 없음 (모든 코드가 같은 모델) |
| 이벤트 루프 | ForkJoinPool carrier |

Python의 async는 명시적인 await가 필요(소위 "function coloring"). Java의 가상 스레드는 동기 코드를 그대로 두면 되므로 기존 라이브러리/패턴이 그대로 동작.
