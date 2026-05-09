# Spring Boot Health 라이브러리 정리

> 이 프로젝트의 `HealthConfig`가 어떤 라이브러리를 쓰고 있는지, 내부적으로 어떻게 동작하는지 정리한 문서.

---

## 1. 이 프로젝트에서 쓰는 코드

`src/main/java/com/shoptracker/shared/config/HealthConfig.java`:

```java
package com.shoptracker.shared.config;

import org.springframework.boot.health.contributor.Health;
import org.springframework.boot.health.contributor.HealthIndicator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class HealthConfig {

    @Bean
    public HealthIndicator eventBusHealth() {
        return () -> Health.up()
                .withDetail("type", "spring-modulith-events")
                .build();
    }
}
```

`build.gradle`의 의존성:

```gradle
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

`application.yml` 노출 설정:

```yaml
spring:
  management:
    endpoints:
      web:
        exposure:
          include: health, info, metrics, prometheus
```

즉 **Spring Boot Actuator**가 제공하는 **Health 시스템**을 사용한다.

---

## 2. 어떤 라이브러리인가

### 2.1 Spring Boot Actuator

`spring-boot-starter-actuator`는 운영(operations) 기능을 모은 스타터다.

| 기능 | 설명 |
|------|------|
| `/actuator/health` | 애플리케이션 상태(UP/DOWN/OUT_OF_SERVICE/UNKNOWN) 확인 |
| `/actuator/info` | 빌드/버전/커스텀 정보 |
| `/actuator/metrics` | Micrometer 메트릭 |
| `/actuator/prometheus` | Prometheus 스크레이프 엔드포인트 |
| `/actuator/env`, `/loggers`, ... | 운영 진단 |

이 중 Health는 가장 많이 쓰이는 엔드포인트이고, 쿠버네티스의 `livenessProbe` / `readinessProbe`, AWS ELB의 헬스체크, 모니터링 시스템 알람의 기준점으로 활용된다.

### 2.2 Spring Boot 3.4+ 모듈 분리

이 프로젝트의 import 경로:

```java
import org.springframework.boot.health.contributor.Health;
import org.springframework.boot.health.contributor.HealthIndicator;
```

기존(Boot 3.3 이하)의 경로는:

```java
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
```

Spring Boot 3.4부터 Health 관련 코어 추상이 `org.springframework.boot.actuate.*`에서 **`org.springframework.boot.health.*`** 모듈(`spring-boot-health`)로 분리됐다. 액추에이터(웹 노출)와 Health(상태 판단) 책임을 나눈 것이다.

| 모듈 | 역할 |
|------|------|
| `spring-boot-health` | `Health`, `HealthIndicator`, `Status` 등 도메인 모델 |
| `spring-boot-actuator` | `HealthEndpoint` 등 엔드포인트 노출 |
| `spring-boot-actuator-autoconfigure` | 자동 설정 |

이 프로젝트는 새 패키지(`o.s.b.health.contributor`)를 쓰고 있으므로 Boot 3.4+ 환경이다.

---

## 3. 핵심 추상 4가지

### 3.1 `Status`

상태를 나타내는 값 객체. 미리 정의된 상수 4개.

```java
Status.UP              // 정상
Status.DOWN            // 비정상 (헬스체크 실패 → HTTP 503)
Status.OUT_OF_SERVICE  // 의도적 차단 (점검 등)
Status.UNKNOWN         // 판단 불가
```

내부적으로 `code(String)` + `description(String)`을 가진 단순 값이다. 커스텀 상태도 만들 수 있다(`new Status("DEGRADED")`).

### 3.2 `Health`

`Status` + 부가 정보(`details: Map<String, Object>`)를 묶은 결과 객체. 빌더로 만든다.

```java
Health.up()
      .withDetail("type", "spring-modulith-events")
      .withDetail("queueSize", 12)
      .build();

Health.down(exception)
      .withDetail("db", "orders")
      .build();
