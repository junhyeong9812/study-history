# 아키텍처 분석 — Hybrid Event-Driven Architecture

> 완성된 시스템에 대한 **분석 문서**.
> README가 "어떻게 쓰는가"를 다룬다면, 이 문서는 "왜 이렇게 만들었는가, 결과는 무엇인가"를 다룬다.

---

## 0. 분석의 관점

이 시스템을 다섯 축으로 분석한다.

| 축 | 질문 |
|----|------|
| **사상** | 무슨 문제를 어떤 사상으로 풀었는가? |
| **구조** | 모듈·계층·경계가 어떻게 짜여있는가? |
| **행동** | 런타임에 데이터/이벤트가 어떻게 흐르는가? |
| **결과 속성** | 일관성·가용성·관측성·확장성 면에서 어디에 위치하는가? |
| **진화** | 다음 단계로 어떻게 갈 수 있는가? |

---

## 1. 사상 분석

### 1.1 출발 문제

> "도메인이 작을 때부터 마이크로서비스로 가면 운영 부담이 너무 크고, 모놀리스로만 두면 도메인이 커졌을 때 분리가 불가능하다."

이 사이의 길을 찾는 게 출발점이었다. 결론:

```
[현재]                                    [미래]
모듈러 모놀리스                           ─진화→  부분/전체 마이크로서비스
단일 JVM, 코드 경계 분리                          (변경 비용을 미리 결제)
```

### 1.2 두 종류의 통신

코드는 두 종류의 통신을 **사상적으로 분리**한다.

| 종류 | 위치 | 메커니즘 | 일관성 |
|------|------|---------|------|
| 내부 (intra-process) | Order ↔ Payment | InMemoryEventBus + EventStore | 강한 일관성 (트랜잭션 경계) |
| 외부 (inter-process) | Order → Notification → 외부 | Outbox + Kafka + Inbox | effectively-exactly-once |

**왜 분리가 중요한가**: 둘을 같은 메커니즘으로 다루면 모든 통신을 가장 무거운 보장 수준에 맞춰야 한다. 내부 통신을 Kafka로 하면 단순 메서드 호출에 디스크 fsync가 끼어든다. 분리하면 각 자리에 정확한 도구를 놓을 수 있다.

### 1.3 Outbox 패턴의 자리

> "DB 저장과 메시지 큐 발행을 어떻게 같은 단위로 묶는가" — 이중 쓰기 함정.

```
[잘못된 방식 — 이중 쓰기]
1. order DB에 저장 ✅
2. Kafka 발행 ❌ (네트워크 단절)
→ 데이터 정합성 깨짐, 사용자는 주문했는데 알림 안 받음.

[Outbox 패턴]
하나의 DB 트랜잭션:
  ├─ order 도메인 저장
  └─ outbox 테이블에 메시지 INSERT
별도 프로세스(Relay):
  outbox 폴링 → Kafka 발행 → 상태 변경
```

이 사상이 "이벤트 = 사건의 기록"이라는 사고와 맞물린다. EventStore가 사건의 기록인 것처럼, outbox 테이블도 사건의 기록의 일종이다 — 둘 다 append-only.

### 1.4 핵심 사상 카드

| 카드 | 위치 | 의미 |
|----|------|------|
| **빌드 그래프 = 캡슐화 다이어그램** | settings.gradle, build.gradle | 빌드 도구가 모듈 경계를 강제 |
| **이벤트 = 사건의 기록** | EventStore, OutboxEvent | 메시지 전달이 아니라 역사 보존 |
| **컨트랙트 분리** | common.event.contract | 도메인끼리 직접 의존 금지 |
| **트랜잭션 경계 = 일관성 단위** | TransactionalEventPublisher, Outbox INSERT | DB 트랜잭션이 진실의 원천 |
| **at-least-once + 멱등성 = exactly-once** | Outbox + Inbox UNIQUE | 분산에서 흔한 결합 |
| **fail-fast + 격리** | RuntimeException wrap, Circuit Breaker | 예외는 위로 전파, 한 실패가 전체를 막지 않음 |
| **운영 가시화** | Metrics + DLQ + admin 엔드포인트 | 시스템 상태를 외부에서 측정 가능 |

---

## 2. 구조 분석

### 2.1 모듈 카탈로그 (다시)

