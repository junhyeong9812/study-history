# Domain Events, Outbox, Inbox 패턴 정리

> 작성일: 2026-05-01
> 배경: VulnScope v0.1 에 발행자(publisher)는 있지만 구독자(listener)가 없는 도메인 이벤트가 4개 있었음 (`ScanCompleted`, `ScanFailed`, `FindingRecorded`, `TargetRegistered`). YAGNI 위반으로 제거. 그 의도가 무엇이었는지, 정통 패턴으로는 어떻게 구현되는지 학습용으로 정리.
>
> 결정 기록: 4개 이벤트 발행 코드 제거. `ScanRequested` 만 유지 (`WorkerExecutor` 가 실제 구독). v0.2 에 notification/audit 모듈 추가 시점에 이벤트 시스템 같이 도입.

---

## 1. 왜 이런 패턴이 등장하는가 — 문제 상황

도메인 모듈 A 가 도메인 사실(domain fact)을 발생시킬 때, **다른 모듈 B/C/D 가 그 사실에 반응**해야 하는 경우:

- A: `RecordFindingService.record()` — finding 발견.
- B: notification — Slack/Email 알림 보내야 함.
- C: audit — 감사 로그 적어야 함.
- D: integration — Jira 티켓 만들어야 함.

**나이브 구현 (do not do)**:
```java
public Finding record(FindingPayload p) {
    Finding saved = repository.save(...);
    slackService.send(...);     // ① B
    auditService.log(...);      // ② C
    jiraService.createIssue(...); // ③ D
    return saved;
}
```

**문제**:
1. **결합도 폭발**: A 가 B/C/D 모두 의존. 5개 모듈 추가 시 A 가 5번 변경.
2. **트랜잭션 경계 흐림**: ② 실패 시 ① 은 이미 보내짐 (rollback 불가). 부분 성공.
3. **신뢰성**: ① slack 호출이 5초 지연 → record() 가 5초 블로킹. 사용자 응답 느림.
4. **테스트 비용**: A 의 단위 테스트가 B/C/D 모두 mock 필요.

→ **도메인 이벤트 패턴**으로 해결.

---

## 2. 도메인 이벤트 패턴 (Domain Events)

A 는 "사실"만 발행. B/C/D 가 자기 책임에서 구독.

```java
// A — 발행만, 누가 듣는지 모름
public Finding record(FindingPayload p) {
    Finding saved = repository.save(...);
    events.publishEvent(new FindingRecorded(saved.id(), ...));
    return saved;
}

// B — 자기 책임에서 listen
@Component
class FindingNotifier {
    @EventListener
    void on(FindingRecorded event) {
        if (event.severity() == Severity.CRITICAL) slack.send(...);
    }
}

// C — 자기 책임에서 listen
@Component
class FindingAuditor {
    @EventListener
    void on(FindingRecorded event) {
        auditLog.append(...);
    }
}
```

**효과**:
- A 가 B/C/D 직접 의존 X.
- 새 listener 추가 시 A 코드 변경 0.
- 테스트는 A 와 B/C/D 각자.

**Spring 의 기본 `@EventListener`**:
- **동기**: publish 호출 스레드에서 listener 직접 실행. record() 가 listener 끝까지 대기.
- **트랜잭션 통합 X**: rollback 무관, listener 가 publish 시점에 즉시 실행.
- → 단순하지만 신뢰성 약함.

---

## 3. 신뢰성 문제 — "부분 실패"

도메인 이벤트의 큰 함정:

```
[트랜잭션 시작]
  ① INSERT finding              ← DB 변경
  ② publishEvent(FindingRecorded) ← Listener 가 Slack 호출
                                    └─► Slack 다운, 에러 throw
  ③ ROLLBACK                    ← finding INSERT 도 취소됨
[트랜잭션 끝]
```

**문제**: Slack 일시 장애가 finding 영속 자체를 막음. 도메인 자체 무결성이 외부 시스템 가용성에 묶임.

또는 반대:

```
[트랜잭션 시작]
  ① INSERT finding
  ② COMMIT                      ← DB 영속 완료
[트랜잭션 끝]
  ③ publishEvent → Slack 호출
                   └─► 프로세스 다운 (kill, OOM)
                       Slack 알림 영원히 발송 X
```

**문제**: DB 영속 됐지만 알림 누락. **at-most-once delivery**. 신뢰 못 함.

