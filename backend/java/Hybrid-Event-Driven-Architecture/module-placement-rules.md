# 모듈 배치 규칙 — "어떤 코드가 어떤 모듈에 살아야 하는가"

> 이 프로젝트에서 같은 종류의 모듈 경계 위반이 여섯 번 발생한 끝에 정리한 학습 노트.
> "build.gradle의 의존성 목록이 곧 그 모듈이 다룰 수 있는 추상의 한계"라는 원칙을 명문화한다.

---

## 0. 출발점 — 반복된 실수의 패턴

이 프로젝트의 페이즈 문서를 처음 쓰면서 같은 종류의 위반이 반복적으로 드러났다:

| # | 사례 | 증상 | 본질 |
|---|------|------|------|
| 1 | `PaymentEventHandlerTest`가 `OrderService` import | 컴파일 실패 | payment가 order를 의존하지 않는데 가져옴 |
| 2 | `OrderEventHandlerTest`가 같은 패턴 | 컴파일 실패 | 동일 |
| 3 | `KafkaFailureRecoveryTest`(common)가 `OrderService` import | 컴파일 실패 | common이 order 위로 의존 |
| 4 | `Phase1E2ETest`(app)에서 `findById` 미해결 | `JpaRepository` 안 보임 | app에 spring-data-jpa 직접 추가 안 함 |
| 5 | `OutboxRelayTest`에서 `KafkaIntegrationTestBase` 미해결 | 모듈 경계 위반 | test 소스셋은 자동 노출 안 됨, testFixtures 필요 |
| 6 | `DeadLetterAdminController`가 common에 위치 | `ResponseEntity` 미해결 | common에 web 의존성 없음 |

**여섯 사례 모두 같은 원인**: "이 모듈에 무엇이 들어와야 하는가"의 결정이 빌드 그래프와 일치하지 않았다.

---

## 1. 핵심 원칙 — build.gradle = 모듈의 의도 선언

```groovy
// common/build.gradle
implementation 'org.springframework.boot:spring-boot-starter'
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
implementation 'org.springframework.boot:spring-boot-starter-actuator'
implementation 'org.springframework.kafka:spring-kafka'
implementation 'org.flywaydb:flyway-core'
implementation 'io.micrometer:micrometer-registry-prometheus'
implementation 'org.postgresql:postgresql'
implementation 'com.fasterxml.jackson.core:jackson-databind'
// ❌ spring-boot-starter-web/webmvc 없음 — 의도적
```

이 목록이 말하는 것:

> "common은 코어 + 데이터 + 메시징 + 메트릭 + 직렬화 인프라를 담당한다.
> **HTTP 노출은 내 책임이 아니다.**"

`@RestController`, `ResponseEntity`, `@RequestMapping` 같은 web 타입은 컴파일 자체가 안 된다. **컴파일러가 모듈 경계를 강제하는 도구**가 된다.

---

## 2. 이 프로젝트의 모듈 카탈로그

| 모듈 | 책임 | 절대 안 들어가는 것 |
|------|------|---------------------|
| **common** | 횡단 인프라 (EventBus, Outbox, Repository, Relay, Sweeper, Cleaner, Metrics) + 이벤트 컨트랙트 | 도메인 비즈니스 로직, HTTP 컨트롤러, admin UI |
| **order** | Order 도메인 (애그리거트, Service, Handler, Controller) | payment/notification 도메인 코드, common 위로의 역참조 |
| **payment** | Payment 도메인 | order/notification 도메인 코드 |
| **notification** | Notification 도메인 + Inbox 처리 + Kafka 컨슈머 | order/payment 도메인 코드 |
| **app** | 모든 도메인 조립 + 운영(admin) 엔드포인트 + E2E 테스트 + bootJar 생성 | 도메인 비즈니스 로직 (얇은 조립체) |

### 2.1 common이 web을 안 갖는 이유

- common은 **여러 도메인이 공유하는 메커니즘**의 자리. HTTP는 메커니즘이 아니라 **외부 노출**의 한 형태.
- common이 web을 가지면 무거운 transitive 의존(Tomcat, Jackson, validation 등)이 모든 도메인 모듈로 따라간다.
- "common = 인프라"의 의미가 "common = 뭐든지"로 흐려진다.