```
common  (인프라 + 컨트랙트)
  ├─ event/                  EventBus, EventStore, TxPublisher
  ├─ event/contract/         OrderCreated, PaymentCompleted, ...
  ├─ outbox/                 OutboxEvent, OutboxRelay, OutboxWriter,
  │                          OutboxMetrics, OutboxCleaner, DLQ
  ├─ db/migration/           V1, V2, V4
  └─ testFixtures/           Kafka{Stub,TestConsumer,TestProducer},
                             KafkaIntegrationTestBase

order, payment, notification (도메인)
  ├─ domain/                 Aggregate + Repository + Status enum
  ├─ service/                UseCase
  ├─ handler/                EventBus 구독자
  └─ web/                    REST 컨트롤러

notification만 추가:
  ├─ inbox/                  멱등성 처리
  ├─ kafka/                  @KafkaListener 외부 통신 종단
  └─ NotificationApplication 단독 실행 진입점

app  (조립 + 운영)
  ├─ HybridEventDrivenApplication  모놀리스 진입점
  ├─ admin/                  운영 HTTP (DLQ 재시도)
  ├─ recovery/               cross-domain orchestration (OrderRecoveryJob)
  └─ integration/            E2E 테스트 + Chaos
```

### 2.2 의존 그래프의 본질

```
의존이 갈 수 있는 방향
────────────────────────────
common ◄─── order      ◄─── app
       ◄─── payment    ◄─── app
       ◄─── notification ◄─── app

도메인끼리 가로로 연결되는 화살표는 없음 (compile-time 강제).
```

이 그래프가 다음을 보장한다:

- common을 분리해도 도메인은 영향 없음 (common API가 stable인 한).
- 한 도메인을 떼어내도 다른 도메인은 컴파일됨 (contract만 공유).
- app만 운영 책임 — 도메인 코드를 운영 변화에 연동시키지 않음.

### 2.3 파일 단위 책임

| 파일 | 한 줄 책임 |
|----|---------|
| `EventStore` (interface) | append-only + offset read 계약 |
| `InMemoryEventStore` | CopyOnWriteArrayList + AtomicLong 구현 |
| `EventBus` (interface) | publish + subscribe 계약 |
| `InMemoryEventBus` | EventStore에 append 후 핸들러 디스패치 (예외 격리 포함) |
| `TransactionalEventPublisher` | TransactionSynchronization으로 afterCommit 예약 |
| `OutboxEvent` | outbox 테이블 JPA 매핑 |
| `OutboxRepository` | findTop100..., countByStatus, lockPending(SKIP LOCKED) |
| `OutboxWriter` | JSON 직렬화 + 예외 unchecked wrap |
| `OutboxRelay` | @Scheduled 폴링 → Kafka 발행 → 상태 갱신 |
| `OutboxMetrics` | Gauge로 PENDING / DEAD_LETTER 노출 |
| `OutboxCleaner` | 7일+ PUBLISHED 행 일괄 DELETE (cron) |
| `DeadLetterSweeper` | DEAD_LETTER outbox → outbox_dlq 이관 |
| `InboxEvent` | inbox 테이블 (UNIQUE messageId) |
| `InboxConsumer` | 멱등성 체크 + @Transactional + 카운터 |
| `NotificationKafkaListener` | @KafkaListener → InboxConsumer 위임 |
| `NotificationService` | EmailSender + PushSender 조립 |
| `EmailGateway` (interface) | 외부 메일 시스템 호출 추상 |
| `LoggingEmailGateway` | 운영 기본 구현 (로그) |
| `EmailSender` | @CircuitBreaker + fallback → DeferredEmailQueue |
| `DeferredEmailQueue` | OPEN 동안 보류, @Scheduled drain |
| `OrderRecoveryJob` (app/recovery) | DB 상태 기반 stuck 주문 복구 |
| `DeadLetterAdminController` (app/admin) | DLQ 수동 재시도 |
| `HybridEventDrivenApplication` | 모놀리스 main + @EnableScheduling |

각 파일이 **하나의 책임**을 갖는다 — 단일 책임 원칙(SRP)이 그대로 모듈 → 패키지 → 파일로 내려간다.

---

## 3. 행동 분석

### 3.1 정상 흐름 — 주문 한 건의 일생

이미 README와 `docs/study/architecture-overview.md`에 시각화. 핵심만:

```
HTTP POST → OrderService.create
  → orderRepo.save + 인메모리 OrderCreated 발행 (afterCommit 예약)
  → 트랜잭션 커밋
    → afterCommit: InMemoryEventBus.publish
       → PaymentEventHandler.onOrderCreated
         → PaymentService.process
           → paymentRepo.save + 인메모리 PaymentCompleted 발행
           → 트랜잭션 커밋
             → afterCommit: InMemoryEventBus.publish
                → OrderEventHandler.onPaymentCompleted
                  → OrderService.confirm
                    → order.confirm + outbox INSERT (OrderConfirmed)
                    → 트랜잭션 커밋
                      → OutboxRelay 다음 폴링에서 fetch
                        → Kafka send (order-events 토픽)
                          → outbox.markPublished
                          → NotificationKafkaListener.onMessage
                            → InboxConsumer.consume
                              → inbox INSERT + NotificationService.process
                                → EmailSender.send (CircuitBreaker)
                                → PushSender.send
```

