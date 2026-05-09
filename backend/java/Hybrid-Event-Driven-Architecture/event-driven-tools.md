# 이벤트 드리븐 시스템에서 자주 등장하는 도구들

> `event-store-vs-simple-bus.md`에서 언급된 도구들의 정체와 위치를 정리한 학습 노트.
> 어느 도구가 "단순 버스"이고 어느 도구가 "Store + Bus 분리"이며, 각각 언제 쓰는가.

---

## 0. 한눈에 보는 분류

```
[단순 버스 모델]                    [Store + Bus 모델]                [Event Sourcing 프레임워크]
이벤트 = 메시지 (휘발)               이벤트 = 사건의 기록 (영속)         이벤트 = 상태 그 자체

  Spring Application                  Apache Kafka                    Axon Framework
   EventPublisher                     RabbitMQ Streams                EventStoreDB
  Guava EventBus                      Redpanda                        Event Store / Snapshot
  RxJava Subject                      Pulsar                          CQRS 전체
  EventBus (이 프로젝트의              ↑                               ↑
   초기 인메모리 버스)                  이 프로젝트가 Phase 2에           이 프로젝트는 거기까지는
                                      서 도달하는 지점                  안 가지만 같은 사상
```

---

## 1. 단순 버스 모델 도구들

### 1.1 Spring `ApplicationEventPublisher`

Spring의 내장 이벤트 시스템.

```java
@Component
class OrderService {
    private final ApplicationEventPublisher publisher;

    public void create(...) {
        publisher.publishEvent(new OrderCreatedEvent(...));
    }
}

@Component
class PaymentListener {
    @EventListener
    public void on(OrderCreatedEvent event) { /* 처리 */ }
}
```

- **위치**: Spring Framework 내장.
- **모델**: 단순 버스. 이벤트는 `ApplicationContext` 내부에서만 흐르고 끝나면 사라진다.
- **장점**: 별도 의존성 0. 어노테이션만으로 끝. 트랜잭션 동기화(`@TransactionalEventListener`) 무료.
- **한계**: 같은 JVM 내부, 같은 Spring 컨텍스트 한정. 영속·재생 없음.

> **이 프로젝트와의 관계**: `TransactionalEventPublisher`가 트랜잭션 동기화 부분에서 비슷한 일을 한다. 다만 Spring `@EventListener`로 가지 않고 직접 Bus를 만든 이유는 **Phase 2에서 Kafka로 바꿔치기할 인터페이스를 명시적으로 두기 위해서**다.

### 1.2 Guava `EventBus`

Google Guava 라이브러리의 이벤트 버스.

```java
EventBus bus = new EventBus();
bus.register(new Object() {
    @Subscribe
    public void on(OrderCreated e) { /* 처리 */ }
});
bus.post(new OrderCreated(...));
```

- **위치**: Guava (`com.google.common.eventbus`).
- **모델**: 단순 버스. 동기형(`EventBus`) / 비동기형(`AsyncEventBus`) 두 종류.
- **장점**: 가볍고 단순. Spring 없이도 쓸 수 있음.
- **한계**: 영속·재생·순서 보장 없음. Guava 라이브러리 자체는 "Maintenance mode"라 신규 프로젝트에선 권장 안 됨.

### 1.3 RxJava `Subject` / Project Reactor `Sinks`

리액티브 스트림 기반의 이벤트 버스 변형.

```java
Sinks.Many<DomainEvent> sink = Sinks.many().multicast().onBackpressureBuffer();
sink.asFlux().subscribe(handler);
sink.tryEmitNext(new OrderCreated(...));
```

- **위치**: RxJava / Reactor.
- **모델**: 본질적으로 단순 버스. 다만 **백프레셔(backpressure)** 와 **함수형 합성**이 강점.
- **장점**: 스트림 처리 파이프라인 구성 자유. 타임아웃·디바운스·머지 같은 연산자 풍부.
- **한계**: 학습 곡선 가파름. 영속은 별도.

> **이런 도구를 쓰는 이유**: 이벤트 처리 자체가 복잡한 흐름(여러 스트림 합치기, 시간 윈도우 등)일 때.

---

## 2. Store + Bus 모델 — 메시지 브로커들

### 2.1 Apache Kafka

이 프로젝트 Phase 2의 핵심 도구.

```
┌──────────────────────────────────────────────────┐
│  Topic: order-events                             │
│  ┌──────┬──────┬──────┬──────┬──────┐           │
│  │ #0   │ #1   │ #2   │ #3   │ ...  │  partition │
│  └──────┴──────┴──────┴──────┴──────┘           │
│   ↑ append-only commit log (디스크 영속)          │
│                                                  │
│  Consumer Group A — offset 5에서 읽는 중          │
│  Consumer Group B — offset 12에서 읽는 중         │
└──────────────────────────────────────────────────┘
```

- **본질**: 분산 commit log. **Store + Bus가 분리된 모델의 결정판**.
- **특징**:
  - 메시지가 **디스크에 저장됨** (보존 기간 동안).
  - Consumer가 자기 offset을 들고 자기 속도로 진행.
  - 같은 메시지를 여러 컨슈머 그룹이 독립적으로 소비 가능.
  - Replay(과거 offset부터 다시 읽기) 자유.
