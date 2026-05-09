# Micrometer — JVM 애플리케이션의 메트릭 표준 facade

> 이 프로젝트의 `OutboxMetrics`, `OutboxRelay`, `InboxConsumer`가 사용하는 `Gauge`, `Counter`, `MeterRegistry`를 풀어 정리한 학습 노트.
> "메트릭이 어디로 가서 어떻게 Prometheus에 노출되는가"의 흐름을 코드 레벨에서 이해한다.

---

## 0. 분석 대상 코드

```java
// common/src/main/java/com/hybrid/common/outbox/OutboxMetrics.java
package com.hybrid.common.outbox;

import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.stereotype.Component;

@Component
public class OutboxMetrics {

    public OutboxMetrics(MeterRegistry registry, OutboxRepository repo) {
        Gauge.builder("outbox.pending.count",
                        () -> repo.countByStatus(OutboxStatus.PENDING))
                .register(registry);
        Gauge.builder("outbox.deadletter.count",
                        () -> repo.countByStatus(OutboxStatus.DEAD_LETTER))
                .register(registry);
    }
}
```

생성자 안에서 게이지 두 개를 등록하는 게 전부다. 이 짧은 코드 안에 Micrometer의 핵심 사상이 들어있다.

---

## 1. Micrometer란

> JVM 애플리케이션의 **메트릭 수집을 위한 facade(추상화 계층)**. SLF4J가 로깅 백엔드(logback, log4j 등)를 추상화하듯, Micrometer는 메트릭 백엔드(Prometheus, Datadog, New Relic, CloudWatch 등)를 추상화한다.

```
[애플리케이션 코드]
   meter.counter("outbox.publish.success").increment();
   ↓
[Micrometer API]
   ↓
[MeterRegistry 구현체 — 백엔드별로 다름]
   ├─ PrometheusMeterRegistry  → /actuator/prometheus 엔드포인트로 pull
   ├─ DatadogMeterRegistry     → Datadog API로 push
   ├─ CloudWatchMeterRegistry  → AWS CloudWatch로 push
   └─ SimpleMeterRegistry      → 메모리만 (테스트용)
```

애플리케이션 코드는 어떤 백엔드를 쓰는지 모른다. **모니터링 도구를 바꿀 때 코드 변경이 필요 없다**.

### 1.1 의존성

```groovy
// common/build.gradle
implementation 'io.micrometer:micrometer-registry-prometheus'
```

`micrometer-core`(추상)는 위 모듈이 transitive로 끌고 옴. Prometheus 연동을 위한 백엔드 구현체가 함께 들어옴.

### 1.2 Spring Boot 통합

`spring-boot-starter-actuator`가 클래스패스에 있고 `micrometer-registry-prometheus`도 있으면, Spring Boot가 자동으로:

1. `PrometheusMeterRegistry` 빈 등록 → `MeterRegistry` 인터페이스로 주입 가능.
2. `/actuator/prometheus` 엔드포인트 활성화 (yml에서 `management.endpoints.web.exposure.include: prometheus` 필요).
3. JVM 기본 메트릭(GC, 메모리, 스레드) + Tomcat·HikariCP·Logback 등 자동 수집.

---

## 2. 핵심 타입들

### 2.1 `MeterRegistry` — 중앙 저장소

```java
public abstract class MeterRegistry {
    public Counter counter(String name, String... tags);
    public Timer timer(String name, String... tags);
    public Gauge.Builder gauge(String name, Supplier<Number>);
    public DistributionSummary summary(String name);
    ...
}
```

**모든 메트릭의 등록·조회 진입점**. 보통 Spring이 `@Autowired`로 주입.

### 2.2 Meter 종류

Micrometer는 5가지 기본 타입을 제공한다.

