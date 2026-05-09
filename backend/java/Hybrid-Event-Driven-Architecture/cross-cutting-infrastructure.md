# 횡단 인프라(Cross-cutting Infrastructure)의 위치 — 왜 `OutboxRepository`가 `common`에 있는가

> 모듈러 모놀리스에서 **여러 도메인이 공유하는 메커니즘**(이벤트 버스, Outbox, 트랜잭션 동기화 등)을 어디에 두어야 하는가.
> 이 프로젝트의 결정 근거와 더 엄격한 대안(헥사고날 Port/Adapter)을 함께 정리한 학습 노트.

---

## 0. 출발점 질문

> 아웃박스 리포지토리는 공통이라 리포지토리를 가져오는거야?

**그렇다.** 그런데 이 단순한 답 뒤에 모듈러 모놀리스의 핵심 설계 결정이 깔려 있다. 풀어보자.

---

## 1. 무엇을 "횡단 인프라"라고 부르는가

### 1.1 도메인 vs 인프라

```
[도메인 (Domain)]
  비즈니스 의미가 있는 개념
  예: Order, Payment, Notification, OrderStatus, PaymentRequested

[인프라 (Infrastructure)]
  비즈니스가 아닌 기술적 메커니즘
  예: EventBus, EventStore, OutboxEvent, OutboxRelay,
      TransactionalEventPublisher, JdbcTemplate, KafkaTemplate
```

`OrderConfirmed`는 도메인 사실(Order가 확정됐다)이지만, `OutboxEvent`는 "DB 트랜잭션과 메시지 발행을 같은 단위로 묶기 위한 그릇"이라 도메인 의미가 없다.

### 1.2 횡단(cross-cutting)이란

여러 도메인이 **같은 메커니즘**을 필요로 하는 관심사를 횡단 관심사라고 부른다.

```
      Order  ─┐
      Payment ┼─ 모두 outbox에 이벤트 INSERT 후 Kafka 발행이 필요
Notification ─┘   ← 횡단 관심사
```

이 프로젝트의 횡단 인프라:

| 컴포넌트 | 위치 | 누가 사용 |
|---------|------|---------|
| `EventBus`, `InMemoryEventBus` | `common.event` | 모든 도메인 |
| `EventStore`, `InMemoryEventStore` | `common.event` | EventBus 내부 |
| `TransactionalEventPublisher` | `common.event` | OrderService, PaymentService 등 |
| **`OutboxEvent`, `OutboxRepository`** | **`common.outbox`** | **OrderService, (향후) PaymentService 등** |
| `OutboxRelay` | `common.outbox` | 백그라운드 Relay 자체 |
| `OutboxMetrics` | `common.outbox` | 운영 관측 |
| `DeadLetterEvent`, `DeadLetterRepository` | `common.outbox` | DLQ 관리 (Phase 3) |
| `KafkaTemplate` (Spring 제공) | (외부 라이브러리) | Relay 등 |

전부 **도메인 의미는 없고, 메커니즘으로서 공유**되는 것들.

---

## 2. 왜 `common`에 두는가 — 세 가지 이유

### 2.1 Outbox는 도메인이 아니라 패턴

`OutboxEvent`의 컬럼을 보면 도메인 무관함이 드러난다.

```sql
CREATE TABLE outbox (
    aggregate_type  VARCHAR(100),    -- "Order", "Payment", "Notification" 무엇이든
    aggregate_id    VARCHAR(100),
    event_type      VARCHAR(100),
    payload         JSONB,           -- 도메인이 알아서 직렬화
    status          VARCHAR(20),
    retry_count     INT,
    ...
);
```

**도메인이 무엇이든 받아내는 일반화된 그릇**이다. 이걸 도메인별로 만드는 건 의미 없다.

```
[좋지 않은 대안 — 도메인별 outbox]
order/.../OrderOutboxEvent + OrderOutboxRepository
payment/.../PaymentOutboxEvent + PaymentOutboxRepository
notification/.../NotificationOutboxEvent + NotificationOutboxRepository

→ 같은 컬럼·같은 동작을 N개 복제
→ Relay도 N개 (또는 라우팅 로직 복잡)
→ DLQ도 N개
```

### 2.2 단일 outbox 테이블 = 단일 Repository

테이블이 하나면 Repository도 하나가 자연스럽다.

```sql
SELECT * FROM outbox WHERE status = 'PENDING' ORDER BY created_at LIMIT 100;
```

이 한 줄로 모든 도메인의 펜딩 이벤트를 가져올 수 있다. 도메인별로 쪼개면 같은 효과를 위해 N개 쿼리 + 결과 머지가 필요하다.