- **언제**:
  - 마이크로서비스 간 비동기 통신.
  - 실시간 데이터 파이프라인.
  - 이벤트 소싱의 영속 백엔드.
- **이 프로젝트에서의 역할**: Phase 2의 Order → Notification 통신. Phase 1의 인메모리 EventStore와 **개념이 같은 추상의 분산 구현체**.

### 2.2 RabbitMQ Streams / Redpanda / Apache Pulsar

Kafka와 같은 commit log 모델을 따르는 다른 브로커들.

| 도구 | 특징 |
|------|------|
| **RabbitMQ Streams** (3.9+) | RabbitMQ에 추가된 Kafka 유사 모드. 기존 RMQ 사용처에서 부분 도입 가능 |
| **Redpanda** | Kafka 와이어 호환. C++로 구현돼 JVM 부담 없음. 운영 단순화 표방 |
| **Apache Pulsar** | 멀티 테넌시·지오 레플리케이션 강점. 메시지 큐 + 스트림 둘 다 지향 |

**이 프로젝트에서는 Kafka 선택** — 학습 자료가 가장 풍부하고, Spring Kafka 통합이 가장 안정적.

### 2.3 전통 메시지 큐 (RabbitMQ AMQP, ActiveMQ)와의 차이

```
[전통 큐]                          [Kafka류 commit log]
메시지가 컨슈머에게 가면 사라짐       메시지는 보존 기간 내내 남음
큐 = 라우팅 + 일시 보관              토픽 = 영구(보존 기간) 로그
컨슈머 늘려서 분산 처리              컨슈머 그룹 / 파티션으로 분산
"메시지를 한 번 소비"               "로그를 어디부터 읽을지 선택"
```

전통 큐는 **단순 버스의 분산 버전**에 가깝고, Kafka는 **Store + Bus의 분산 버전**이다. 본 프로젝트의 인메모리 EventStore가 Kafka와 자연스럽게 매핑되는 이유다.

---

## 3. Event Sourcing 전용 도구

### 3.1 EventStoreDB

Greg Young(Event Sourcing 개념을 정리한 사람) 진영에서 만든 **이벤트 소싱 전용 DB**.

```
[Stream: order-42]
  event #0  OrderCreated   { customerId: 1, amount: 1000 }
  event #1  PaymentCompleted { ... }
  event #2  OrderConfirmed   { ... }

→ Stream을 처음부터 재생하면 현재 상태가 도출됨
```

- **본질**: 애그리거트(예: `Order` 인스턴스) 단위의 **이벤트 스트림**을 저장하는 DB.
- **특징**:
  - 스트림 = 한 애그리거트의 일생.
  - 상태가 아니라 사건만 저장 → 어떤 시점의 상태든 재생으로 복원 가능.
  - Projections(특정 뷰로 변환)·Subscription(스트림 구독)·Snapshot(중간 캐시) 내장.
- **Kafka와의 차이**: Kafka는 토픽 단위 로그(메시징 중심), EventStoreDB는 스트림 단위 저장(애그리거트 중심). 같은 commit log 사상이지만 모델링 단위가 다름.

> **이 프로젝트에서는 안 씀** — 학습 목표가 "메시징 패턴"이지 "이벤트 소싱 자체"가 아니기 때문. 다만 인메모리 EventStore의 사상은 이 도구와 결이 같다.

### 3.2 Axon Framework

자바/스프링 기반 **CQRS + Event Sourcing 프레임워크**.

```java
@Aggregate
class Order {
    @AggregateIdentifier private String id;
    private OrderStatus status;

    @CommandHandler
    Order(CreateOrderCommand cmd) {
        AggregateLifecycle.apply(new OrderCreated(cmd.id(), cmd.amount()));
    }

    @EventSourcingHandler
    void on(OrderCreated event) { this.id = event.id(); this.status = CREATED; }
}
```

- **제공 기능**:
  - Command Bus / Event Bus / Query Bus (CQRS의 세 축).
  - Aggregate에 대한 자동 이벤트 적용.
  - 이벤트 저장(Axon Server 또는 RDB / MongoDB 백엔드).
  - Saga 지원.
- **장단점**:
  - 장점: CQRS+ES를 처음부터 끝까지 일관되게 강제. 도메인 모델이 깔끔해짐.
  - 단점: 프레임워크 종속이 큼. 한번 들어가면 빠져나오기 어려움. 학습 곡선 가파름.

> **이 프로젝트에서는 안 씀** — Axon은 "Event Sourcing 풀세트"이고, 이 프로젝트는 "Event Bus + Outbox/Inbox 학습"에 한정한다. 다만 Phase 1의 `Order.create` → `OrderCreated` 발행 패턴은 Axon의 `AggregateLifecycle.apply`와 사상이 닿는다.

---

## 4. 비교 매트릭스