| 타입 | 의미 | 예시 |
|------|------|------|
| **Counter** | 단조 증가만 (감소 불가) | 누적 발행 성공 수, 에러 수 |
| **Gauge** | 임의 시점의 측정값 | 현재 큐 크기, 메모리 사용량 |
| **Timer** | 지연 + 호출 횟수 | HTTP 응답 시간, DB 쿼리 시간 |
| **DistributionSummary** | 분포 (값들의 통계) | 요청 페이로드 크기 분포 |
| **LongTaskTimer** | 진행 중인 장기 작업 | 백그라운드 잡 실행 시간 |

### 2.3 이 프로젝트가 쓰는 두 종류

```java
// Counter — 발행 성공/실패 수 누적
meter.counter("outbox.publish.success").increment();
meter.counter("outbox.publish.failure").increment();
meter.counter("inbox.processed.count").increment();
meter.counter("inbox.duplicate.count").increment();

// Gauge — 현재 PENDING outbox 행 수
Gauge.builder("outbox.pending.count", () -> repo.countByStatus(OutboxStatus.PENDING))
    .register(registry);
```

---

## 3. `Counter` — 단조 증가 카운터

```java
Counter counter = registry.counter("outbox.publish.success");
counter.increment();          // +1
counter.increment(5.0);       // +5

double current = counter.count();   // 현재 누적값 (테스트용)
```

특징:
- **감소 불가** — `decrement()` 메서드가 없다.
- 값 자체보다 **변화율(per-second)** 이 더 의미 있음 → Prometheus가 `rate()`로 계산.
- "에러가 늘었나", "처리량이 떨어졌나" 같은 추세 분석에 적합.

### 3.1 명명 규칙

```
outbox.publish.success      — 점(.) 으로 계층 표현
outbox.publish.failure
inbox.duplicate.count
order.recovered
```

Micrometer의 점 표기는 백엔드별로 자동 변환:
- Prometheus: `outbox_publish_success_total` (점 → 언더스코어, `_total` 자동 붙임)
- Datadog: `outbox.publish.success` 그대로
- Graphite: 점 표기 그대로

→ **애플리케이션 코드는 점 표기로 통일**, 백엔드별 변환은 라이브러리에 맡긴다.

---

## 4. `Gauge` — 현재 시점 측정값

```java
Gauge.builder("outbox.pending.count", () -> repo.countByStatus(OutboxStatus.PENDING))
    .register(registry);
```

핵심: **람다(Supplier)로 등록**. Micrometer가 메트릭을 노출할 때마다 람다를 호출해 **현재 값**을 얻는다.

```
[메트릭 수집 시점]
Prometheus가 /actuator/prometheus를 GET (예: 15초마다)
   ↓
Micrometer가 등록된 모든 게이지의 Supplier 호출
   ↓
람다 () -> repo.countByStatus(OutboxStatus.PENDING) 실행
   ↓
DB에서 COUNT(*) 쿼리
   ↓
응답 본문에 "outbox_pending_count 5.0" 같은 형태로 포함
```

### 4.1 Counter vs Gauge — 언제 무엇을 쓰나

```
값이 "증가량"인가, "현재 상태"인가?

  ├ 증가량 (이벤트 횟수) → Counter
  │   "발행 성공이 1번 일어났다" → counter.increment()
  │
  └ 현재 상태 (스냅샷) → Gauge
      "지금 PENDING outbox가 5개다" → repo.countByStatus(PENDING)
```

**같은 사실을 둘 다로 표현 가능한 경우**:

```java
// 큐 크기를 카운터 두 개로 표현 (들어온 것, 나간 것)
counter.counter("queue.enqueued").increment();
counter.counter("queue.dequeued").increment();
// 현재 크기 = enqueued - dequeued (Prometheus 쿼리에서 계산)

// 또는 게이지 하나로 직접
Gauge.builder("queue.size", queue::size).register(registry);
```

Counter는 **재시작에 강하지만 계산 필요**, Gauge는 **즉시 값을 얻지만 재시작 시 0부터**.

### 4.2 이 프로젝트의 게이지가 DB 쿼리를 호출하는 이유

```java
() -> repo.countByStatus(OutboxStatus.PENDING)
```

매번 메트릭 수집 시 SQL `COUNT(*)`를 실행. 무겁지 않은가?

