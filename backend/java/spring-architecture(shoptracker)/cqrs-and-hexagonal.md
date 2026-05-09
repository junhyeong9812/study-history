# CQRS + Hexagonal Architecture

> 이 프로젝트의 골격을 이루는 두 가지 아키텍처 패턴.

## Hexagonal (Ports & Adapters)

**Alistair Cockburn (2005)**. 도메인을 외부 기술에서 격리하는 패턴.

### 구조

```
        ┌──────────────────────────────────┐
        │           Adapters (외부)         │
        │  ┌────────┐         ┌──────────┐ │
        │  │ Web/   │         │  JPA/    │ │
        │  │ REST   │         │  Kafka/  │ │
        │  │        │         │  Cache   │ │
        │  └───┬────┘         └────▲─────┘ │
        │      │                   │       │
        │  ────│───────────────────│────── │
        │      ▼                   │       │
        │  ┌────────┐         ┌────┴─────┐ │
        │  │ Input  │         │  Output  │ │
        │  │ Port   │         │  Port    │ │
        │  └───┬────┘         └────▲─────┘ │
        │      │                   │       │
        │      ▼                   │       │
        │  ┌──────────────────────────┐    │
        │  │      Application         │    │
        │  │      (UseCase)           │    │
        │  └──────────┬───────────────┘    │
        │             ▼                    │
        │  ┌──────────────────────────┐    │
        │  │      Domain (순수)        │    │
        │  └──────────────────────────┘    │
        │                                  │
        └──────────────────────────────────┘
```

- **Domain**: 비즈니스 규칙. 어떤 프레임워크/DB/HTTP에도 의존하지 않음.
- **Application (UseCase)**: 도메인을 조립해서 시나리오 수행.
- **Input Port**: UseCase 인터페이스 (예: `CreateOrderUseCase`).
- **Output Port**: 도메인이 외부에 요구하는 것 (예: `OrderRepository`, `PaymentGateway`).
- **Adapter**: 포트 구현체. Web 어댑터(컨트롤러), Persistence 어댑터(JPA).

### 의존성 방향

**모든 화살표는 도메인을 향한다.** 도메인은 어댑터를 모르고, 어댑터가 도메인의 인터페이스를 구현. → 의존성 역전(DIP).

### 이 프로젝트 매핑

```
orders/
├── domain/
│   ├── model/        ← Order, OrderItem, OrderStatus, Money (순수 Java)
│   ├── port/outbound/ ← OrderRepository (Output Port)
│   └── exception/
├── application/
│   ├── port/inbound/ ← CreateOrderUseCase, CancelOrderUseCase (Input Port)
│   ├── service/      ← OrderCommandService, OrderQueryService (UseCase 구현)
│   └── eventhandler/ ← OrderEventHandler
└── adapter/
    ├── inbound/web/  ← OrderController, CreateOrderRequest (HTTP 어댑터)
    └── outbound/persistence/  ← OrderJpaEntity, OrderMapper, OrderPersistenceAdapter
```

### 효과

1. **단위 테스트 빠름**: 도메인은 DB/HTTP 없이 `new`로 생성 가능 → 0.x초 테스트.
2. **인프라 교체 가능**: JPA → MongoDB, REST → gRPC 바꿔도 도메인 그대로.
3. **비즈니스 규칙이 코드 가운데에**: 어댑터가 두꺼우면 도메인이 얇아져 분산.

## CQRS (Command Query Responsibility Segregation)

**Greg Young (2010)**. **상태를 바꾸는 명령(Command)** 과 **상태를 읽는 조회(Query)** 를 분리.

### 왜?

- 명령과 조회의 요구사항이 다르다:
  - 명령: 비즈니스 규칙, 검증, 트랜잭션, 이벤트 발행
  - 조회: 페이징, 필터링, 검색 최적화, 캐싱
- 한 모델에 다 담으면 거대해짐. 분리하면 각자 최적화 가능.

### 이 프로젝트의 적용