각 단계가 **자기 트랜잭션 경계** 안에서 일관성을 보장하고, 단계 사이는 **이벤트가 다리** 역할을 한다.

### 3.2 실패 시나리오 — 정직성

| 실패 위치 | 결과 |
|---------|------|
| 주문 생성 트랜잭션 롤백 | order, outbox, 인메모리 디스패치 모두 폐기 (afterCommit이 호출 안 됨) |
| 결제 처리 중 예외 | Payment 트랜잭션 롤백 → PaymentCompleted 발행 안 됨 → 주문은 PAYMENT_PENDING 상태로 남음 → 60초 후 OrderRecoveryJob이 발견하지만 Payment가 COMPLETED 아니므로 그냥 둠 (정확) |
| 결제는 성공했으나 OrderEventHandler 직전 JVM 크래시 | 재시작 후 OrderRecoveryJob이 PAYMENT_PENDING + COMPLETED Payment 조합 발견 → orderService.confirm 호출 → 정상 회복 |
| Kafka가 일시 다운 | OutboxRelay의 send가 실패 → retryCount++ → 다음 폴링에서 재시도 |
| Kafka가 영구 다운 (10회 실패) | outbox.status = DEAD_LETTER → DeadLetterSweeper가 outbox_dlq로 이관 → 운영자가 admin 엔드포인트로 재시도 |
| 같은 Kafka 메시지 두 번 도착 | InboxConsumer.existsByMessageId가 중복 감지 → inbox.duplicate.count++ + return |
| 외부 메일 시스템 50% 실패율 | Resilience4j Circuit Breaker가 OPEN → fallback → DeferredEmailQueue에 보류 → 회복 후 자동 재발송 |

각 실패 시나리오에 **명시적 처리 경로**가 있다. 어디로 갈지 모르는 메시지는 없다.

---

## 4. 결과 속성 분석

### 4.1 일관성 모델

```
┌─────────────────────────────────────────────────────────────┐
│  Order DB write + Outbox INSERT + 인메모리 디스패치 예약      │
│  (모두 같은 @Transactional)                                  │
│  → 강한 일관성                                               │
└─────────────────────────────────────────────────────────────┘
              │ 트랜잭션 커밋
              ▼
┌─────────────────────────────────────────────────────────────┐
│  Outbox 행 PENDING                                           │
│  → DB 트랜잭션이 진실의 원천 (영속, 재시도 보장)              │
└─────────────────────────────────────────────────────────────┘
              │ Relay가 비동기로 fetch + Kafka 발행
              ▼
┌─────────────────────────────────────────────────────────────┐
│  Kafka 토픽 도착                                             │
│  → at-least-once (재시도 시 중복 가능)                       │
└─────────────────────────────────────────────────────────────┘
              │ NotificationKafkaListener 수신
              ▼
┌─────────────────────────────────────────────────────────────┐
│  InboxConsumer가 messageId 체크                              │
│  → 중복 차단 → effectively-exactly-once                      │
└─────────────────────────────────────────────────────────────┘
```

CAP 정리에 비추면: **단일 노드 / 단일 DB 모놀리스**라 P(분할) 자체가 거의 없고, C와 A는 강한 일관성과 가용성 둘 다 추구한다. Kafka 단계에선 P를 인정하면서 **A를 우선** (outbox에 쌓이고 retry).

### 4.2 가용성

- **읽기 가용성**: 도메인 API는 자기 DB만 조회하므로 Kafka가 죽어도 읽기는 정상.
- **쓰기 가용성**: 주문 생성·확정은 자기 DB만 쓰므로 Kafka가 죽어도 가능. outbox에 쌓이고 나중에 발행.
- **외부 통신 가용성**: Kafka 다운 → outbox 백압 → Kafka 복구 후 자동 따라잡기 (Phase 2 KafkaFailureRecoveryTest 증명).

### 4.3 관측성

| 무엇 | 어떻게 |
|------|------|
| 시스템 상태 | Prometheus + Grafana (메트릭 9종 + JVM/HTTP/Kafka 자동) |
| 단일 메시지 추적 | outbox.id 그대로 messageId로 → Kafka header → inbox.message_id로 보존 |
| 운영 개입 | `/admin/dlq/{id}/retry`, `flywayInfo`, `flyway_schema_history` |
| 빌드 시점 검증 | ArchUnit으로 모듈 경계, JPA validate로 스키마 일치 |

### 4.4 확장성

- **수직**: outbox 폴링 + DB 인덱스(부분 인덱스) → 단일 인스턴스로도 수만 TPS 처리 가능.
- **수평**: `FOR UPDATE SKIP LOCKED`로 멀티 인스턴스 Relay → 노드 추가 시 자연 분산.
- **분리 가능성**: Notification → 단독 실행 가능 증명. 같은 패턴으로 Order/Payment 분리도 통신 계층 교체만으로 가능.

