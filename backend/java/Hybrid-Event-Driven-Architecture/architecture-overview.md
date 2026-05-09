# Hybrid Event-Driven Architecture — 설계 개념 종합

> 이 프로젝트가 사용한 **모든 사상적 도구**를 한 문서에 모은 종합 학습 노트.
> 다른 study 노트들이 각 개념을 깊게 다룬다면, 이 문서는 **그 개념들이 어떻게 서로 짜여 있는가**를 보여준다.

---

## 0. 한 문장 요약

> **"내부 도메인끼리는 인메모리 이벤트 버스로 가볍게, 외부 시스템과는 Outbox/Inbox 패턴 + Kafka로 견고하게"** — 단일 JVM 모듈러 모놀리스를 운영 안정성과 마이크로서비스 전환 가능성을 모두 갖추고 만든다.

---

## 1. 사상적 출발점 — 왜 하이브리드인가

### 1.1 두 극단의 함정

| 극단 | 함정 |
|------|------|
| 처음부터 마이크로서비스 | MSA 운영 부담을 도메인이 작을 때부터 감당 — 분산 디버깅, 네트워크 지연, 운영 도구 폭발 |
| 영원한 거대 모놀리스 | 도메인이 커져도 분리 불가능, 코드와 데이터 결합 폭증 |

이 프로젝트는 **모듈러 모놀리스(Modular Monolith)** 로 가운데를 잡는다 — 코드는 모듈로 분리, 실행은 단일 JVM. 그리고 **언제든 마이크로서비스로 분리할 수 있게** 통신 추상을 처음부터 둔다.

### 1.2 통신의 두 종류

```
[내부 통신]                          [외부 통신]
같은 JVM, 같은 트랜잭션              다른 시스템, 다른 트랜잭션
└─ 인메모리 이벤트 버스                └─ Kafka + Outbox/Inbox
   (강한 일관성)                          (effectively-exactly-once)

Order ↔ Payment                     Order → Notification
                                    Order → 미래의 외부 서비스
```

---

## 2. 사용한 패턴들 — 한눈에 정리

| 패턴 | 어디서 | 풀어주는 문제 |
|------|------|------------|
| **모듈러 모놀리스** | 전체 빌드 그래프 | "분리는 코드 경계로 먼저, 프로세스 경계는 나중에" |
| **이벤트 드리븐 아키텍처** | EventBus, contract 패키지 | 도메인 결합 최소화, 비동기 진행 |
| **Event Log / Commit Log** | EventStore | 메시지가 아니라 사건의 기록으로 다룸 |
| **Outbox 패턴** | OutboxEvent + OutboxRelay | DB와 메시지 큐 사이 이중 쓰기 함정 |
| **Inbox 패턴** | InboxEvent + InboxConsumer | at-least-once 메시지를 멱등 처리 |
| **CQRS의 일부** | 도메인은 명령, 통합은 이벤트 | 읽기·쓰기·통합 책임 분리 |
| **Saga (보상 트랜잭션) 준비** | TransactionalEventPublisher | 분산 트랜잭션 회피 |
| **Hexagonal 일부** | EmailGateway interface + Stub | 외부 시스템 추상화 |
| **Circuit Breaker** | EmailSender + Resilience4j | 외부 시스템 장애 격리 |
| **Bulkhead 일부** | DeferredEmailQueue | OPEN circuit 동안 메일 보류 |
| **Dead Letter Queue** | OutboxStatus.DEAD_LETTER + outbox_dlq | 영구 실패 메시지의 운영 가시화 |
| **Idempotency** | InboxRepository.message_id UNIQUE | 중복 수신 방어 |
| **WAL 영감** | InMemoryEventStore | append-only + offset 기반 read |
| **Skip Locked** | OutboxRepository.lockPending | 멀티 인스턴스 worker 안전성 |

---

## 3. 모듈 책임 매트릭스