→ **Outbox 패턴**으로 해결.

---

## 4. Outbox 패턴 (이게 본질)

**핵심 아이디어**: 이벤트를 외부 시스템에 직접 발행 X. 같은 DB 트랜잭션에 "이벤트 row" 로 저장 → 별도 워커가 outbox 읽어서 발송.

```
[트랜잭션 시작]
  ① INSERT finding
  ② INSERT outbox(event=FindingRecorded, status=PENDING)  ← 같은 트랜잭션
  ③ COMMIT                                                ← 둘 다 atomic
[트랜잭션 끝]

[별도 outbox dispatcher (스케줄러)]
  ④ SELECT * FROM outbox WHERE status=PENDING
  ⑤ for each: 실제 listener 호출 (Slack/Email/...)
  ⑥ UPDATE outbox SET status=DISPATCHED
  ⑦ 실패 시 retry (exponential backoff)
```

**보장**:
- **at-least-once delivery**: outbox row 가 영속이라 프로세스 죽어도 다음 dispatcher 가 처리.
- **트랜잭션 일관성**: finding 영속과 이벤트 영속이 같은 commit. 부분 실패 X.
- **외부 시스템 분리**: Slack 다운이 record() 응답 시간 영향 0.

**비용**:
- outbox 테이블 + dispatcher 워커 추가.
- dedupe 책임 (consumer 측 — Inbox 패턴, 5번 참조).
- 순서 보장은 별도 (FIFO 큐 + per-aggregate 키).

**Spring Modulith 의 지원**:
- `@ApplicationModuleListener` 어노테이션 = `@EventListener + @Async + @TransactionalEventListener(AFTER_COMMIT) + outbox`.
- `spring-modulith-events-jpa` 모듈이 outbox 테이블 자동 관리.

```java
// Spring Modulith outbox listener
@ApplicationModuleListener
void on(FindingRecorded event) {                // 자동:
    slack.send(event);                           // - AFTER_COMMIT 에서 실행
}                                                // - 실패 시 outbox row 유지 + 재시도
                                                 // - 다른 트랜잭션에서 비동기 실행
```

---

## 5. Inbox 패턴 (Idempotent Receiver)

Outbox 의 **at-least-once** 라는 말은 **중복 가능** 의 뜻. dispatcher 가 발송 후 ACK 받기 전에 죽으면 다음 사이클에서 같은 이벤트 다시 발송.

→ **Consumer 가 같은 이벤트 두 번 받아도 같은 결과** (idempotent) 보장 필요.

**Inbox 테이블**: consumer 측에 "받은 이벤트 ID" 기록. 같은 ID 재수신 시 무시.

```java
@ApplicationModuleListener
void on(FindingRecorded event) {
    String eventId = event.id().toString();
    if (inbox.contains(eventId)) {
        return;                              // 이미 처리됨
    }
    slack.send(event);
    inbox.record(eventId);                   // 처리 완료 마킹
}
```

또는 **자연스럽게 idempotent 한 작업**으로 설계:
- "Slack 메시지 발송" 은 idempotent X (두 번 보내면 두 개 메시지).
- "DB row UPSERT (key=eventId)" 는 idempotent O.
- "Email subject 가 eventId 포함" + Email 서버의 dedupe → idempotent 흉내 가능.

---

## 6. 패턴 비교표

| 패턴 | 신뢰성 | 복잡도 | 사용 시점 |
|---|---|---|---|
| **직접 호출** (A → B → C) | n/a | 낮음 | 결합 OK + 단순 케이스 |
| **Spring `@EventListener` 동기** | 약함 (rollback 동결) | 낮음 | 같은 트랜잭션 내 read-only side effect |
| **Spring `@TransactionalEventListener(AFTER_COMMIT)`** | 중 (commit 후 비동기) | 중 | 외부 시스템에 알리되 영속 분리 |
| **Outbox (at-least-once)** | 강함 | 높음 | 외부 시스템 신뢰 보장 필수 |
| **Outbox + Inbox (idempotent)** | 매우 강함 | 매우 높음 | 결제/주문 같은 비즈니스 critical |

---

## 7. VulnScope 의 결정

### 7.1 v0.1 의 실제 상태 (정정 전)