```

직렬화하면 다음과 같은 JSON이 된다:

```json
{
  "status": "UP",
  "details": {
    "type": "spring-modulith-events",
    "queueSize": 12
  }
}
```

### 3.3 `HealthIndicator`

**상태를 판단하는 단일 메서드 인터페이스.**

```java
@FunctionalInterface
public interface HealthIndicator extends HealthContributor {
    Health health();
}
```

Bean으로 등록하면 자동으로 수집된다. Bean 이름이 컴포넌트 이름이 된다. 이 프로젝트 코드에서 `eventBusHealth`라는 메서드명이 곧 `/actuator/health` 응답의 `eventBus` 키가 된다(접미사 `Health`/`HealthIndicator`는 자동 제거됨).

### 3.4 `HealthContributor` 계층

```
HealthContributor (마커)
├── HealthIndicator       (단일 컴포넌트, 동기)
├── ReactiveHealthIndicator (단일, 리액티브 — Mono<Health>)
└── CompositeHealthContributor (여러 자식을 묶음, 트리 구조)
```

`db.datasource1`, `db.datasource2` 처럼 **계층적 헬스 트리**를 만들 때 `CompositeHealthContributor`를 쓴다.

---

## 4. 내부 동작 흐름

요청부터 응답까지의 흐름을 따라가 보자.

### 4.1 자동 등록

부트가 시작될 때 `HealthContributorAutoConfiguration`이 동작한다.

1. 컨테이너에서 `HealthIndicator` 타입의 모든 Bean을 수집한다.
2. `HealthContributorRegistry`(기본 구현 `DefaultHealthContributorRegistry`)에 Bean 이름을 키로 등록한다.
3. 기본 제공되는 indicator도 클래스패스에 따라 자동 등록된다.

기본 제공되는 indicator 예시:

| 클래스 | 트리거 조건 |
|--------|-----------|
| `DataSourceHealthIndicator` | `DataSource` Bean 존재 시 (`SELECT 1` 류 실행) |
| `DiskSpaceHealthIndicator` | 항상 등록 (디스크 임계치 초과 시 DOWN) |
| `RedisHealthIndicator` | Redis 클라이언트 Bean 존재 시 |
| `MailHealthIndicator` | `JavaMailSender` Bean 존재 시 |
| `PingHealthIndicator` | 항상 (단순 UP — fallback) |

이 프로젝트 기준 **자동 등록되는 것**: `db`(JPA/HikariCP 있음), `diskSpace`, `ping`, `livenessState`, `readinessState`. 거기에 우리가 만든 `eventBus`가 추가된다.

### 4.2 요청 처리: `HealthEndpoint`

```
GET /actuator/health
        ↓
HealthEndpoint.health()
        ↓
HealthEndpointSupport.getHealth(...)
        ↓
HealthContributorRegistry 순회
        ↓
각 HealthIndicator.health() 호출 (병렬 가능)
        ↓
StatusAggregator로 전체 상태 집계
        ↓
HttpCodeStatusMapper로 HTTP 상태코드 매핑
        ↓
JSON 응답
```

핵심 협력자:

- **`StatusAggregator`** (기본: `SimpleStatusAggregator`)
  - 자식들의 상태를 모아 부모 상태를 결정한다.
  - 우선순위: `DOWN > OUT_OF_SERVICE > UP > UNKNOWN`
  - 즉 **하나라도 DOWN이면 전체 DOWN**.

- **`HttpCodeStatusMapper`** (기본: `SimpleHttpCodeStatusMapper`)
  - `UP` → 200
  - `DOWN` → 503
  - `OUT_OF_SERVICE` → 503
  - `UNKNOWN` → 200

- **`ShowDetails` / `ShowComponents`** (Boot의 `management.endpoint.health.show-details` 설정)
  - `never` (기본): 상태만 노출
  - `when-authorized`: 인증된 사용자에게만 details 노출
  - `always`: 항상 details 노출

### 4.3 응답 형태

`show-details: always`일 때:

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": { "database": "PostgreSQL", "validationQuery": "isValid()" }
    },
    "diskSpace": {
      "status": "UP",
      "details": { "total": 500107862016, "free": 250053931008, "threshold": 10485760 }
    },
    "eventBus": {
      "status": "UP",
      "details": { "type": "spring-modulith-events" }
    },
    "ping": { "status": "UP" }
  }
}
```

`show-details: never`(기본):

```json
{ "status": "UP" }
```

---

## 5. Health Group (그룹화)

쿠버네티스 환경에서 자주 쓰는 기능.

```yaml
management:
  endpoint:
    health:
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState, db, eventBus
```

이렇게 하면:

- `GET /actuator/health/liveness` → 살아있는지만(이벤트 루프 죽음 등)
- `GET /actuator/health/readiness` → 트래픽 받을 준비 됐는지(DB, 외부 의존 OK인지)