```
┌────────────────────────────────────────────────────────────────────────┐
│                         실행 가능 / 진입점                              │
│                                                                        │
│  ┌──────────────────────────┐   ┌──────────────────────────────────┐  │
│  │  HybridEventDriven       │   │  NotificationApplication         │  │
│  │  Application (모놀리스)  │   │  (Phase 3 단독 실행)              │  │
│  └──────────┬───────────────┘   └─────────────┬────────────────────┘  │
│             │                                  │                       │
└─────────────┼──────────────────────────────────┼───────────────────────┘
              │                                  │
┌─────────────┼──────────────────────────────────┼───────────────────────┐
│             ▼   조립 / 운영 (app)              │                       │
│  com.hybrid.HybridEventDrivenApplication       │                       │
│  com.hybrid.admin.DeadLetterAdminController    │                       │
│  com.hybrid.recovery.OrderRecoveryJob          │                       │
└────┬────────┬────────┬────────┬────────────────┼───────────────────────┘
     │        │        │        │                │
     ▼        ▼        ▼        ▼                ▼
┌────────┬────────┬─────────┬─────────────────────────────┐
│ order  │payment │ noti..  │  common (인프라 + 컨트랙트)   │
│        │        │         │                              │
│ 도메인 │ 도메인 │ 도메인  │ event/         outbox/       │
│        │        │ + Inbox │  EventBus      OutboxEvent   │
│        │        │ + Kafka │  EventStore    OutboxRelay   │
│        │        │         │  Trans...      OutboxMetrics │
│        │        │         │  contract/...  OutboxCleaner │
│        │        │         │  ...           DLQ + Sweeper │
└────────┴────────┴─────────┴─────────────────────────────┘
   ↑       ↑        ↑          ↑
 implementation  project(':common')

(testFixtures of common: KafkaIntegrationTestBase, KafkaProducerStub,
 KafkaTestConsumer, KafkaTestProducer — 모든 모듈 테스트가 공유)
```

| 모듈 | 패키지 | 책임 |
|------|------|------|
| common | event, event.contract, outbox | 인프라 + 도메인 간 통신 컨트랙트 |
| order | domain, service, handler, web | Order 도메인 |
| payment | domain, service, handler, web | Payment 도메인 |
| notification | domain, inbox, kafka, service, web | Notification 도메인 + 외부 통신 종단 |
| app | (root), admin, recovery, integration | 조립 + 운영 + E2E 테스트 |

---

## 4. 데이터 흐름 — 한 주문의 일생

```
[HTTP]
   POST /api/orders
        │
        ▼
[order]
   OrderController.create
        │
        ▼  @Transactional
   OrderService.create
   ├─ orderRepo.save(Order)           ← orders 테이블 INSERT
   └─ publisher.publish(OrderCreated) ← afterCommit에 예약
        │
        ▼ 트랜잭션 커밋 후
   InMemoryEventBus.publish
   ├─ EventStore.append (offset 부여)
   └─ 구독자 호출
        │
        ▼ (PaymentEventHandler 구독자 호출)
[payment]
   PaymentEventHandler.onOrderCreated
        │
        ▼  @Transactional
   PaymentService.process
   ├─ Payment 생성 + complete()
   ├─ paymentRepo.save                ← payments 테이블 INSERT
   └─ publisher.publish(PaymentCompleted)
        │
        ▼ 커밋 후
   InMemoryEventBus.publish(PaymentCompleted)
        │
        ▼ (OrderEventHandler 구독자 호출)
[order]
   OrderEventHandler.onPaymentCompleted
        │
        ▼  @Transactional
   OrderService.confirm
   ├─ order.confirm()                  ← orders.status = CONFIRMED
   ├─ outboxRepo.save(OrderConfirmed   ← outbox 테이블 INSERT
   │                  payload = JSON)
   └─ publisher.publish(OrderConfirmed)

────────────────── 같은 트랜잭션 끝, 같은 JVM 끝 ──────────────────

[common]
   OutboxRelay.poll @Scheduled (1초마다)
        │
        ▼  @Transactional
   ├─ outbox.lockPending(100)          ← FOR UPDATE SKIP LOCKED
   │  → PENDING 행들 잠그며 fetch
   └─ for each:
       kafka.send(record)               ← 외부 Kafka 토픽 'order-events'로
       e.markPublished()                ← outbox.status = PUBLISHED
       (실패 시 retryCount++ , 임계 초과 시 DEAD_LETTER)

────────────────── Kafka 브로커 ──────────────────

[notification]
   NotificationKafkaListener.onMessage @KafkaListener
        │
        ▼
   InboxConsumer.consume(messageId, eventType, payload)
   ├─ inbox.existsByMessageId  ← 중복 체크 (멱등성)
   │   있으면 → 카운터++ , return
   ├─ inbox.save(InboxEvent)            ← inbox 테이블 INSERT
   └─ NotificationService.process
       ├─ Notification 저장 (EMAIL)
       ├─ EmailSender.send                ← @CircuitBreaker
       │   ├─ EmailGateway.deliver         (정상)
       │   └─ fallback → DeferredEmailQueue.enqueue (OPEN)
       ├─ Notification 저장 (PUSH)
       └─ PushSender.send

────────────────── 백그라운드 운영 ──────────────────

[common] OutboxCleaner @Scheduled(daily 03:00)
   → DELETE FROM outbox WHERE status='PUBLISHED' AND published_at < NOW() - INTERVAL '7 days'

[common] DeadLetterSweeper @Scheduled(60s)
   → DEAD_LETTER outbox 행을 outbox_dlq 테이블로 이관 + 카운터 증가

[app/recovery] OrderRecoveryJob @Scheduled(30s)
   → PAYMENT_PENDING으로 60초+ 묶인 주문 + COMPLETED Payment가 있으면
   → orderService.confirm()로 강제 확정 (outbox INSERT 포함)

[notification] DeferredEmailQueue @Scheduled(30s)
   → Circuit이 닫혔으면 보류 메일 재시도 발송

[관측]
   /actuator/prometheus
   → outbox.pending.count (gauge)
   → outbox.publish.success / failure (counter)
   → inbox.processed.count / duplicate.count (counter)
   → outbox.deadletter.count (gauge)
   → order.recovered (counter)
   → JVM/HTTP/Hikari/Kafka 자동 메트릭

[관리]
   POST /admin/dlq/{id}/retry
   → DLQ 행을 PENDING outbox로 되돌림 (수동 재시도)
```

