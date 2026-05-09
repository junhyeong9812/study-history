# Spring 테스트의 동적 프로퍼티 주입 — `DynamicPropertyRegistry` & `DynamicPropertySource`

> 테스트 시점에 결정되는 값(예: 무작위 포트로 뜬 Postgres 컨테이너의 JDBC URL)을 Spring `Environment`에 주입하는 메커니즘.
> `application.yml`의 정적 값으로는 표현할 수 없는 것을 Spring 테스트 컨텍스트가 어떻게 해결하는지 정리한 학습 노트.

---

## 0. 분석 대상 코드

```java
// common/src/test/java/com/hybrid/common/support/KafkaIntegrationTestBase.java
package com.hybrid.common.support;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.KafkaContainer;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

@SpringBootTest
@Testcontainers
public abstract class KafkaIntegrationTestBase {

    @Container
    static final PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:17");

    @Container
    static final KafkaContainer kafka = new KafkaContainer(
            DockerImageName.parse("confluentinc/cp-kafka:7.7.0"));

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
        r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
}
```

이 짧은 코드 안에 **"테스트 컨테이너의 무작위 포트를 Spring 컨텍스트에 알려주기"** 라는 비단순한 문제가 깔끔히 풀려 있다. 풀어보자.

---

## 1. 어떤 문제를 푸는가

### 1.1 정적 프로퍼티의 한계

운영 환경의 데이터베이스 접속은 빌드 시점에 알 수 있다.

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://prod-db:5432/hybrid
    username: hybrid
    password: ${DB_PASSWORD}
```

값이 **고정**이거나 **환경변수**로 풀 수 있다. 여기까진 yml로 끝.

테스트는 다르다.

```yaml
# 이런 게 안 됨 — 테스트 시작 전엔 컨테이너가 없으니까 포트도 없음
spring:
  datasource:
    url: jdbc:postgresql://localhost:???????/test
                                   ^^^^^ 컨테이너 띄울 때 결정되는 무작위 포트
```

Testcontainers의 PostgreSQL은 **호스트 측 포트를 무작위로** 매핑한다 (`5432:5432` 충돌 방지). 그래서 컨테이너가 `start()`된 직후에야 `pg.getJdbcUrl()`이 의미 있는 값을 돌려준다.

### 1.2 시도해볼 만한 옛 접근들

```java
// ❌ @TestPropertySource — 컴파일 타임 상수만 가능
@TestPropertySource(properties = "spring.datasource.url=" + pg.getJdbcUrl())
                                                            // 안 됨, static 초기화 시점 모호
```

```java
// ❌ static 블록에서 System.setProperty — 동작은 하지만 더러움
static {
    pg.start();
    System.setProperty("spring.datasource.url", pg.getJdbcUrl());
}
// 시스템 프로퍼티 누수, 테스트 격리 깨짐, 컨테이너 라이프사이클 어긋남
```

```java
// ❌ ApplicationContextInitializer — 동작은 하지만 보일러플레이트 폭발
@ContextConfiguration(initializers = MyInitializer.class)
class MyTest { ... }

class MyInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    @Override public void initialize(ConfigurableApplicationContext ctx) {
        TestPropertyValues.of("spring.datasource.url=" + pg.getJdbcUrl()).applyTo(ctx);
    }
}
```

전부 동작은 하지만 **테스트 한 줄당 보일러플레이트 폭발** 또는 **격리 깨짐**.

### 1.3 해결 — `@DynamicPropertySource`

Spring 5.2.5(2020년 3월)에 도입된 우아한 해결책.

```java
@DynamicPropertySource
static void props(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", pg::getJdbcUrl);   // 메서드 레퍼런스 = lazy Supplier
}
```

- **테스트 컨텍스트 시작 직전**에 호출됨.
- 이미 `@Container`가 컨테이너를 띄운 상태라 `pg.getJdbcUrl()` 호출 가능.
- 람다(메서드 레퍼런스)로 등록하므로 Spring은 **값이 필요할 때마다** 호출 → 항상 현재 컨테이너 상태 반영.

---

## 2. 메커니즘 풀어보기

### 2.1 두 가지 핵심 타입

```java
package org.springframework.test.context;