| 도구 | 모델 | 영속 | Replay | 분산 | CQRS/ES 강제 | 학습 곡선 |
|------|------|------|--------|------|-------------|---------|
| Spring `ApplicationEventPublisher` | 단순 버스 | ❌ | ❌ | ❌ (단일 컨텍스트) | ❌ | 매우 낮음 |
| Guava `EventBus` | 단순 버스 | ❌ | ❌ | ❌ | ❌ | 매우 낮음 |
| RxJava `Subject` / Reactor `Sinks` | 단순 버스 (스트림형) | ❌ | ❌ | ❌ | ❌ | 높음 |
| RabbitMQ (전통 큐) | 분산 단순 버스 | △ (잠시) | ❌ | ✅ | ❌ | 중간 |
| **Apache Kafka** | **Store + Bus** | ✅ (보존기간) | ✅ | ✅ | ❌ (직접 만들 수 있음) | 높음 |
| RabbitMQ Streams / Redpanda / Pulsar | Store + Bus (Kafka 계열) | ✅ | ✅ | ✅ | ❌ | 높음 |
| EventStoreDB | Event Sourcing DB | ✅ | ✅ | ✅ | ✅ | 높음 |
| Axon Framework | CQRS+ES 프레임워크 | ✅ (백엔드 위임) | ✅ | ✅ | ✅ | 매우 높음 |
| **이 프로젝트의 인메모리 EventStore** | **Store + Bus 축소판** | △ (메모리) | ✅ | ❌ | ❌ | 낮음 (학습용) |

---

## 5. 선택 가이드

```
필요한 게 단지 "같은 모듈 안 통지" 인가?
   ├─ Yes → Spring ApplicationEventPublisher
   └─ No
       │
       복잡한 스트림 합성/시간 윈도우가 필요한가?
       ├─ Yes → RxJava / Project Reactor
       └─ No
           │
           프로세스 경계를 넘어야 하는가?
           ├─ No  → 인메모리 Store + Bus (이 프로젝트의 Phase 1)
           ├─ Yes
           │
           이벤트의 "역사"가 비즈니스 가치인가?
           ├─ No  (메시지 신뢰 전달이 목표) → Kafka + Outbox/Inbox (Phase 2)
           ├─ Yes (애그리거트 일생을 저장하고 싶다) → EventStoreDB
           │
           CQRS 전체 + Saga까지 강제하고 싶은가?
           └─ Yes → Axon Framework
```

---

## 6. 이 프로젝트에서 직접 다루는 도구

| 단계 | 도구 | 역할 |
|------|------|------|
| Phase 1 | 자체 인메모리 EventStore + EventBus | "Store + Bus 분리" 사상의 작은 손모형 |
| Phase 1 | `TransactionalEventPublisher` (Spring TransactionSynchronization) | 커밋 후 디스패치 / 롤백 시 폐기 |
| Phase 2 | Apache Kafka (Spring Kafka) | 외부 시스템과의 비동기 통신 |
| Phase 2 | Outbox/Inbox 테이블 (PostgreSQL + Flyway) | 신뢰성·멱등성 보장 |
| Phase 2 | Testcontainers (Postgres, Kafka) | 통합 테스트 |
| Phase 3 | Resilience4j Circuit Breaker | 외부 시스템 장애 격리 |
| Phase 3 | Micrometer + Prometheus | 관측성 |
| Phase 3 | ArchUnit | 모듈 경계 검증 |

EventStoreDB·Axon·Pulsar 같은 도구는 **언급은 하되 도입하지 않는다.** 학습 목표 범위를 벗어나기 때문.

---

## 7. 한 줄 요약

| 도구 | 한 줄 |
|------|------|
| Spring `ApplicationEventPublisher` | 같은 컨텍스트 안의 가벼운 통지. 무료, 즉시 사용 |
| Guava `EventBus` | Spring 없는 환경의 가벼운 동기/비동기 버스. 신규엔 비추천 |
| RxJava / Reactor | 이벤트가 "스트림"이고 합성이 복잡할 때 |
| RabbitMQ (전통) | 신뢰성 있는 분산 메시지 큐. "한 번 소비하면 사라지는" 모델 |
| **Kafka** | **분산 commit log. 이 프로젝트 Phase 2의 도구** |
| EventStoreDB | 애그리거트 단위 이벤트 스트림 DB. ES 풀세트 |
| Axon Framework | CQRS+ES를 강제하는 자바 프레임워크. 종속성 높음 |
| **이 프로젝트 EventStore** | **Kafka의 사상을 인메모리로 축소한 학습용 손모형** |

---

## 8. 함께 보면 좋은 자료

- `docs/study/concurrent-collections.md` — 인메모리 Store/Bus가 쓰는 동시성 자료구조
- `docs/study/event-store-vs-simple-bus.md` — 단순 버스 vs Store+Bus 트레이드오프
- Jay Kreps, *The Log: What every software engineer should know* (Kafka의 사상적 출발점)
- Greg Young, *CQRS Documents* (Event Sourcing 정리)
- Martin Fowler, *What do you mean by "Event-Driven"?* — 이 다양한 모델들이 다 "이벤트 드리븐"으로 뭉뚱그려지는 혼란을 정리