---

## 5. 핵심 설계 결정 — Why

### 5.1 EventStore + EventBus 분리

`docs/study/event-store-vs-simple-bus.md` 참조.

> **단순 맵 버스(`Map<Class, List<Handler>>`)** 만으로 끝낼 수 있지만, **EventStore를 옆에 두면 이벤트가 사건의 기록**이 된다.
> Phase 1의 인메모리 EventStore가 Phase 2의 Outbox + Kafka topic으로 자연스럽게 진화 — 사상의 연속성.

### 5.2 contract 패키지 분리

> 도메인끼리 직접 import 금지. `OrderCreated`/`PaymentCompleted` 같은 **통신용 이벤트는 common.event.contract**.
> 양 도메인이 같은 컨트랙트만 보고 통신 → MSA 분리 시 그대로 가져갈 수 있는 명시적 인터페이스.

### 5.3 도메인 → 인프라 의존, 도메인 ↔ 도메인 의존 금지

`docs/study/cross-cutting-infrastructure.md`, `docs/study/module-placement-rules.md`.

> 빌드 그래프가 곧 캡슐화 다이어그램. 컴파일러가 모듈 경계를 강제한다.
> common이 web을 의도적으로 안 갖는 것도 같은 사상.

### 5.4 외부키 없음 (orders → payments)

> Phase 1의 V1__init.sql이 외래키를 두지 않는다.
> Phase 3의 분리 가능성을 위해. 정합성은 이벤트 흐름 + 회복 잡(`OrderRecoveryJob`)이 보장.

### 5.5 트랜잭션 동기화 후 디스패치

`docs/study/spring-transaction-synchronization.md`.

> `TransactionalEventPublisher`가 `afterCommit` 훅에 등록 → "커밋 안 됐는데 이벤트 발행" / "발행 후 롤백" 함정 모두 차단.
> Phase 2의 outbox INSERT가 같은 트랜잭션에 들어가면서 강한 일관성 보강.

### 5.6 Outbox 단일 테이블 + 단일 Relay

> 도메인별로 outbox를 만드는 대신 **횡단 인프라로** 하나만. aggregate_type 컬럼으로 도메인 구분.
> 운영 단순성 + 메트릭 단일화.

### 5.7 Inbox UNIQUE 인덱스

> 멱등성의 **최후 방어선**. 애플리케이션 레벨 `existsByMessageId` 체크가 경쟁 상태에서 통과해도, DB UNIQUE가 막아준다.
> `DataIntegrityViolationException` catch로 "중복 스킵"으로 번역.

### 5.8 KafkaProducerStub로 결정적 chaos

> `kafka.stop()/start()` 컨테이너 재시작은 bootstrap URL 변경 등 비결정 요소가 큼.
> 발행 실패 자체를 stub으로 결정적으로 시뮬레이션 → 학습·CI에서 안정.

### 5.9 testFixtures로 베이스 공유

`docs/study/sharing-test-helpers.md`.