**부분 인덱스가 보호한다**:

```sql
CREATE INDEX idx_outbox_pending ON outbox(created_at) WHERE status = 'PENDING';
```

PENDING 행만 인덱스에 들어 있으니 카운트가 매우 빠름. PUBLISHED 행이 수백만이어도 PENDING이 100건이면 100건만 스캔.

대안으로 **카운터 두 개로 추적**(증가/감소)도 가능하지만 코드가 복잡해진다 — DB 쿼리가 충분히 빠른 한 게이지가 깔끔.

---

## 5. Tags — 메트릭의 차원 분리

같은 메트릭을 여러 차원으로 쪼갤 때 사용.

```java
meter.counter("outbox.publish",
    "result", "success",
    "topic", "order-events").increment();

meter.counter("outbox.publish",
    "result", "failure",
    "topic", "order-events").increment();
```

Prometheus에선 다음과 같이 노출:

```
outbox_publish_total{result="success",topic="order-events"} 1234
outbox_publish_total{result="failure",topic="order-events"} 56
```

쿼리:
```promql
sum(rate(outbox_publish_total[5m])) by (result)
# 성공/실패별 초당 발행 수
```

이 프로젝트는 단순화를 위해 태그 대신 **메트릭 이름을 분리**(`success`, `failure`)했다. 운영 시스템에선 태그를 적극 활용.

---

## 6. 흐름 다이어그램 — 이 프로젝트의 메트릭 파이프라인

```
[애플리케이션]
   OutboxRelay.poll()
      ├─ 발행 성공 → meter.counter("outbox.publish.success").increment()
      └─ 발행 실패 → meter.counter("outbox.publish.failure").increment()

   InboxConsumer.consume()
      ├─ 정상 처리 → meter.counter("inbox.processed.count").increment()
      └─ 중복 스킵 → meter.counter("inbox.duplicate.count").increment()

   OutboxMetrics (생성자에서 등록)
      ├─ Gauge "outbox.pending.count" — repo.countByStatus(PENDING)
      └─ Gauge "outbox.deadletter.count" — repo.countByStatus(DEAD_LETTER)

         ↓ 모두 동일한 MeterRegistry 인스턴스에 등록됨

[MeterRegistry (PrometheusMeterRegistry)]
   메모리에 모든 메트릭 보관

         ↓ /actuator/prometheus 엔드포인트가 노출

[GET /actuator/prometheus]
   응답:
   # HELP outbox_publish_success_total ...
   # TYPE outbox_publish_success_total counter
   outbox_publish_success_total 1234.0
   # HELP outbox_pending_count ...
   # TYPE outbox_pending_count gauge
   outbox_pending_count 5.0
   ...

         ↓ Prometheus 서버가 15초마다 scrape

[Prometheus]
   시계열 DB에 저장 → Grafana에서 쿼리·시각화
```

---

## 7. 테스트 — 이 프로젝트의 OutboxMetricsTest

```java
@Test
void outbox_pending_count_게이지가_PENDING_레코드_수를_반영한다() {
    outboxRepo.save(OutboxEvent.of("Order","1","OrderConfirmed","{}"));
    outboxRepo.save(OutboxEvent.of("Order","2","OrderConfirmed","{}"));

    Gauge g = registry.find("outbox.pending.count").gauge();
    assertThat(g).isNotNull();
    await().atMost(2, SECONDS).untilAsserted(() ->
        assertThat(g.value()).isEqualTo(2.0));
}
```

핵심 메서드:
- `registry.find("이름").gauge()` — 등록된 메트릭 검색.
- `g.value()` — 현재 값 (게이지면 람다 호출, 카운터면 누적값).

`await`을 쓰는 이유: 게이지의 람다가 호출되는 시점은 비동기일 수 있음 (Micrometer 내부 캐시).