### 2.2 도메인 모듈끼리 안 보는 이유

`common.event.contract` 패키지가 도메인 간 통신용 이벤트(`OrderCreated`, `PaymentCompleted`, ...)를 보관. 양 도메인은 **계약(contract)만** 보고 통신.

```
order ──publish(OrderCreated)──> common.event.bus ──> payment (subscribe)
              ↑                                              ↑
         contract만 봄                                  contract만 봄
         (order는 payment를 모름)                    (payment는 order를 모름)
```

빌드 그래프에 `order ↔ payment` 의존이 없음을 컴파일러가 보장.

### 2.3 app이 admin/HTTP를 갖는 이유

- 운영자용 엔드포인트는 **모든 도메인 위에 얹히는 관제** 레이어.
- 도메인이 자기 admin을 만들면 도메인 책임이 흐려짐 — Order가 자기 Outbox 운영 API를 노출? 그건 Order의 책임이 아님.
- app은 어차피 모든 모듈을 의존하므로 어떤 admin 기능이든 표현 가능.
- 미래에 별도 `admin-service`로 분리할 때도 app에서 빼내는 게 도메인 모듈 영향 없이 깔끔.

---

## 3. 코드의 위치 결정 — 의사결정 트리

```
새 클래스를 만들 때:

  이 클래스가 비즈니스 의미를 가지는가?
  ├ Yes (Order, Payment, Notification 등)
  │   └ 그 도메인 모듈 (order/, payment/, notification/)
  │
  └ No, 메커니즘이다
      │
      여러 도메인이 같은 메커니즘을 쓰는가?
      ├ Yes (Outbox, EventBus, Metrics)
      │   └ common 모듈
      │
      └ 한 도메인 내부 구현 디테일
          └ 그 도메인 모듈

  HTTP를 노출하는가?
  ├ 도메인 자체 API → 그 도메인 모듈 (web 의존 추가)
  └ 운영자/관제용 → app 모듈 (또는 별도 admin)

  도메인 간 통신용 이벤트 / DTO인가?
  └ common.event.contract
```

---

## 4. 테스트의 위치 결정 — 같은 원칙

테스트도 코드다. 어느 모듈에 둘지가 의존을 결정한다.

```
이 테스트가 import하는 게 무엇인가?

  ├ 한 도메인 모듈만 (예: Order만) → 그 도메인 모듈의 src/test
  ├ common만 (인프라 검증) → common/src/test
  ├ 여러 도메인 함께 → app/src/test/integration (E2E)
  ├ web 검증 (MockMvc, WebTestClient) → web 의존 있는 모듈 (app, 또는 그 도메인)
  └ 여러 모듈이 공유하는 헬퍼 (KafkaIntegrationTestBase 등)
      → testFixtures 소스셋 (common/src/testFixtures)
```

### 4.1 testFixtures 도입 이유

`src/test/`는 다른 모듈에 자동 노출 안 됨. 공유 헬퍼는 `java-test-fixtures` 플러그인을 통한 별도 소스셋이 필요.

`docs/study/sharing-test-helpers.md` 참고.

### 4.2 의존성 누락의 신호

| 컴파일 오류 | 가능한 원인 |
|-----------|-----------|
| `XxxRepository.findById` 미해결 | 의존 모듈의 `spring-data-jpa`가 `implementation` 스코프 → 의존자에 노출 안 됨. 의존자 build.gradle에 직접 추가 |
| `WebTestClient` / `ResponseEntity` 미해결 | 그 모듈에 `spring-boot-starter-webmvc-test` / `webmvc` 없음 |
| 다른 도메인 클래스 미해결 | **위반** — 다른 도메인을 의존하면 안 됨. contract 패키지 사용 또는 테스트 위치 변경 |
| testFixtures 헬퍼 미해결 | `testImplementation testFixtures(project(':common'))` 누락 |

---

## 5. 책임의 직교성 — 도메인 vs 인프라 vs 컨트랙트 vs 조립