> `src/test`는 모듈 간에 자동 노출 안 됨. `KafkaIntegrationTestBase`, `Kafka{Producer,Consumer,Producer}{Stub,}` 같은 헬퍼는 `src/testFixtures/`에서 다른 모듈에 노출.

### 5.10 헥사고날 부분 도입 — `EmailGateway`

> 도메인이 인프라를 직접 의존하는 것을 외부 시스템(메일)에서만 인터페이스로 분리.
> Phase 3의 `LoggingEmailGateway`(운영) ↔ `EmailSenderStub`(테스트) 교체로 Circuit Breaker 검증.

---

## 6. 사용한 도구·프레임워크 카탈로그

| 도구 | 역할 | 학습 노트 |
|------|------|---------|
| Spring Boot 4.0 | 부트스트랩 + 자동 구성 | `spring-boot-4-migration.md` |
| Spring Framework 7 | DI, 트랜잭션, MVC | `spring-transaction-synchronization.md` |
| Spring Data JPA | ORM + Repository 추상 | `cross-cutting-infrastructure.md` |
| Spring JdbcTemplate | 보일러플레이트 없는 SQL | `jdbc-template.md` |
| Spring Kafka | KafkaTemplate + @KafkaListener | `spring-kafka-objects.md` |
| Resilience4j | Circuit Breaker | (페이즈 3 Step 5) |
| Micrometer | 메트릭 facade | `micrometer-metrics.md` |
| Prometheus | 메트릭 스크래핑 | `micrometer-metrics.md` |
| Flyway | DB 스키마 마이그레이션 | `flyway-migrations.md` |
| PostgreSQL 17 | 영속 + JSONB + 부분 인덱스 + SKIP LOCKED | `sql-fundamentals.md` |
| Kafka 3.9 (KRaft) | 분산 commit log | `event-driven-tools.md`, `spring-kafka-objects.md` |
| JUnit 5 | 테스트 프레임워크 | `testcontainers.md` |
| Mockito + `@MockitoBean` | 모킹 | `spring-boot-4-migration.md` |
| AssertJ | fluent assertion | (테스트 파일 참조) |
| Awaitility | 비동기 대기 | (각 테스트 파일) |
| Testcontainers | 진짜 인프라 격리 테스트 | `testcontainers.md` |
| ArchUnit | 모듈 경계 자동 검증 | `module-placement-rules.md` |
| Gradle 9.3 + java-test-fixtures | 빌드 + 테스트 헬퍼 공유 | `gradle-dependency-scopes.md`, `sharing-test-helpers.md` |
| Jackson | JSON 직렬화 | (Phase 2 OutboxWriter) |

---

## 7. 자바 언어/JDK 도구 카탈로그

| 도구 | 역할 | 학습 노트 |
|------|------|---------|
| `Consumer<T>` 등 함수형 인터페이스 | 핸들러 표현 | `java-functional-toolbox.md` |
| `<?>`, `<?,?>` wildcard | 타입 무관심 | `java-generics-wildcard.md` |
| `CopyOnWriteArrayList` | 읽기 폭발 워크로드 | `concurrent-collections.md` |
| `ConcurrentHashMap` | 버킷 락 기반 동시 맵 | `concurrent-collections.md` |
| `AtomicLong`, `AtomicReference` | 락 없는 원자 갱신 | (EventStore, KafkaProducerStub) |
| `CompletableFuture` | 비동기 결과 | (Spring Kafka send) |
| `record` | 불변 데이터 캐리어 | (OrderConfirmedPayload, DeferredEmail) |
| `enum` | 상태 표현 | (OrderStatus, PaymentStatus, OutboxStatus) |
| `@FunctionalInterface` | 단일 추상 메서드 표시 | (Consumer, EmailGateway) |
| Java 25 toolchain | 최신 JVM | (build.gradle) |

---

## 8. 페이즈별 진화 — 사상의 시간순