```java
@Test
void relay_발행_성공_카운터가_증가한다() {
    outboxRepo.save(OutboxEvent.of("Order","1","OrderConfirmed","{}"));
    Counter before = registry.counter("outbox.publish.success");
    double v0 = before.count();

    relay.poll();

    assertThat(registry.counter("outbox.publish.success").count())
        .isGreaterThan(v0);
}
```

`registry.counter("이름")`은 **없으면 만들고 있으면 가져온다** — Map.computeIfAbsent와 같은 idempotent 패턴.

---

## 8. JVM 기본 메트릭 — 무료로 따라오는 것들

`management.endpoints.web.exposure.include: prometheus`만 켜면 자동으로:

| 메트릭 | 의미 |
|------|------|
| `jvm_memory_used_bytes{area, id}` | 힙/메타스페이스 사용량 |
| `jvm_gc_pause_seconds` | GC 일시정지 시간 (Timer) |
| `jvm_threads_live` | 활성 스레드 수 |
| `system_cpu_usage` | CPU 사용률 |
| `process_uptime_seconds` | 프로세스 가동 시간 |
| `http_server_requests_seconds` | Spring MVC 요청 시간 (Timer with tags: method, status, uri) |
| `tomcat_sessions_*` | 톰캣 세션 |
| `hikaricp_connections_*` | HikariCP 풀 상태 |
| `kafka_consumer_*` | Kafka 컨슈머 lag, 처리량 (Spring Kafka가 자동 노출) |

→ **운영에 필요한 80%는 코드 한 줄 안 써도 자동 등록**된다. 우리가 직접 만드는 건 도메인 고유 메트릭(outbox/inbox)뿐.

---

## 9. Prometheus 노출 형식

```
# HELP outbox_publish_success_total Number of successful Kafka publishes from outbox
# TYPE outbox_publish_success_total counter
outbox_publish_success_total 1234.0

# HELP outbox_pending_count Number of outbox events waiting to be published
# TYPE outbox_pending_count gauge
outbox_pending_count 5.0

# HELP http_server_requests_seconds HTTP request duration
# TYPE http_server_requests_seconds histogram
http_server_requests_seconds_bucket{method="POST",uri="/api/orders",status="201",le="0.05"} 42.0
http_server_requests_seconds_bucket{...,le="0.1"} 50.0
...
```

각 메트릭에 `# HELP`(설명)와 `# TYPE`(counter/gauge/histogram/summary) 메타가 붙음. Prometheus가 이를 보고 시계열 타입을 결정.

---

## 10. Spring Boot 자동 노출 설정

```yaml
# app/src/main/resources/application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    prometheus:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}    # 모든 메트릭에 application 태그 자동 추가
```

`management.metrics.tags`로 글로벌 태그를 박으면 여러 인스턴스에서 같은 메트릭 이름이 와도 구분 가능 (`application=hybrid-event-driven`, `instance=pod-1`).

---

## 11. 자주 만나는 함정

### 11.1 함정 1 — Gauge 람다가 GC됨

```java
// ❌ 위험
public void register(MeterRegistry registry) {
    AtomicLong counter = new AtomicLong();
    Gauge.builder("foo", counter::get).register(registry);
    // counter는 메서드 종료 시 GC 대상
    // Micrometer는 약한 참조로 들고 있어서 GC 발생 시 게이지가 NaN 반환
}
```

해결: 게이지가 참조하는 객체를 **장기 보유**하는 곳에서 등록.

```java
// ✅ 빈 생성자에서 등록
@Component
public class OutboxMetrics {
    public OutboxMetrics(MeterRegistry registry, OutboxRepository repo) {
        Gauge.builder("outbox.pending.count", () -> repo.countByStatus(...))
            .register(registry);
        // repo는 빈이라 GC 안 됨
    }
}
```

이 프로젝트가 이 패턴.

### 11.2 함정 2 — 카운터를 재할당

```java
// ❌
private Counter counter = registry.counter("outbox.success");
counter.increment();
// 이후 어디선가
counter = registry.counter("outbox.success");   // 같은 메트릭이지만
counter.increment();
```