이 프로젝트의 모든 클래스가 다음 네 칸 중 하나에 들어간다.

```
                  HTTP/외부 노출
                       │
        ┌──────────────┼──────────────┐
        │              │              │
  [도메인]           [인프라]      [조립/관제]
  Order              EventBus         OrderController(API)
  Payment            OutboxEvent      DeadLetterAdminController
  Notification       OutboxRelay      Phase1E2ETest
  ...                Metrics          ChaosScenarioTest
  ...                JdbcTemplate
                     ...
        │              │
        └──────[컨트랙트]
               OrderCreated
               PaymentCompleted
               OrderConfirmedPayload
               ...
```

| 분류 | 모듈 | 특징 |
|------|------|------|
| 도메인 | order, payment, notification | 비즈니스 의미, 자기 도메인 안에서만 의존 |
| 인프라 | common | 도메인 무관, 횡단 메커니즘 |
| 컨트랙트 | common.event.contract | 도메인 간 통신용 이벤트/DTO |
| 조립/관제 | app | HTTP, E2E 테스트, admin |

**한 클래스는 정확히 하나의 분류에 속해야 한다**. "도메인이면서 인프라"인 클래스는 없다.

---

## 6. 여섯 사례의 회고 — 어디서 어디로 옮겼나

| # | 잘못된 위치 | 올바른 위치 | 옮긴 이유 |
|---|------------|------------|---------|
| 1 | `payment/test/PaymentEventHandlerTest`가 `order.service` import | 같은 자리 + 이벤트는 `common.event.contract`로 이동 | payment ↔ order 직접 의존 금지 |
| 2 | `order/test/OrderEventHandlerTest`가 같은 패턴 | 동일 | 동일 |
| 3 | `common/test/KafkaFailureRecoveryTest`가 `OrderService` import | common 안에서 자족 (outbox 직접 INSERT) | common이 도메인 위로 의존 금지 |
| 4 | `app/test/Phase1E2ETest`에서 JpaRepository 안 보임 | `app/build.gradle`에 spring-data-jpa 직접 추가 | implementation 스코프는 transitive 노출 안 함 |
| 5 | `common/src/test/.../KafkaIntegrationTestBase` | `common/src/testFixtures/.../` | 다른 모듈도 상속해야 하므로 testFixtures가 정답 |
| 6 | `common/.../DeadLetterAdminController` | `app/.../admin/DeadLetterAdminController` | common은 web 의존 없음, admin은 app 책임 |

여섯 모두 **빌드가 정직하게 잘못을 알려준 사례**다 — 컴파일 오류는 설계 흐트러짐의 신호.

---

## 7. 자주 만나는 함정 (요약)

### 7.1 "그냥 common에 넣자"의 유혹

가장 흔한 실수. "공통이니까 common"으로 시작하지만:

- 한 도메인만 쓰면 → 그 도메인 모듈
- 운영 전용 (HTTP) → app
- 도메인 간 통신용 → common.event.contract
- 진짜 횡단 메커니즘만 → common.outbox / common.event 등

### 7.2 "테스트는 어디 둬도 컴파일만 되면 OK"의 유혹

테스트도 모듈 경계 안에서 살아야 한다. 그렇지 않으면:
- `implementation` 의존이 의도한 캡슐화가 깨짐.
- 테스트 헬퍼가 운영 jar에 새는 사고 발생.
- 모듈 분리 시 테스트가 같이 안 따라감.

### 7.3 "그냥 api로 풀면 되잖아"의 유혹

`implementation` → `api`로 바꾸면 컴파일 오류는 사라진다. 하지만:
- 모든 의존자가 그 transitive 의존을 끌고 가게 됨.
- 빌드 그래프가 거미줄.
- "왜 이 모듈에 이 라이브러리가 들어와 있지?" 추적 불가능.

→ 진짜 공개 API에 등장하는 타입만 `api`. 컴파일 오류 회피용으로는 사용 금지. 자세히는 `docs/study/gradle-dependency-scopes.md`.