```
Phase 1: 모듈러 모놀리스 + 인메모리 이벤트 버스
   ├─ 도메인 모듈 분리 (order, payment)
   ├─ EventStore (WAL 영감)
   ├─ EventBus (pub/sub)
   ├─ TransactionalEventPublisher (커밋 후 디스패치)
   └─ "Order ↔ Payment를 같은 트랜잭션 경계 안에서 동작"
         ↓
Phase 2: Outbox / Inbox + Kafka
   ├─ Outbox 테이블 + Relay (at-least-once)
   ├─ Inbox 테이블 + UNIQUE (멱등성)
   ├─ Kafka 토픽 (외부 통신 표준)
   ├─ Notification 도메인 추가
   └─ "내부 인메모리 + 외부 Kafka의 하이브리드"
         ↓
Phase 3: 운영 안정성 + MSA 전환 준비
   ├─ Metrics (Micrometer + Prometheus)
   ├─ DLQ + 수동 재시도 (운영 가시화)
   ├─ Outbox Cleaner (테이블 비대화 방지)
   ├─ OrderRecoveryJob (DB 상태 기반 복구)
   ├─ Circuit Breaker + Deferred Queue (외부 장애 격리)
   ├─ FOR UPDATE SKIP LOCKED (멀티 인스턴스 안전성)
   ├─ ArchUnit (모듈 경계 자동 검증)
   └─ Notification 단독 실행 (분리 가능성 증명)
```

각 단계가 **이전 단계를 깨지 않으면서** 새 책임을 추가. 회귀 테스트가 가드레일.

---

## 9. "어떤 종류의 일관성을 보장하는가" — CAP / 일관성 모델

```
[같은 트랜잭션 (강한 일관성)]
   Order.create + outbox INSERT + EventBus.publish
   → 셋 다 commit 또는 셋 다 rollback

[afterCommit 훅 (커밋 후 디스패치)]
   인메모리 핸들러는 커밋이 확정된 뒤에만 실행
   → 다른 도메인이 보는 시점에는 데이터가 일관됨

[at-least-once via Outbox]
   Relay가 발행 못 했으면 retry → Kafka에 한 번 이상 도착 보장
   (단, 두 번 이상 도착 가능)

[멱등성 via Inbox]
   같은 messageId는 한 번만 처리 → 두 번 도착해도 한 번 효과

[effectively-exactly-once]
   at-least-once + 멱등성 = 결과적으로 한 번만 처리된 것과 동일

[최종 일관성 (Eventually Consistent)]
   외부 Notification은 Kafka 지연 + 처리 지연 후 최종 일치
   사용자가 "주문 확정"을 보는 시점과 "이메일 받는 시점"은 다를 수 있음
```

이 프로젝트는 **내부엔 강한 일관성, 외부엔 최종 일관성** 의 명시적 분리를 코드 구조로 실현한다.

---

## 10. 모듈 경계 위반 9회 — 회고

이 프로젝트를 만드는 동안 같은 종류의 위반이 9번 반복되었다. 모두 컴파일러/사용자 점검에 의해 잡혔고, 각 사례가 모듈 사상의 한 측면을 가르쳤다.

| # | 사례 | 가르침 |
|---|------|------|
| 1 | PaymentEventHandlerTest가 OrderService import | 도메인 ↔ 도메인 직접 의존 금지 → contract |
| 2 | OrderEventHandlerTest 동일 패턴 | 동일 |
| 3 | KafkaFailureRecoveryTest(common)가 OrderService | common → domain 역의존 금지 |
| 4 | Phase1E2ETest의 JpaRepository 미해결 | implementation 캡슐화 효과 |
| 5 | KafkaIntegrationTestBase 공유 안 됨 | testFixtures 필요 |
| 6 | DeadLetterAdminController가 common에 위치 | 도메인 무관 web 노출은 app 책임 |
| 7 | OrderRecoveryJob이 payment import | 도메인 cross orchestration은 app |
| 8 | EmailSenderStub 미존재 | 테스트 헬퍼는 명시적 정의 필요 |
| 9 | OutboxMetricsTest에 InboxConsumer + EventDispatcher 미사용 | 모듈별 메트릭 검증 분리 + 데드 코드 정리 |

**공통 본질**: build.gradle의 의존성 목록이 곧 그 모듈이 다룰 수 있는 추상의 한계. 빌드 그래프와 일치하지 않는 코드를 두면 컴파일러가 정직하게 알려준다.

---

## 11. 지표 — 운영 대시보드의 시작점

