# Event Store + Event Bus 분리 vs 단순 맵 기반 이벤트 버스

> 인메모리 메시징을 만들 때 두 가지 접근의 트레이드오프 정리.
> 이 프로젝트가 왜 후자(Store + Bus 분리)를 선택했는가에 대한 학습 노트.

---

## 1. 두 가지 설계

### 1.1 단순 맵 기반 이벤트 버스 (흔한 방식)

```java
class SimpleEventBus {
    Map<Class<?>, List<Consumer<?>>> handlers;

    void publish(Object event) {
        handlers.get(event.getClass()).forEach(h -> h.accept(event));
        // 이벤트는 "전달"만 되고 사라진다 (fire-and-forget)
    }
}
```

- 이벤트는 **휘발성**. 핸들러가 처리하면 끝, 어디에도 안 남는다.
- Spring `ApplicationEventPublisher`, Guava `EventBus`, RxJava `Subject` 등이 이 모델.
- **메시지(message)** 에 가까운 사고방식.

### 1.2 Store + Bus 분리 (이 프로젝트의 방식)

```java
class InMemoryEventBus {
    EventStore store;                              // 통합 append-only 로그
    Map<Class<?>, List<Consumer<?>>> handlers;     // 라우팅 테이블

    void publish(DomainEvent event) {
        store.append(event);              // ① 영속화 (offset 부여, 순서 고정)
        handlers.get(event.getClass())    // ② 라우팅
                .forEach(h -> h.accept(event));
    }
}
```

- 이벤트는 **사건의 기록**으로 남는다. offset과 함께.
- 핸들러가 호출되든 말든 **로그는 그대로 존재**.
- 사실상 **Kafka의 인메모리 축소판**.
- **사건(event as fact)** 에 가까운 사고방식.

---

## 2. 책임의 분리

```
┌─────────────────────────────────────────────────┐
│  EventStore = 통합 append-only 로그              │
│  ┌───────────────────────────────────────────┐ │
│  │ [0] OrderCreated                          │ │
│  │ [1] PaymentRequested                      │ │
│  │ [2] PaymentCompleted                      │ │
│  │ [3] OrderConfirmed                        │ │
│  └───────────────────────────────────────────┘ │
│   ↑ 타입 무관 단일 시퀀스, offset = 전역 순서     │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  EventBus = pub/sub 라우터 (이벤트 안 보관)       │
│  Map<Class<?>, List<Consumer<?>>> handlers       │
│  ┌───────────────────────────────────────────┐ │
│  │ OrderCreated     → [PaymentEventHandler]  │ │
│  │ PaymentCompleted → [OrderEventHandler]    │ │
│  │ OrderConfirmed   → [Outbox 라이터]         │ │
│  └───────────────────────────────────────────┘ │
│   ↑ 타입별 핸들러를 찾기 위한 매핑만             │
└─────────────────────────────────────────────────┘
```

| 컴포넌트 | 책임 | 자료구조 | 동시성 전략 |
|---------|------|---------|-----------|
| `EventStore` | 이벤트 보관 (사건의 기록) | 통합 append-only 리스트 | `synchronized append` + `AtomicLong offset` |
| `EventBus` | 이벤트 라우팅 (구독자 디스패치) | `Map<Class, List<Handler>>` | `ConcurrentHashMap` + `CopyOnWriteArrayList` |

각자의 워크로드에 맞춘 동시성 자료구조 선택이 다르다.

---

## 3. 이 패턴의 정식 이름

이 설계는 새로운 발명이 아니라 잘 알려진 패턴이다.

| 이름 | 의미 |
|------|------|
| **Event Log / Commit Log 패턴** | Kafka의 핵심 자료구조와 동일. append-only + offset 기반 읽기 |
| **Event Sourcing의 일부** | 상태가 아니라 사건의 시퀀스를 진실의 원천으로 삼는 사상의 한 조각 |
| **Pub/Sub + Persistence 결합** | 메시징(라우팅)과 저장(역사)을 두 컴포넌트로 분리 |