5개 도메인 이벤트 발행:
| 이벤트 | 발행자 | listener | 신뢰성 모델 |
|---|---|---|---|
| ScanRequested | TriggerScanService | WorkerExecutor (`@EventListener + @Async`) | Spring 기본 |
| ScanCompleted | ScanLifecycleService | **0** | n/a |
| ScanFailed | ScanLifecycleService | **0** | n/a |
| FindingRecorded | RecordFindingService | **0** | n/a |
| TargetRegistered | RegisterTargetService | **0** | n/a |

→ **4개는 listener 없음**. 발행은 silent no-op. Outbox 도 아니고 Spring async 도 아닌 **그냥 무효 코드**.

### 7.2 정정

- 4개 이벤트 클래스 **제거** (ScanCompleted, ScanFailed, FindingRecorded, TargetRegistered).
- 각 service 의 `events.publishEvent(...)` 호출 + `ApplicationEventPublisher` 의존 제거.
- 테스트의 `RecordingPublisher` + `register_publishes_event` 같은 검증 제거.
- 문서에서 해당 이벤트 언급 제거.

### 7.3 ScanRequested 만 유지하는 이유

```java
// TriggerScanService.trigger()
events.publishEvent(new ScanRequested(saved.id(), ...));
                ↓
// WorkerExecutor (scanengine 모듈)
@EventListener @Async
public void on(ScanRequested event) {
    execute(event);  // 워커 시작
}
```

→ **이건 진짜 동작**. 제거하면 워커가 안 돔. listener 가 있는 유일한 이벤트.

### 7.4 v0.2 에 notification 모듈 추가 시

그 시점에 같이 도입할 것:
1. `FindingNotified` (또는 재명명 `FindingRecorded`) 이벤트 클래스 신규.
2. RecordFindingService 에 publishEvent 코드 한 줄 복원.
3. `notification/FindingNotifier` 가 `@ApplicationModuleListener` 로 구독 (outbox 자동).
4. `spring-modulith-events-jpa` 의존 추가 → outbox 테이블 자동 생성.

→ **이벤트 + listener + outbox 가 같이 등장**. 셋이 분리되면 무효 표면만 남음.

---

## 8. 학습 포인트 (한 줄 요약)

1. **Domain event** = "다른 모듈에 알릴 도메인 사실". A 는 발행만, B/C/D 가 구독.
2. **Spring 기본 `@EventListener`** = 동기 + 트랜잭션 통합 X. 단순 side effect 만.
3. **`@TransactionalEventListener(AFTER_COMMIT)`** = commit 후 실행. 트랜잭션과 분리.
4. **Outbox 패턴** = 같은 트랜잭션에 이벤트 row INSERT → 별도 dispatcher 가 발송. **at-least-once + 영속 일관**.
5. **Inbox 패턴** = consumer 측 dedupe. 중복 수신 안전성.
6. **Spring Modulith `@ApplicationModuleListener`** = `@EventListener + @Async + AFTER_COMMIT + outbox` 한 어노테이션.
7. **YAGNI 우선**: listener 없으면 발행 자체를 만들지 말 것. 추상 표면 미리 깔지 말 것.
8. **둘은 짝**: 이벤트 + listener 둘 다 있을 때만 의미. 한쪽만 있으면 dead code.
9. **신뢰성 모델 명시**: at-most-once / at-least-once / exactly-once 중 어디인지 공유 약속.
10. **v0.1 무효 표면 제거 = 학습/유지보수 비용 감소**. 진짜 필요할 때 5분 추가 비용.

---

## 9. 추가 참고

- ADR `docs/decisions/0002-realtime-bus-strategy.md` — 실시간 UI push 와 도메인 이벤트의 신뢰성 모델 분리.
- Spring Modulith Events 가이드: https://docs.spring.io/spring-modulith/reference/events.html
- Vaughn Vernon, "Implementing DDD" — Chapter 8 (Domain Events).
- Chris Richardson, "Microservices Patterns" — Outbox / Saga 패턴.

---

## 10. 본 프로젝트의 현재 상태 검색

```bash
# 현재 도메인 이벤트 (1개만 — ScanRequested)
grep -rn "publishEvent" back/src/main --include="*.java"

# listener (1개만 — WorkerExecutor)
grep -rn "@EventListener\|@ApplicationModuleListener" back/src --include="*.java"
```

→ 두 검색 모두 결과 1개씩이면 정상. v0.1 의 의도된 상태.