### 7.4 "도메인 간 직접 호출이 빠르잖아"의 유혹

`OrderService`를 PaymentService에서 직접 호출하면 코드가 짧아진다. 하지만:
- 도메인 분리 의지가 깨짐.
- Phase 3의 마이크로서비스 전환이 불가능해짐.
- 단방향 의존 + 이벤트 통신 사상 전체가 무너짐.

→ 항상 이벤트 컨트랙트 (`common.event.contract`)를 거친다.

### 7.5 "운영 admin 기능을 도메인에 두자"의 유혹

OrderService에 `/admin/orders/cleanup` 같은 메서드가 들어가는 케이스. 하지만:
- 도메인 책임이 흐려짐.
- admin 보안 정책(인증/인가)이 도메인 코드에 섞임.
- 운영 UI 분리 시 도메인 코드를 수정해야 함.

→ 운영 엔드포인트는 항상 app(또는 별도 admin 모듈)이 책임.

---

## 8. 자가 점검 체크리스트

새 클래스/테스트를 만들 때 5초 자가 점검:

- [ ] 이 클래스가 어느 분류(도메인/인프라/컨트랙트/조립)인가?
- [ ] 이 클래스가 import하는 모듈이 build.gradle에 다 있는가?
- [ ] 이 클래스가 다른 도메인 모듈을 import하는가? (그렇다면 잘못)
- [ ] 이 테스트가 가져온 헬퍼가 testFixtures에 있는가?
- [ ] 이 코드에 web 타입이 있는데 그 모듈에 web 의존이 있는가?
- [ ] 이 코드를 다른 모듈로 옮길 때 import만 바꾸면 되는가? 아니라면 **잘못된 위치**.

---

## 9. 한 줄 요약

> **`build.gradle`의 의존성 목록은 곧 그 모듈이 다룰 수 있는 추상의 한계 선언이다.** 그 선언과 일치하지 않는 코드를 두면 컴파일러가 정직하게 알려준다.
> common은 web을 갖지 않으므로 `@RestController`가 살 수 없고, 도메인 모듈끼리 의존하지 않으므로 `OrderService`가 payment에서 보이지 않는다.
> **빌드 그래프 = 캡슐화 다이어그램** — 이 프로젝트가 같은 사실을 여섯 번 다른 모습으로 증명했다.

---

## 10. 함께 보면 좋은 자료

- `docs/study/gradle-dependency-scopes.md` — implementation vs api, 의존 그래프 캡슐화
- `docs/study/cross-cutting-infrastructure.md` — common이 인프라를 갖는 이유
- `docs/study/sharing-test-helpers.md` — testFixtures와 test-support 모듈
- `docs/study/event-store-vs-simple-bus.md` — 도메인 간 통신을 contract로 분리하는 사상
- `docs/phase-3-tdd.md` Step 6 — ArchUnit으로 모듈 경계 자동 검증

---

## 11. 다음 단계 — ArchUnit으로 자동화

지금까지의 규칙은 **사람의 점검**에 의존한다. Phase 3에서 ArchUnit을 도입하면 빌드 시점에 자동 검증 가능.

```java
@AnalyzeClasses(packages = "com.hybrid")
class ModuleBoundaryTest {

    @ArchTest
    static final ArchRule order_payment_상호_의존_금지 =
        noClasses().that().resideInAPackage("com.hybrid.order..")
            .should().dependOnClassesThat().resideInAPackage("com.hybrid.payment..");

    @ArchTest
    static final ArchRule common_은_도메인_의존_금지 =
        noClasses().that().resideInAPackage("com.hybrid.common..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "com.hybrid.order..", "com.hybrid.payment..", "com.hybrid.notification..");

    @ArchTest
    static final ArchRule common_은_web_의존_금지 =
        noClasses().that().resideInAPackage("com.hybrid.common..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "org.springframework.web..", "org.springframework.http..");
}
```

이 세 규칙이 자동 통과하면 이 문서의 핵심 규칙이 빌드 시점에 강제된다.

→ 다음 위반은 **컴파일이 아니라 ArchUnit이 먼저** 잡아낼 것이다.