실무에서 Kafka, EventStoreDB, Axon Framework 같은 도구를 쓰는 시스템들은 사실 다 이 모델이다.

---

## 4. 트레이드오프 표

| 측면 | 단순 맵 버스 | Store + Bus 분리 |
|------|-----------|---------------|
| **복잡도** | 낮음 — 클래스 하나 | 살짝 높음 — 두 컴포넌트 + 협력 |
| **메모리** | 핸들러만 보관 | 모든 이벤트 누적 (정리 정책 필요) |
| **처리 비용** | 디스패치만 | append + 디스패치 |
| **이벤트의 의미** | 전달용 메시지 | 사건의 기록 |
| **재생(replay)** | ❌ 불가능 (이미 사라짐) | ✅ offset으로 자유롭게 |
| **순서 보장** | ❌ 약함 (스레드/큐 정책 의존) | ✅ 전역 offset이 곧 순서 |
| **감사(audit)** | ❌ 별도 로깅 필요 | ✅ Store가 곧 감사 로그 |
| **디버깅** | 어려움 — 사건 흐름 재구성 어려움 | 쉬움 — 시간순 시퀀스를 그대로 봄 |
| **테스트 용이성** | 핸들러 호출 여부만 검증 | offset/스토어 상태로도 검증 가능 |
| **외부 브로커로의 진화** | 코드 거의 새로 작성 | 인터페이스 그대로 유지, 구현체만 교체 |
| **학습 곡선** | 낮음 | 살짝 높음 |
| **소규모 통신에 적합** | ✅ | 과함 |
| **장기 진화 가능성** | 낮음 | 높음 |

---

## 5. Store가 있어야 가능한 일

### 5.1 Replay (재생)

```java
// 오프셋 100부터 1000건 다시 읽기
store.read(100, 1000).forEach(handler::accept);
```

- 새 핸들러가 늦게 등록되어도, 과거 이벤트를 처음부터 다시 처리 가능.
- 단순 맵 버스에선 불가능 — 이벤트가 이미 사라짐.

### 5.2 디버깅과 감사

> "이 주문이 왜 이 상태가 됐지?"

→ Store를 시간순으로 훑어보면 **사건의 흐름**이 그대로 드러남.
→ 단순 맵 버스에선 외부 로깅을 별도로 만들어야 함.

### 5.3 외부 메시지 브로커로의 매끄러운 진화

```
EventStore (인메모리 append-only) → Kafka topic (분산 append-only)
EventBus.publish()                → KafkaProducer.send()
EventBus.subscribe()              → KafkaConsumer
```

- 개념과 인터페이스가 동일.
- 단순 맵 버스로 시작했으면 Kafka 도입 시 코드 거의 새로 짜야 함.

### 5.4 동시성 일관성 (전역 순서)

```java
// InMemoryEventStore.append
public synchronized long append(DomainEvent event) {
    long current = offset.getAndIncrement();
    log.add(event);
    return current;
}
```

- offset이 곧 **전역 발생 순서의 정의**.
- 단순 맵 버스는 "어떤 이벤트가 먼저 발생했는가"를 확정할 방법이 없음.

### 5.5 다중 구독자의 독립적 진행

각 핸들러(혹은 외부 시스템)가 자기 offset을 들고 자기 속도로 진행 가능.
Kafka의 consumer group이 이걸 그대로 한다. 단순 맵 버스는 "동기 동시 호출"만 가능.

---

## 6. 단순 맵 버스로 충분한 경우

이 분리가 항상 옳은 건 아니다. 아래 조건이면 단순 버스가 더 낫다.

- 도메인 이벤트가 **닫힌 모듈 안에서만** 흐른다.
- 이벤트의 **수가 적고**, 잃어도 큰 문제 없다.
- **외부 브로커 도입 계획이 없다**.
- 디버깅은 다른 수단(로그, APM)으로 이미 잘 되어 있다.
- 학습/예제 프로젝트라 단순함이 가치다.