```
                ┌─────────────────────┐
                │  OrderController    │
                └──────┬──────┬───────┘
                       │      │
          POST/cancel  │      │  GET / GET by id
                       ▼      ▼
       ┌──────────────────┐ ┌──────────────────┐
       │ OrderCommand     │ │ OrderQuery       │
       │ Service          │ │ Service          │
       │                  │ │                  │
       │ @Transactional   │ │ @Transactional   │
       │                  │ │ (readOnly=true)  │
       │ EventPublisher   │ │ NO publisher     │
       │ + Repository     │ │ + Repository     │
       │   (writes)       │ │   (reads only)   │
       └──────────────────┘ └──────────────────┘
              ▼                      ▼
     ┌────────────┐         ┌────────────────┐
     │ Domain     │         │  ReadModel     │
     │ (Order)    │         │  (OrderSummary) │
     └────────────┘         └────────────────┘
```

- **Command 측**: 도메인 모델 사용, 트랜잭션 ON, 이벤트 발행.
- **Query 측**: ReadModel(`OrderSummary`) 사용, `readOnly=true`, 이벤트 발행 없음.

### ReadModel(DTO Projection)

```java
// orders/application/query/OrderSummary.java
public record OrderSummary(
    UUID id, String customerName, String status,
    BigDecimal totalAmount, BigDecimal shippingFee,
    BigDecimal discountAmount, BigDecimal finalAmount,
    int itemCount, Instant createdAt
) {}
```

- Aggregate(Order) 전체가 아닌 **목록 화면이 필요한 필드만** 노출.
- 테이블 구조와 1:1이 아니어도 됨 (조인 결과를 그대로 매핑 가능).

### 더 나아간 CQRS

이 프로젝트는 **간단한 CQRS**: 같은 DB에 같은 테이블, 서비스만 분리.

진짜 CQRS는:
- Command와 Query가 **별도 데이터베이스** 사용
- Command가 이벤트 발행 → Query 측이 이벤트로 ReadModel 업데이트 (Eventual consistency)
- Query DB는 검색 엔진(Elasticsearch), 컬럼형 DB 등으로 최적화

이 프로젝트의 `Tracking` 모듈이 약하게 그 모양: 다른 모듈의 이벤트를 받아 별도 테이블(`order_tracking`, `tracking_events`) 에 ReadModel 형태로 저장.

## Hexagonal + CQRS의 시너지

- Hexagonal이 큰 골격(도메인 격리)
- CQRS가 application layer 안에서 책임 분리
- 둘 다 **인터페이스 기반** 이라 테스트/교체 쉬움

## 안티패턴 / 주의

1. **Mapper 폭발**: 도메인 ↔ JPA ↔ DTO 3단 매퍼. 작은 프로젝트에서는 과한 비용. 적정선이 중요.
2. **Repository에 비즈니스 로직 누수**: `findActiveByCustomer` 정도는 OK, `findCustomersWithOverdueOrdersFromLastMonth`는 도메인 서비스로.
3. **Read 모델 비대화**: 한 ReadModel에 모든 화면을 다 채워 넣으려는 욕심. 화면별로 나눠도 됨.
4. **CQRS = Event Sourcing 아님**: 같이 쓰이지만 별개 개념.

## FastAPI 버전과의 비교

이 프로젝트는 FastAPI Python 버전을 Spring으로 옮긴 학습 프로젝트. 두 버전 모두 동일한 Hexagonal + CQRS를 적용했지만:

| 구현 측면 | FastAPI | Spring |
|----------|---------|--------|
| 인터페이스 | `typing.Protocol` | `interface` |
| 어댑터 등록 | Dishka Provider | `@Component` + DI |
| 트랜잭션 | `async with session.begin()` | `@Transactional` |
| 이벤트 | InMemoryEventBus 직접 | ApplicationEventPublisher |
| 모듈 검증 | grep 수동 | Spring Modulith |

골격은 같지만 프레임워크가 보일러플레이트를 얼마나 흡수하느냐가 다르다.