`registry.counter(name)`은 같은 이름을 호출하면 **같은 인스턴스를 반환**. 안전하지만 매번 호출은 비용이 들 수 있어 한 번 받아 캐시하는 게 관행.

### 11.3 함정 3 — 너무 많은 태그 차원

```java
// ❌ orderId를 태그로
meter.counter("order.created", "orderId", String.valueOf(orderId));
// orderId가 100만 가지면 시계열도 100만 개 → Prometheus 메모리 폭발
```

태그 카디널리티는 **수십 ~ 수백** 수준에서 그쳐야 함. ID처럼 무한 카디널리티는 절대 태그로 안 함.

### 11.4 함정 4 — 수동 메트릭 수집을 잊음

`@Timed`, `@Counted` 같은 어노테이션을 쓸 수 있지만 모든 곳에 박을 순 없다. 코드에서 직접 `meter.counter(...).increment()`를 호출하는 게 가장 명시적.

### 11.5 함정 5 — Test에서 메트릭이 누적

```java
@SpringBootTest
class TestA { /* counter.increment() */ }

@SpringBootTest
class TestB {
    @Test void check() {
        // 같은 컨텍스트 캐시면 TestA의 카운트가 남아있을 수 있음
    }
}
```

`registry.clear()` 또는 `@DirtiesContext`로 정리. 또는 테스트가 절대값이 아닌 **delta**(전·후 차이)를 검증.

---

## 12. 한 줄 요약

> **Micrometer는 SLF4J의 메트릭 버전 — 코드는 facade(`MeterRegistry`, `Counter`, `Gauge`)에만 의존하고, 백엔드(Prometheus, Datadog, ...)는 의존성과 설정으로 결정한다.**
> 이 프로젝트는 `Counter`로 누적 이벤트를(발행 성공/실패, 처리/중복) 추적하고, `Gauge`로 현재 상태를(PENDING/DEAD_LETTER outbox 수) 노출한다. JVM·HTTP·HikariCP·Kafka 같은 운영 메트릭은 Spring Boot Actuator + Micrometer가 무료로 자동 등록.

---

## 13. 함께 보면 좋은 자료

- 공식: [Micrometer Documentation](https://micrometer.io/docs)
- 공식: [Spring Boot Actuator Metrics](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#actuator.metrics)
- `docs/study/jdbc-template.md` — outbox.pending.count 게이지가 호출하는 `countByStatus` 쿼리의 효율성
- `docs/study/event-driven-tools.md` — Kafka 컨슈머 lag 같은 Kafka 메트릭의 자동 노출
- `docs/phase-3-tdd.md` Step 1 — `OutboxMetrics`, `OutboxMetricsTest`, `PrometheusEndpointTest`

---

## 14. 이 프로젝트의 메트릭 카탈로그

| 메트릭 | 타입 | 의미 |
|------|------|------|
| `outbox.pending.count` | Gauge | 발행 대기 중인 outbox 행 수 (0에 수렴해야 정상) |
| `outbox.deadletter.count` | Gauge | DLQ 상태 행 수 (0이어야 정상, 0 초과 시 알람) |
| `outbox.publish.success` | Counter | 누적 발행 성공 횟수 |
| `outbox.publish.failure` | Counter | 누적 발행 실패 횟수 (rate가 비정상이면 알람) |
| `outbox.deadletter.moved` | Counter | DLQ로 이관된 누적 횟수 (Phase 3) |
| `inbox.processed.count` | Counter | Inbox에서 처리한 누적 메시지 수 |
| `inbox.duplicate.count` | Counter | Inbox에서 중복 스킵한 누적 수 |
| `order.recovered` | Counter | 복구 잡이 confirm시킨 주문 수 (Phase 3) |
| `eventbus.dispatch.latency` | Timer | 인메모리 EventBus 디스패치 지연 (Phase 3 Refactor) |

이 카탈로그가 곧 운영 대시보드의 핵심 패널이 된다 — Phase 3에서 Grafana JSON으로 굳히면 시스템의 건강 상태를 한 화면에서 보게 된다.