### 4.5 테스트 가능성

- 모듈별 단위 테스트 + 모듈 통합 테스트 + E2E + Chaos.
- Testcontainers로 운영과 동일 인프라 검증.
- Stub 기반 결정적 chaos (KafkaProducerStub, EmailSenderStub).
- ArchUnit으로 빌드 시점 모듈 경계 검증.

---

## 5. 약점·트레이드오프

정직하게 짚는 한계:

| 항목 | 한계 | 회피 / 진화 경로 |
|------|------|--------------|
| **Outbox 폴링** | 폴링 주기(1초)만큼 지연 | Phase 3+ Debezium CDC 도입 |
| **단일 DB** | Order/Payment가 같은 DB 공유 | 외래키 안 두고, 향후 분리 시 같은 사상의 outbox 추가 |
| **In-memory EventStore 재시작 시 휘발** | JVM 크래시 시 인메모리 큐 사라짐 | OrderRecoveryJob이 DB 상태로 복구 (대부분 충분) |
| **Kafka 트랜잭션 미사용** | producer 트랜잭션 안 씀 | Outbox로 같은 보장 → 중복 |
| **헥사고날 부분 적용** | 도메인이 OutboxRepository 직접 의존 | OutboxWriter로 한 단계 추상, 더 가려면 Port/Adapter |
| **Saga 미도입** | 보상 트랜잭션 패턴 없음 | 결제 환불 같은 시나리오 추가 시 도입 |

이 한계들은 **현재 학습 단계의 의도된 단순화**다. 운영 시스템에선 각 항목에 대해 명시적 결정이 필요.

---

## 6. 진화 로드맵

### 6.1 단기 (모놀리스 안에서)

- ArchUnit 룰 강화 → CI에서 모듈 경계 자동 검증
- Saga 패턴 도입 → 환불·취소·재시도 흐름
- `OutboxWriter` 외에 도메인 ↔ 인프라 추상화 (`IntegrationEventPublisher` Port)
- 분산 추적 (OpenTelemetry) → trace ID가 outbox/inbox/Kafka 헤더 따라 흐름

### 6.2 중기 (부분 분리)

- Notification → 별도 서비스 (자기 DB, 자기 Kafka 컨슈머 그룹)
- 분리 시점에 모놀리스가 가진 contract 패키지가 그대로 다른 서비스의 의존이 됨

### 6.3 장기 (전체 MSA)

- Order/Payment 분리 → 인메모리 EventBus 통신을 Outbox+Kafka로 교체
- 통신 패턴이 이미 익숙하므로 코드 변경은 최소 (인터페이스 그대로)

---

## 7. 학습 가치

이 프로젝트가 가르치는 것을 한 단어씩 묶으면:

```
모듈러 모놀리스 · 이벤트 드리븐 · CQRS의 일부 · 멱등성 · at-least-once
강한 일관성 · 최종 일관성 · WAL · 부분 인덱스 · SKIP LOCKED · 트랜잭션 동기화
Outbox · Inbox · Circuit Breaker · DLQ · Recovery Job
TDD · Testcontainers · ArchUnit · testFixtures · Chaos
Spring Boot 4 모듈화 · Resilience4j · Micrometer · Prometheus · Flyway
```

각 단어가 코드의 어느 자리에 살아있는지를 학습 노트(`docs/study/`)와 페이즈 문서(`docs/phase-*-tdd.md`)가 매핑한다.

---

## 8. 결론

> **이 시스템은 "단일 JVM의 효율 + 분산 시스템의 견고함"을 양립시키는 패턴 모음의 살아있는 카탈로그다.**

- 사상은 명확하다: **내부는 가볍게, 외부는 견고하게.**
- 구조는 정직하다: **빌드 그래프가 곧 캡슐화 다이어그램.**
- 행동은 추적 가능하다: **모든 메시지가 사건의 기록으로 남음.**
- 진화는 점진적이다: **현재 구조 위에서 모든 다음 단계가 가능.**

설계 문서, 페이즈 TDD 문서, 19개 학습 노트가 함께 이 시스템의 **사상과 손기술을 모두 보여준다.**

---

## 9. 함께 보면 좋은 자료

- [`README.md`](../README.md) — 빠른 시작과 운영
- [`docs/study/architecture-overview.md`](study/architecture-overview.md) — 모든 학습 노트의 색인
- [`docs/phase-{1,2,3}-tdd.md`](.) — 단계별 TDD 구현
- [`docs/study/module-placement-rules.md`](study/module-placement-rules.md) — 모듈 경계 결정 규칙
- [`docs/study/event-store-vs-simple-bus.md`](study/event-store-vs-simple-bus.md) — Store + Bus 분리 사상
- [`docs/study/cross-cutting-infrastructure.md`](study/cross-cutting-infrastructure.md) — 횡단 인프라 위치