### 2.3 단일 Relay = 운영 단순성

```java
// common/.../OutboxRelay.java
@Scheduled(fixedDelay = 1000)
public void poll() {
    List<OutboxEvent> pending = repo.findTop100ByStatusOrderByCreatedAtAsc(OutboxStatus.PENDING);
    for (OutboxEvent e : pending) {
        kafka.send(...);   // aggregate_type 신경 안 씀
    }
}
```

운영 입장에서는 "outbox 펜딩 건수 게이지" 하나, "발행 성공 카운터" 하나, "DLQ 건수" 하나만 모니터링하면 된다. 도메인별로 쪼개면 모니터링 대상이 N배.

---

## 3. 의존 방향 — 도메인 → 인프라는 정상

```
common (인프라)
  ↑ implementation
  │
order, payment, notification (도메인)
  ↑ implementation
  │
app (조립)
```

```java
// order/src/main/java/com/hybrid/order/service/OrderService.java
import com.hybrid.common.outbox.OutboxEvent;          // ← 도메인이 인프라를 import
import com.hybrid.common.outbox.OutboxRepository;     // ← OK
```

**이 방향은 정상**이다. 비유하자면:

- 자바 코드(도메인)가 JDBC(인프라)를 import하는 건 정상.
- JDBC가 자바 코드를 import하면 이상.

원칙:

```
✅ 도메인 → 인프라 (의존 OK)
❌ 인프라 → 도메인 (역방향 금지)
❌ 도메인 → 다른 도메인 (이벤트 컨트랙트로만 통신)
```

이 프로젝트의 `common.event.contract` 패키지가 마지막 원칙을 구현한다 — 도메인끼리 직접 안 보고 contract만 본다.

---

## 4. 트레이드오프 — 도메인이 인프라를 "안다"는 것

### 4.1 현재 구조의 단점

```java
// OrderService가 outbox 메커니즘을 직접 인지
@Transactional
public void confirm(Long orderId) {
    Order o = repo.findById(orderId).orElseThrow();
    o.confirm();

    // ↓ 도메인 코드가 outbox 메커니즘을 직접 호출
    outboxRepository.save(OutboxEvent.of(
        "Order", orderId.toString(), "OrderConfirmed",
        objectMapper.writeValueAsString(...)
    ));
}
```

문제점:
- 도메인이 `OutboxEvent.of(...)` 같은 인프라 API를 직접 호출.
- JSON 직렬화 책임이 도메인 서비스에 섞임.
- "이 도메인은 outbox라는 메커니즘에 의존한다"가 코드에 명시.

### 4.2 더 엄격한 대안 — 헥사고날 Port/Adapter

```java
// order 모듈 내부 — Port 인터페이스 (도메인 언어로)
package com.hybrid.order.port;

public interface IntegrationEventPublisher {
    void publish(IntegrationEvent event);
}

public record IntegrationEvent(String type, String aggregateId, Object payload) {}
```

```java
// 도메인 서비스는 Port만 안다
@Service
public class OrderService {
    private final IntegrationEventPublisher events;   // ← Port만 의존

    public void confirm(Long orderId) {
        ...
        events.publish(new IntegrationEvent(
            "OrderConfirmed", orderId.toString(),
            new OrderConfirmedPayload(orderId, o.getAmount())
        ));
    }
}
```

```java
// common/.../OutboxAdapter.java — Port 구현체 (인프라 쪽)
@Component
public class OutboxAdapter implements IntegrationEventPublisher {
    private final OutboxRepository repo;
    private final ObjectMapper objectMapper;

    @Override
    public void publish(IntegrationEvent event) {
        repo.save(OutboxEvent.of(
            extractAggregateType(event),
            event.aggregateId(),
            event.type(),
            objectMapper.writeValueAsString(event.payload())
        ));
    }
}
```

이러면:
- 도메인은 `OutboxRepository`, `OutboxEvent`를 모름.
- "outbox라는 메커니즘"이 어댑터 한 곳에만 등장.
- outbox를 다른 메커니즘(예: 직접 Kafka 발행, AWS SQS, EventGrid)으로 갈아치울 때 도메인 변경 없음.

### 4.3 이 프로젝트가 헥사고날을 안 간 이유

학습 곡선과 가독성의 균형. 헥사고날을 풀세트로 도입하면:

| 측면 | 직접 Repository (현재) | 헥사고날 Port/Adapter |
|------|---------------------|------------------|
| 코드 라인 | 적음 | Port 인터페이스 + Adapter 구현체 + 매핑 코드 추가 |
| 학습 부담 | 낮음 (Spring + JPA만 알면 됨) | 추상 한 단계 더 |
| 테스트 용이성 | 통합 테스트 위주 | Port를 mock해 도메인 단위 테스트 가능 |
| 인프라 교체 비용 | 도메인 코드 수정 필요 | 어댑터만 새로 |
| "outbox 패턴을 배운다"는 학습 목표 | 메커니즘이 코드에 직접 보임 ✅ | 메커니즘이 어댑터에 숨음 |

이 프로젝트는 **"Outbox 패턴을 직접 만들어 이해한다"** 가 학습 목표라 메커니즘을 일부러 노출했다. 운영 시스템에 가까울수록 헥사고날이 정답이다.

---

## 5. 다른 횡단 인프라들 — 같은 사상

이 프로젝트에서 `OutboxRepository`만이 아니라 모든 인프라가 같은 패턴을 따른다.

### 5.1 `EventBus`

```
common.event.EventBus           ← 추상 인터페이스
common.event.InMemoryEventBus   ← 구현체
                  │
                  ▼
order, payment에서 @Autowired로 사용
```

도메인은 "이벤트를 발행한다"는 사실만 알고 인메모리/Kafka/RabbitMQ 어느 구현이든 무관.

### 5.2 `TransactionalEventPublisher`

```
common.event.TransactionalEventPublisher
              │
              ▼
모든 도메인 서비스가 이걸로 이벤트 발행 (커밋 후 디스패치 보장)
```

"트랜잭션 커밋 후 발행"이라는 약속이 한 곳에 모여 있고, 도메인은 그 약속만 인지.

### 5.3 `OutboxRelay`

```
common.outbox.OutboxRelay   ← @Scheduled로 단독 동작
                              어떤 도메인의 의존도 없음
```

Relay가 도메인을 모르는 게 핵심. **aggregate_type을 그저 문자열로** 처리하니 어느 도메인이든 받아낸다.

### 5.4 `DeadLetterSweeper` / `OutboxCleaner`

같은 패턴. 도메인 무관 백그라운드 작업.

### 5.5 정리

| 컴포넌트 | 도메인을 아는가 | 도메인이 아는가 |
|---------|------------|--------------|
| `EventBus` | ❌ | ✅ |
| `EventStore` | ❌ | ❌ (Bus 내부에서만) |
| `TransactionalEventPublisher` | ❌ | ✅ |
| `OutboxRepository` | ❌ | ✅ (현재 구조) |
| `OutboxRelay` | ❌ | ❌ (홀로 동작) |
| `OutboxMetrics` | ❌ | ❌ |
| `DeadLetterSweeper` | ❌ | ❌ |

**핵심**: **인프라는 도메인을 모르고, 도메인은 인프라를 안다(현재) 또는 Port 추상만 안다(헥사고날)**.

---

## 6. 언제 `common`에서 도메인 모듈로 옮길까

`OutboxRepository`를 도메인 모듈로 옮길 일은 거의 없다. 하지만 어떤 컴포넌트가 **사실은 특정 도메인의 것**이었음이 드러나면 옮긴다.

### 6.1 옮겨야 할 신호

- **그 컴포넌트의 변경이 항상 한 도메인 변경과 함께 일어남.**
- **다른 도메인이 절대 안 씀.**
- **그 컴포넌트의 컬럼·필드에 도메인 의미가 박힘** (예: `customer_id` 같은).

이런 신호가 있으면 그건 횡단 인프라가 아니라 **그 도메인의 일부**였던 것이다. `common`에 잘못 둔 셈.

### 6.2 옮길 때 절차

1. 도메인 모듈에 새 클래스 생성.
2. 의존자 import 변경.
3. `common`에서 옛 클래스 삭제.
4. 마이그레이션 (테이블 이름 / 컬럼 변경) 필요 시 새 V Flyway 파일.

이 프로젝트는 **OutboxEvent를 옮길 일이 없을 것**이다 — 정의상 도메인 무관한 그릇이라 횡단성이 본질이기 때문.

---

## 7. 다른 패턴들에서의 같은 결정

### 7.1 Spring Modulith의 `@ApplicationModule`

Spring 팀의 모듈러 모놀리스 도구는 **도메인 모듈만 명시적 모듈로 다루고**, 인프라성 코드는 모듈 외부에 둔다. 본 프로젝트의 `common` = Spring Modulith 관점에선 "모듈 외부의 공유 인프라".

### 7.2 마이크로서비스의 "공유 라이브러리"

마이크로서비스에서도 같은 문제가 있다. 보통:

- **공유 라이브러리(shared lib)** — outbox 라이브러리, observability 라이브러리 등 → 횡단 인프라.
- **각 서비스 코드** — 도메인 로직 + 위 라이브러리 사용.

규모가 다를 뿐 사상은 같다.

### 7.3 클린 아키텍처의 외부 계층

```
[Clean Architecture 동심원]
   ┌──────────────────────────────────┐
   │  Frameworks / Drivers (외부)      │  ← Outbox 테이블, JDBC 드라이버
   │  ┌────────────────────────────┐  │
   │  │  Interface Adapters         │  │  ← OutboxAdapter (헥사고날 어댑터)
   │  │  ┌──────────────────────┐  │  │
   │  │  │  Application/UseCase  │  │  │  ← OrderService
   │  │  │  ┌────────────────┐  │  │  │
   │  │  │  │  Domain/Entity │  │  │  │  ← Order, Payment (애그리거트)
   │  │  │  └────────────────┘  │  │  │
   │  │  └──────────────────────┘  │  │
   │  └────────────────────────────┘  │
   └──────────────────────────────────┘
```

규칙: **의존은 항상 안쪽으로**. 외부(인프라)가 안쪽(도메인)을 의존하지 않음.

이 프로젝트는 헥사고날을 풀로 가지 않았기 때문에 도메인이 인프라(`OutboxRepository`)를 직접 의존하지만, **방향만큼은 클린 아키텍처와 맞다** (`common` 도메인 → `order`/`payment`로의 역방향 의존이 없음).

---

## 8. 자주 만나는 함정

### 8.1 함정 1 — 인프라가 도메인을 의존하기 시작

```java
// ❌ common.outbox 안에서
import com.hybrid.order.event.OrderConfirmed;   // 인프라 → 도메인 역방향
```

이러면 **빌드 그래프가 깨진다.** common이 order를 의존하면 order가 common을 의존하므로 순환. Gradle이 막아준다.

→ outbox는 항상 `aggregate_type`, `payload` 같은 **도메인 무관 표현**으로만 다룬다.

### 8.2 함정 2 — common이 너무 비대해짐

작은 인프라 코드들을 자꾸 common에 던지면 common이 거대 모듈이 되어 모듈 분리의 의미가 사라진다.

→ common은 **진짜 횡단 관심사만**. 두 도메인 이상이 쓰지 않는 건 도메인 모듈에 둔다.

### 8.3 함정 3 — common에 있는데 한 도메인만 씀

처음엔 "범용 같다"고 common에 뒀는데 결국 한 도메인만 쓰게 되는 경우. **그 도메인 모듈로 옮겨야 한다.**

신호: `git log`로 그 클래스의 변경 이력을 봤을 때, **항상 같은 도메인 PR과 함께** 변경된다면 횡단성이 거짓이었던 것.

### 8.4 함정 4 — 인프라 추상화가 너무 일찍 들어감

```java
// 헥사고날 Port를 처음부터 도입
public interface OrderEventPublisher { ... }
public interface PaymentEventPublisher { ... }
public interface NotificationEventPublisher { ... }
// 결국 다 비슷한 메서드 시그니처
```

도메인 1~2개일 때 헥사고날을 너무 일찍 들이면 **추상의 비용만 감당하고 효용은 없다**. **3개 이상의 구체 사례에서 공통점이 드러날 때 추상화**한다 — Rule of Three.

이 프로젝트가 헥사고날을 안 간 이유 중 하나.

---

## 9. 진화 시나리오

### 9.1 현재 (Phase 1~3) — 직접 Repository 의존

```java
// 학습 친화적, 메커니즘이 코드에 보임
class OrderService {
    OutboxRepository outboxRepo;
    void confirm(Long id) {
        outboxRepo.save(OutboxEvent.of(...));
    }
}
```

### 9.2 Phase 4 가상 — 헥사고날로 진화

이벤트 발행 메커니즘이 다양해지고 (Outbox / 직접 Kafka / SQS / GRPC 스트림 등) 도메인이 무관하게 만들고 싶어질 때.

```java
class OrderService {
    IntegrationEventPublisher publisher;     // Port
    void confirm(Long id) {
        publisher.publish(new OrderConfirmedEvent(id));
    }
}
```

### 9.3 Phase 5 가상 — 도메인 분리

`order`가 별도 서비스가 될 때:

| 컴포넌트 | 어디로 |
|---------|------|
| `Order`, `OrderRepository`, `OrderService` | order 서비스로 |
| `IntegrationEventPublisher` Port (있다면) | order 서비스로 (인터페이스만) |
| `OutboxAdapter`, `OutboxEvent`, `OutboxRepository` | order 서비스의 인프라 패키지로 (테이블도 분리) |
| `OutboxRelay` | 각 서비스가 자기 Relay |
| `KafkaTemplate` 사용은 그대로 | 외부 통신 표준 |

분리 시점에 `common.outbox`가 어색해진다. 그래서 **분리를 진지하게 고려한다면 처음부터 헥사고날**이 낫다는 주장이 가능하지만, 학습 비용을 감안하면 현재 선택도 정당하다.

---

## 10. 한 줄 요약

> **`OutboxRepository`가 `common`에 있는 이유는 "outbox는 모든 도메인을 가로지르는 인프라 패턴"이기 때문이다.** 단일 테이블·단일 Repository·단일 Relay가 운영 단순성을 만든다. 도메인 → 인프라 의존은 정상 방향이고, 더 엄격한 분리(헥사고날 Port/Adapter)는 운영 시스템에서 권장되지만 학습 프로젝트에선 메커니즘을 노출시키는 게 가르침이 된다.
> **빌드 그래프가 캡슐화 다이어그램이라는 사실**이 여기서도 작동한다 — `common`이 어떤 도메인 모듈도 의존하지 않는 한 횡단성은 안전하다.

---

## 11. 함께 보면 좋은 자료

- `docs/study/event-store-vs-simple-bus.md` — Store + Bus 분리 사상 (인프라 추상의 가치)
- `docs/study/gradle-dependency-scopes.md` — 의존 방향이 빌드 그래프로 강제되는 메커니즘
- `docs/study/event-driven-tools.md` — Axon Framework 같은 헥사고날 강제 도구
- `docs/phase-3-tdd.md` Step 6 — ArchUnit으로 모듈 경계를 자동 검증
- Robert C. Martin, *Clean Architecture* — 의존 방향의 일반 원칙
- Alistair Cockburn, *Hexagonal Architecture* — Port/Adapter 사상의 출처
- Vaughn Vernon, *Implementing Domain-Driven Design* — Bounded Context와 인프라 분리

---

## 12. 의사결정 가이드

```
새로운 컴포넌트(클래스, 인터페이스, 테이블 등)를 만들 때:

  이게 비즈니스 의미가 있는가?
  ├ Yes (예: Order, PaymentStatus, OrderConfirmedPayload)
  │   → 도메인 모듈
  │
  └ No, 메커니즘이다 (예: OutboxEvent, EventBus, MetricsRegistry)
      │
      여러 도메인이 이걸 쓸 것인가?
      ├ Yes → common 모듈
      ├ 하나만 쓴다 → 그 도메인 모듈
      └ 모르겠다 → 그 도메인 모듈에서 시작, 두 번째 사용처가 생길 때 common으로 끌어올림

도메인이 인프라 메커니즘을 직접 호출하는 게 거슬리는가?
  ├ 학습 / 작은 프로젝트 → 그대로 직접 의존
  ├ 운영 / 큰 프로젝트 → Port/Adapter로 추상화 (헥사고날)
  └ 여러 인프라 구현을 동시 지원 / 서비스 분리 임박 → 반드시 추상화
```

---

## 13. 이 프로젝트에서의 핵심 사례 표

| 컴포넌트 | 위치 | 이유 |
|---------|------|------|
| `Order`, `Payment`, `Notification` | 각 도메인 모듈 | 비즈니스 의미 |
| `OrderCreated`, `PaymentCompleted` | `common.event.contract` | 도메인 간 통신 계약 (양쪽이 봐야 함) |
| `OrderRepository`, `PaymentRepository` | 각 도메인 모듈 | 그 도메인의 데이터 액세스 |
| **`OutboxEvent`, `OutboxRepository`** | **`common.outbox`** | **횡단 인프라 — 모든 도메인이 같은 outbox 사용** |
| `EventBus`, `EventStore` | `common.event` | 횡단 인프라 |
| `TransactionalEventPublisher` | `common.event` | 횡단 메커니즘 |
| `OutboxRelay`, `OutboxCleaner` | `common.outbox` | 백그라운드 인프라 |
| `OutboxMetrics`, `DeadLetterSweeper` | `common.outbox` | 운영 인프라 |
| `KafkaIntegrationTestBase` | `common.testFixtures` | 횡단 테스트 헬퍼 |

이 표를 한눈에 보면 **"도메인 vs 인프라 vs 컨트랙트"** 의 삼분이 코드 트리에 그대로 박혀 있음을 알 수 있다.