public interface DynamicPropertyRegistry {
    void add(String name, Supplier<Object> valueSupplier);
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DynamicPropertySource {}
```

- **`DynamicPropertyRegistry`** — 등록을 받는 콜백 인터페이스. 메서드 하나(`add`).
- **`@DynamicPropertySource`** — "이 static 메서드를 컨텍스트 시작 전에 호출해서 위 Registry에 프로퍼티를 등록받아라"는 마커.

### 2.2 메서드의 시그니처 규칙

`@DynamicPropertySource` 메서드는 다음을 만족해야 한다.

- `static`이어야 함.
- 반환 타입 `void`.
- 매개변수 정확히 하나 — `DynamicPropertyRegistry` 타입.
- 접근 제어자 자유 (보통 `package-private` 또는 `private`).

```java
@DynamicPropertySource
static void props(DynamicPropertyRegistry r) { ... }   // ✅ 표준 형태
```

### 2.3 `Supplier`로 등록하는 이유

```java
r.add("spring.datasource.url", pg::getJdbcUrl);    // 메서드 레퍼런스 (Supplier)
```

이게 함수형 인터페이스 `Supplier<Object>`로 등록되는 이유:

- `@DynamicPropertySource` 호출 **시점**과 Spring이 그 값을 **읽는 시점**이 다를 수 있음.
- 등록 시점에 즉시 평가하면 컨테이너 라이프사이클이 어긋날 위험.
- 람다로 등록 → Spring이 필요할 때 호출 → 그 시점의 최신 값을 확실히 반환.

`pg::getJdbcUrl`은 컨테이너의 현재 상태를 매번 새로 읽으니, 컨테이너가 `restart()` 같은 일을 겪어도 새 URL이 반영된다.

### 2.4 라이프사이클 시간순

```
[T+0]   JUnit 5 클래스 로드 → static 필드 초기화
        @Container static PostgreSQLContainer<?> pg = new ...   (인스턴스만)
        @Container static KafkaContainer kafka = new ...

[T+0.1] @Testcontainers 확장 활성화
        → @BeforeAll 시점에 pg.start(), kafka.start() 호출
        → Docker가 컨테이너 띄움, 무작위 포트 매핑

[T+~3]  컨테이너들 ready

[T+~3]  Spring Test 컨텍스트 부팅 시작 직전
        → @DynamicPropertySource 메서드 들 검색
        → KafkaIntegrationTestBase.props(registry) 호출
        → registry.add(...) 4번 — Supplier들이 등록됨

[T+~3]  Spring Boot 컨텍스트 초기화
        → "spring.datasource.url" 프로퍼티 필요
        → 등록된 Supplier 호출 → pg.getJdbcUrl() 결과 반환
        → HikariCP 데이터소스가 그 URL로 연결

[T+~5]  컨텍스트 초기화 완료, 테스트 실행

[T+종료] 컨테이너 정지, ThreadLocal 정리
```

핵심: **`@Container`가 먼저 동작 → `@DynamicPropertySource`는 컨테이너가 떠 있는 상태에서 호출됨**. 이 순서가 보장되니 안전.

### 2.5 등록된 프로퍼티의 우선순위

Spring `Environment`에는 여러 PropertySource가 있다 (yml, env vars, command line 등). 동적으로 등록된 프로퍼티는 **가장 높은 우선순위**로 들어간다.

```
[Spring Environment 우선순위 (높음 → 낮음)]
  ① @DynamicPropertySource로 등록된 값        ← 여기
  ② @TestPropertySource("...")
  ③ @SpringBootTest(properties = {...})
  ④ application-test.yml / application.yml
  ⑤ 시스템 프로퍼티 (-D)
  ⑥ 환경변수
```

테스트가 운영 yml의 같은 키를 덮어씌울 수 있는 이유다.

---

## 3. 등장하는 다른 패키지/타입들

`KafkaIntegrationTestBase`의 import에 등장한 모든 타입을 정리한다.

### 3.1 `org.springframework.test.context` 패키지

| 타입 | 역할 |
|------|------|
| `DynamicPropertyRegistry` | 동적 프로퍼티 콜백 — `add(name, supplier)` |
| `DynamicPropertySource` | 위 콜백을 호출할 메서드 표시 어노테이션 |
| `@TestPropertySource` (별개) | 정적 프로퍼티 파일/값 명시 |
| `@ContextConfiguration` (별개) | 컨텍스트 설정 (보통 `@SpringBootTest`가 대신함) |
| `@ActiveProfiles` (별개) | 활성 프로파일 지정 |

이 패키지는 **Spring Framework Test**(`spring-test` 모듈)에 있다 — Spring Boot 종속 아님.

### 3.2 `org.springframework.boot.test.context`

```java
import org.springframework.boot.test.context.SpringBootTest;
```

`@SpringBootTest`만 Spring Boot 쪽에 있다. 자동 구성·기본 설정·웹 환경 옵션을 제공.

### 3.3 `org.testcontainers.junit.jupiter`

```java
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
```

Testcontainers의 JUnit 5 통합. 자세한 내용은 `docs/study/testcontainers.md` 참조.

### 3.4 `org.testcontainers.containers`

```java
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.containers.KafkaContainer;
```

특정 인프라 전용 컨테이너 클래스들. 헬스체크·환경변수·기본값을 자동 설정.

### 3.5 `org.testcontainers.utility.DockerImageName`

```java
DockerImageName.parse("confluentinc/cp-kafka:7.7.0")
```

이미지 이름과 태그를 안전하게 파싱하는 유틸. Confluent Kafka처럼 비표준 이미지를 쓸 때 호환성 검증 메시지를 보여줌.

---

## 4. 비교 — 다른 프로퍼티 주입 방식과의 위치

### 4.1 정적 vs 동적

| 방식 | 값 결정 시점 | 용도 |
|------|------------|------|
| `application.yml` | 빌드/패키징 시점 | 운영 기본값 |
| `application-test.yml` | 빌드 시점 | 테스트 정적 설정 |
| `@TestPropertySource("classpath:test.properties")` | 빌드 시점 | 특정 테스트 클래스만 다른 정적 값 |
| `@TestPropertySource(properties = "...")` | 컴파일 타임 상수 | 인라인 정적 값 |
| `@SpringBootTest(properties = {...})` | 컴파일 타임 상수 | `@TestPropertySource`와 비슷 |
| **`@DynamicPropertySource`** | **테스트 시작 시점** | **컨테이너 포트 등 런타임 값** |
| `ApplicationContextInitializer` | 컨텍스트 초기화 단계 | 가장 자유로우나 보일러플레이트 큼 |

### 4.2 언제 `@DynamicPropertySource`가 맞는가

- **Testcontainers**의 무작위 포트 / 임의 사용자명 (이 프로젝트의 사례)
- **Wiremock**으로 띄운 가짜 외부 API의 동적 포트
- **K3s/EmbeddedKafka** 등 임베디드 인프라의 런타임 결정 값
- 테스트가 사용하는 외부 시스템의 **헬스체크 후 실제 endpoint URL**

### 4.3 언제 `@TestPropertySource`로 충분한가

- 테스트 전용 로깅 레벨 / 비활성화 플래그
- 고정된 모킹 URL (예: `http://localhost:0` 같은 자동 포트)
- 운영과 다른 정적 설정

---

## 5. Spring 6.2의 진화 — `DynamicPropertyRegistrar`

Spring 6.2(Spring Boot 3.4+)에서 같은 일을 **빈으로** 등록할 수 있는 경로가 추가됐다.

```java
@TestConfiguration
class TestcontainersConfig {

    @Bean
    PostgreSQLContainer<?> postgres() {
        return new PostgreSQLContainer<>("postgres:17");
    }

    @Bean
    DynamicPropertyRegistrar postgresProperties(PostgreSQLContainer<?> pg) {
        return registry -> {
            registry.add("spring.datasource.url", pg::getJdbcUrl);
            registry.add("spring.datasource.username", pg::getUsername);
            registry.add("spring.datasource.password", pg::getPassword);
        };
    }
}
```

- `@DynamicPropertySource` 메서드 대신 **빈 등록**으로 대체.
- 컨테이너도 빈으로 관리 → `@ServiceConnection`(또 다른 신기능)과 조합하면 더 짧아짐.
- 이 프로젝트는 학습 흐름상 **`@DynamicPropertySource` 어노테이션 방식**을 유지 — 메커니즘이 더 직관적.

### 5.1 `@ServiceConnection` (Spring Boot 3.1+)

```java
@Container
@ServiceConnection
static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:17");
```

Spring Boot가 컨테이너 타입을 보고 **`@DynamicPropertySource` 코드를 자동 생성**한다. 가장 짧음.

| 방식 | 코드량 | 학습 가치 | 권장 시점 |
|------|------|---------|---------|
| `@DynamicPropertySource` | 짧음 | 메커니즘 직관 | 학습 / 명시성 중요 |
| `DynamicPropertyRegistrar` 빈 | 중간 | 빈 라이프사이클과 결합 | 큰 테스트 베이스 |
| `@ServiceConnection` | 가장 짧음 | 마법 (내부 동작 안 보임) | 실무 효율 |

이 프로젝트는 **`@DynamicPropertySource`** 를 선택했다 — 동작 원리를 코드로 보이게 하기 위함.

---

## 6. 자주 만나는 함정

### 6.1 함정 1 — `@Container`를 안 붙임

```java
static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:17");
// @Container 누락 → 컨테이너가 자동 시작 안 됨
```

`@DynamicPropertySource`가 호출될 때 `pg.getJdbcUrl()`이 **컨테이너 미시작 예외**를 던진다.

### 6.2 함정 2 — `static`을 빼먹음

```java
@Container
PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:17");   // ❌ instance field
```

instance 필드에 `@Container`를 붙이면 **테스트 메서드마다** 컨테이너가 새로 뜬다 (느림).
또한 `@DynamicPropertySource` 메서드는 **반드시 static**이라 instance 필드를 못 본다 → 호출 오류.

### 6.3 함정 3 — 등록 시점에 즉시 평가

```java
@DynamicPropertySource
static void props(DynamicPropertyRegistry r) {
    String url = pg.getJdbcUrl();              // 즉시 평가
    r.add("spring.datasource.url", () -> url); // Supplier 안엔 캡처된 값
}
```

대부분의 경우 동작하지만, 컨테이너 재시작 같은 시나리오에서 옛 URL을 들고 있게 된다. **메서드 레퍼런스(`pg::getJdbcUrl`)** 가 안전한 표준.

### 6.4 함정 4 — 컨텍스트 캐시 충돌

```java
class TestA extends KafkaIntegrationTestBase {}
class TestB extends KafkaIntegrationTestBase {}
```

Spring 테스트는 컨텍스트를 캐시한다. 같은 베이스를 상속한 두 테스트는 **같은 컨텍스트, 같은 컨테이너**를 공유한다 — 보통 좋은 일이다.

하지만 한 테스트가 DB 상태를 더럽히면 다른 테스트에 누수가 생긴다. 해결:

- `@Transactional` + 자동 롤백 (가장 단순)
- `@DirtiesContext` (컨텍스트를 일부러 무효화 — 느려짐)
- `@Sql(scripts = "/cleanup.sql")` (테스트 전후 정리 스크립트)

### 6.5 함정 5 — 컨텍스트가 컨테이너보다 먼저 떠버림

극히 드문 케이스 — `@DynamicPropertySource` 메서드 안에서 컨테이너를 명시적으로 시작하면 순서가 어긋날 수 있음.

```java
// ❌ 권장 안 함
@DynamicPropertySource
static void props(DynamicPropertyRegistry r) {
    pg.start();   // 여기서 컨테이너를 띄우면 @Container와 라이프사이클 어긋남
    ...
}
```

`@Container`로 라이프사이클을 맡기고, `@DynamicPropertySource`에선 등록만 한다.

---

## 7. 디버깅 팁

### 7.1 어떤 프로퍼티가 들어갔는지 보기

```java
@Autowired Environment env;

@Test
void debug() {
    System.out.println(env.getProperty("spring.datasource.url"));
    // jdbc:postgresql://localhost:32789/test 같은 실제 값 확인 가능
}
```

### 7.2 `@DynamicPropertySource` 메서드가 호출됐는지 확인

```java
@DynamicPropertySource
static void props(DynamicPropertyRegistry r) {
    System.out.println("=== DynamicPropertySource called ===");
    r.add("spring.datasource.url", pg::getJdbcUrl);
}
```

호출이 안 된다면:
- `static`이 아니거나
- 매개변수 타입이 잘못됐거나
- 어노테이션 import가 잘못됐거나(이상한 다른 `DynamicPropertySource`를 import한 경우 등).

### 7.3 컨텍스트 캐시 문제 의심 시

```bash
./gradlew test --rerun-tasks    # 캐시 무시하고 다시 실행
```

또는 IDE에서 일시적으로 `@DirtiesContext`를 붙여 컨텍스트를 매번 새로 띄워본다.

---

## 8. 한 줄 요약

> **`@DynamicPropertySource`는 "테스트 시작 시점에 결정되는 값(예: Testcontainer의 무작위 포트)을 Spring `Environment`에 동적으로 주입"하는 표준 메커니즘이다.** `DynamicPropertyRegistry.add(name, supplier)`로 등록하면 Spring이 프로퍼티를 읽는 시점마다 supplier를 호출해 최신 값을 반영한다. `@Container`가 컨테이너 라이프사이클을 맡고 `@DynamicPropertySource`는 등록만 하는 분업 구조.
> `@TestPropertySource`(정적)와 `ApplicationContextInitializer`(보일러플레이트) 사이에서, **컨테이너 기반 통합 테스트의 정답**.

---

## 9. 함께 보면 좋은 자료

- 공식: [Spring Framework Reference — Context Configuration with Dynamic Property Sources](https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/ctx-management/dynamic-property-sources.html)
- 공식 API: [`DynamicPropertyRegistry`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/context/DynamicPropertyRegistry.html)
- `docs/study/testcontainers.md` — `@Container` / `@Testcontainers`의 동작 원리
- `docs/phase-1-tdd.md` Step 4 — `TransactionalEventPublishingTest`가 같은 메커니즘을 어떻게 쓰는지
- `docs/phase-2-tdd.md` 0.3 — Kafka까지 확장한 `KafkaIntegrationTestBase`

---

## 10. 이 프로젝트에서의 사용처

| 테스트 베이스 | 동적 등록 프로퍼티 |
|------------|-----------------|
| `TransactionalEventPublishingTest` (Phase 1) | `spring.datasource.url`, `username`, `password` (Postgres) |
| `KafkaIntegrationTestBase` (Phase 2/3 공용) | 위 3개 + `spring.kafka.bootstrap-servers` (Kafka) |
| `NotificationStandaloneTest` (Phase 3) | 위 4개 (독립 부팅 시나리오) |

모든 통합 테스트의 기반 → **이 메커니즘을 한 번 이해하면 모든 통합 테스트 환경의 비밀이 풀린다**.