> Spring 자체의 `ApplicationEventPublisher`가 단순 버스 모델인 이유 — Spring 컨텍스트 내부 통지용으로 충분하고, 영속/재생까지 책임지면 프레임워크 본분을 넘어선다.

---

## 7. Store + Bus가 정답인 경우

- **곧 외부 시스템(다른 서비스, 다른 DB, 다른 큐)으로 통신을 확장할 가능성**이 있다.
- 도메인 이벤트가 **비즈니스적으로 중요**하다 (주문, 결제, 송금 등).
- 사건의 **역사가 진실**이다 — "지금 상태"보다 "어떻게 여기까지 왔는가"가 중요하다.
- **Event Sourcing / CQRS** 사상으로 점진적으로 가고 싶다.
- 디버깅/감사를 코드 자체로 해결하고 싶다.

> 이 프로젝트의 Phase 2/3가 정확히 이 방향. Phase 1의 인메모리 Store가 Phase 2에서 Outbox + Kafka topic으로 자연스럽게 확장된다. **개념의 연속성**이 핵심 가치다.

---

## 8. 이 프로젝트의 결정

`PHASE_PLAN.md`와 `LEARNING_NOTE.md`의 의도를 따르면 명확하다.

- **목표**: "내부는 가볍게, 외부는 견고하게"의 하이브리드 아키텍처 학습.
- **Phase 1**: 인메모리 이벤트 버스로 시작 (모듈러 모놀리스).
- **Phase 2**: Kafka + Outbox/Inbox 패턴 도입 (외부 통신).
- **Phase 3**: Notification 서비스 분리 준비.

이 진화 경로가 자연스러우려면 **처음부터 "이벤트는 사건의 기록"이라는 사고방식**으로 시작해야 한다. 그래야 Phase 2의 outbox 테이블, Phase 3의 Kafka topic이 같은 추상의 다른 구현체로 보인다.

만약 Phase 1을 단순 맵 버스로 시작했다면:

- Phase 2에서 outbox 도입 시 **이벤트의 의미가 바뀜** (메시지 → 사건). 도메인 코드 전체 재고.
- "이벤트가 발행됐다"의 의미가 인메모리(즉시 휘발) vs Kafka(영속)에서 달라짐. 추론 비용 폭증.
- 테스트 패턴도 달라짐 (단순 호출 검증 vs offset/store 검증).

즉, 이 프로젝트의 Store + Bus 분리는 **나중을 위한 보험이 아니라, Phase 2/3에서 발생할 개념 충돌을 처음부터 차단하기 위한 설계 결정**이다.

---

## 9. 한 줄 요약

> **"인메모리 이벤트 버스"라고 하면 대부분 맵 하나짜리 디스패처를 떠올리지만, 그 옆에 Store를 두면 이벤트 자체가 데이터(사건의 역사)가 된다.**
> 단순 버스는 "메시지 전달", Store + Bus는 "사건의 기록 + 사건의 알림".
> 트레이드오프는 명확하다 — **단순함을 포기하고, 진화 가능성·관찰 가능성·재생 가능성을 얻는다.**
> 외부 브로커로 갈 계획이 있다면 처음부터 분리하는 쪽이 옳고, 닫힌 모듈 내부 통지면 단순 맵으로 충분하다.

---

## 10. 함께 보면 좋은 자료

- `docs/study/concurrent-collections.md` — 두 컴포넌트가 다른 동시성 자료구조를 쓰는 이유
- `docs/phase-1-tdd.md` Step 2~3 — EventStore, EventBus 구현 사이클
- `docs/phase-2-tdd.md` — 인메모리 Store가 outbox + Kafka로 확장되는 모습
- Martin Fowler, *Event Sourcing*: https://martinfowler.com/eaaDev/EventSourcing.html
- Jay Kreps, *The Log: What every software engineer should know about real-time data's unifying abstraction*