Spring Boot는 `LivenessStateHealthIndicator`, `ReadinessStateHealthIndicator`를 자동 등록한다. 이는 `ApplicationAvailability` 빈을 통해 애플리케이션이 발행하는 라이프사이클 이벤트(`AvailabilityChangeEvent`)를 추적한다.

---

## 6. `HealthIndicator` 작성 패턴

### 6.1 람다 (이 프로젝트의 방식)

```java
@Bean
public HealthIndicator eventBusHealth() {
    return () -> Health.up()
            .withDetail("type", "spring-modulith-events")
            .build();
}
```

- 간단한 정적 정보 노출에 적합.
- 실제로 외부 시스템을 체크하지 **않는다.** (단순히 "이벤트 버스가 설정돼 있다"는 표시)

### 6.2 `AbstractHealthIndicator` 상속

체크 로직에서 예외가 날 수 있을 때 권장.

```java
@Component
public class RedisHealthIndicator extends AbstractHealthIndicator {
    private final RedisConnectionFactory factory;

    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        try (var conn = factory.getConnection()) {
            String pong = conn.ping();
            builder.up().withDetail("ping", pong);
        }
    }
}
```

- `doHealthCheck`에서 던진 예외는 `AbstractHealthIndicator`가 잡아서 자동으로 `Health.down(ex)`로 변환한다.
- `management.endpoint.health.logging.slow-indicator-threshold`로 느린 indicator를 로그에 남길 수 있다.

### 6.3 리액티브

WebFlux 환경:

```java
@Component
public class MyReactiveIndicator implements ReactiveHealthIndicator {
    @Override
    public Mono<Health> health() {
        return webClient.get().uri("/ping").retrieve()
                .toBodilessEntity()
                .map(r -> Health.up().build())
                .onErrorResume(e -> Mono.just(Health.down(e).build()));
    }
}
```

---

## 7. 이 프로젝트의 `eventBusHealth`에 대한 평가

**현재 구현:**

```java
return () -> Health.up().withDetail("type", "spring-modulith-events").build();
```

**의미:** 항상 `UP`을 리턴한다. 즉 실제로 Spring Modulith 이벤트 발행/소비 파이프라인이 살아있는지를 검증하지 **않는다.** "이벤트 버스를 쓰고 있다"는 메타데이터를 `/actuator/health` 응답에 노출하는 용도다.

**의도 추정:**
- 운영 모니터링에서 "이 서비스는 modulith 이벤트 기반"이라는 정보를 한눈에 보여주려는 마커.
- 향후 실제 체크(예: 미발행 이벤트 수, publication 테이블의 미완료 행 수)로 확장하기 위한 자리.

**확장 예시 (참고):**

Spring Modulith는 `EventPublicationRegistry`를 제공한다. 미완료 publication을 확인할 수 있다.

```java
@Bean
public HealthIndicator eventBusHealth(EventPublicationRegistry registry) {
    return () -> {
        var incomplete = registry.findIncompletePublications();
        var builder = incomplete.size() > 100 ? Health.down() : Health.up();
        return builder
                .withDetail("type", "spring-modulith-events")
                .withDetail("incompletePublications", incomplete.size())
                .build();
    };
}
```

이러면 미완료 이벤트가 임계치를 넘을 때 헬스체크가 `DOWN`이 되어 알람을 트리거할 수 있다.

---

## 8. 정리

| 항목 | 내용 |
|------|------|
| 라이브러리 | `spring-boot-starter-actuator` (전이로 `spring-boot-health`) |
| 핵심 추상 | `Status`, `Health`, `HealthIndicator`, `HealthContributor` |
| 등록 방식 | `HealthIndicator` 타입 Bean을 컨테이너가 자동 수집 |
| 집계 규칙 | `DOWN > OUT_OF_SERVICE > UP > UNKNOWN` (하나라도 DOWN이면 전체 DOWN) |
| 노출 경로 | `/actuator/health` (+ `/liveness`, `/readiness`) |
| Boot 3.4+ 변화 | `o.s.b.actuate.health.*` → `o.s.b.health.contributor.*`로 패키지 이동 |
| 이 프로젝트의 `eventBusHealth` | 정적 마커. 실제 검증 로직은 없으나 향후 확장 포인트 |