```
┌─────────────────────────────────────────────────────────────┐
│  Outbox Health                                               │
│  ┌──────────────────────┬──────────────────────┐            │
│  │ outbox.pending.count │ outbox.deadletter.count │           │
│  │   (gauge, 0이 정상)  │   (gauge, 0이어야 정상) │           │
│  └──────────────────────┴──────────────────────┘            │
│                                                              │
│  rate(outbox.publish.success[5m])  — 처리량                  │
│  rate(outbox.publish.failure[5m])  — 에러율                  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Inbox Health                                                │
│  rate(inbox.processed.count[5m])    — 정상 처리량            │
│  rate(inbox.duplicate.count[5m])    — 중복 수신율            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Domain Recovery                                             │
│  rate(order.recovered[1h])  — 시간당 복구된 stuck 주문        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  JVM / HTTP / Kafka (자동 노출)                              │
│  jvm_memory_used_bytes                                       │
│  http_server_requests_seconds  (p50, p95, p99)              │
│  kafka_consumer_lag_max                                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 12. 다음 단계 — 이 프로젝트가 여기서 더 가려면

| 방향 | 무엇을 도입 |
|------|---------|
| 진정한 분산 | Notification을 별도 서비스로 분리 (build.gradle 기준 그대로 떼어내면 됨) |
| 더 강한 캡슐화 | 헥사고날 Port/Adapter 풀세트 — `OutboxPort` 등 |
| Event Sourcing | `EventStore`를 진실의 원천으로 — 도메인 상태를 이벤트 시퀀스 재생으로 도출 |
| 분산 추적 | OpenTelemetry 도입 — Order → Payment → Notification 흐름을 trace로 |
| Outbox CDC | Debezium으로 폴링 → CDC, DB 부하 감소 + 지연 감소 |
| Saga | 여러 도메인 트랜잭션을 보상 트랜잭션 흐름으로 명시화 |
| 멀티 테넌트 | tenant_id 기반 격리 (학습 노트의 멀티테넌트 개념 적용) |

이 모든 방향이 **현재 구조 위에서 점진적으로** 도입 가능하게 코드 추상이 잡혀 있다.

---

## 13. 학습 노트 인덱스 (`docs/study/`)

| 파일 | 다루는 사상 |
|------|-----------|
| `architecture-overview.md` (이 문서) | 모든 개념의 종합 |
| `concurrent-collections.md` | CopyOnWriteArrayList, ConcurrentHashMap, 락 전략 |
| `cross-cutting-infrastructure.md` | 횡단 인프라가 common에 있는 이유, 헥사고날 대안 |
| `event-driven-tools.md` | Spring/Guava/Kafka/EventStoreDB/Axon 비교 |
| `event-store-vs-simple-bus.md` | Store + Bus 분리 vs 단순 맵 버스 |
| `flyway-migrations.md` | DB 마이그레이션 도구 |
| `gradle-dependency-scopes.md` | implementation vs api, 캡슐화 |
| `java-functional-toolbox.md` | Consumer, computeIfAbsent, 제네릭 |
| `java-generics-wildcard.md` | `<?>`, `<? extends>`, `<? super>` |
| `jdbc-template.md` | SQL 보일러플레이트 제거 |
| `micrometer-metrics.md` | Counter / Gauge / 백엔드 추상 |
| `module-placement-rules.md` | 어떤 코드가 어떤 모듈에 살아야 하는가 |
| `sharing-test-helpers.md` | testFixtures, test-support |
| `spring-boot-4-migration.md` | 4.0 모듈화 패키지 이동 |
| `spring-kafka-objects.md` | ProducerFactory, KafkaTemplate, ConsumerRecord |
| `spring-test-property-injection.md` | `@DynamicPropertySource` |
| `spring-transaction-synchronization.md` | afterCommit 훅 |
| `sql-fundamentals.md` | INTERVAL, JOIN, UNION, PostgreSQL 고유 기능 |
| `testcontainers.md` | 진짜 인프라 통합 테스트 |

이 인덱스가 "프로젝트의 뼈대를 이해하기 위한 지도" 역할을 한다.

---

## 14. 한 문장 회고

> **이 프로젝트는 "단일 JVM의 효율 + 분산 시스템의 견고함"을 양립시키는 패턴 모음의 살아있는 카탈로그다.**
> EventStore가 Kafka topic으로, in-memory subscribe가 @KafkaListener로, 인메모리 트랜잭션 동기화가 Outbox로 — **같은 사상의 다른 구현체**가 페이즈를 따라 진화한다.
> 그 진화 경로에서 모듈 경계 9번을 반복해 깨뜨리면서 빌드 그래프가 캡슐화 다이어그램이라는 사실을 손으로 익혔다.

이 학습 노트가 **다음에 비슷한 시스템을 설계할 때 30분 안에 핵심 결정을 회상할 수 있는 지도**가 되길.
