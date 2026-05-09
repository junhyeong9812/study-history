# Gradle 의존성 스코프와 모듈 경계

> 멀티모듈 프로젝트에서 `implementation`, `api`, `compileOnly` 등의 차이가 곧 **모듈 캡슐화의 강도**를 결정한다.
> 이 프로젝트에서 실제로 마주친 컴파일 오류를 출발점으로, 스코프 선택의 원칙을 정리한 학습 노트.

---

## 0. 출발점 — 실제 마주친 오류

`Phase1E2ETest`(app 모듈에 위치)에서 컴파일 오류 두 개:

```
'OrderRepository'의 메서드 'findById'을(를) 해결할 수 없습니다
org.springframework.data.jpa.repository.JpaRepository에 액세스할 수 없습니다
```

코드 자체는 정상이다.

```java
// app/src/test/java/com/hybrid/integration/Phase1E2ETest.java
@Autowired OrderRepository orderRepository;

void someTest() {
    Order o = orderRepository.findById(orderId).orElseThrow();   // ← 컴파일 오류
}
```

`OrderRepository`는 `JpaRepository<Order, Long>`을 상속하므로 `findById`가 있어야 한다. 그런데 컴파일러가 **부모 인터페이스 `JpaRepository`에 접근할 수 없다**고 한다.

원인은 **Gradle 의존성 스코프**다.

---

## 1. 무엇이 일어났는가 — 의존성 그래프

```
[빌드 의존 그래프]

  app  ───implementation───>  order
                                │
                                │ implementation
                                ▼
                         spring-data-jpa  (JpaRepository 들어있음)
```

```
order/build.gradle:
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
```

`order` 모듈은 자기 안에서 `JpaRepository`를 써서 `OrderRepository`를 정의했다 — 컴파일 OK.

문제는 `app`이 `order`를 **`implementation` 스코프로** 의존했을 때 발생한다.

| 컴파일 시점 | `JpaRepository`가 보이는가 |
|-----|-----|
| `order`를 컴파일할 때 | ✅ 자기 의존성이니 보임 |
| `app`을 컴파일할 때 | ❌ `order`의 `implementation` 의존성은 노출 안 됨 |

`app`에서 `OrderRepository`를 받아 `findById`를 호출하면, 컴파일러는 **`OrderRepository`의 부모 인터페이스(`JpaRepository`)를 확인해야** 한다. 부모가 안 보이면 메서드 시그니처도 못 본다.

---

## 2. Gradle 스코프 한 장 정리

```
┌───────────────────────────────────────────────────────────────┐
│                    의존성 스코프                                │
├──────────────┬────────────┬─────────────┬─────────────────────┤
│              │ 컴파일에 보임│ 의존자에 노출│ 런타임에 포함         │
├──────────────┼────────────┼─────────────┼─────────────────────┤
│ implementation │    ✅      │     ❌      │       ✅            │
│ api            │    ✅      │     ✅      │       ✅            │
│ compileOnly    │    ✅      │     ❌      │       ❌            │
│ runtimeOnly    │    ❌      │     ❌      │       ✅            │
│ testImplementation │ ✅(테스트)│     -      │ ✅(테스트만)         │
│ testRuntimeOnly │ ❌(테스트)  │     -      │ ✅(테스트만)         │
└──────────────┴────────────┴─────────────┴─────────────────────┘
```

### 2.1 `implementation`

```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
```

- **이 모듈을 컴파일할 때만** 의존성이 보임.
- **이 모듈을 의존하는 다른 모듈에는 노출 안 됨**.
- 런타임 클래스패스에는 포함 (이 모듈이 실제로 쓰니까).

이게 **거의 모든 경우의 기본 선택**이다. 과거 `compile` 스코프(deprecated)의 후계 — 더 엄격한 캡슐화.

### 2.2 `api`

```groovy
api 'org.springframework.boot:spring-boot-starter-data-jpa'
```

- 컴파일에 보임 + **의존자에게도 노출**.
- 의존자는 추가 선언 없이 `JpaRepository`를 import할 수 있음.

라이브러리 모듈에서 자기 API 시그니처에 의존성 타입을 노출할 때 쓴다.

```java
// order 모듈에서 외부에 노출하는 API
public interface OrderRepository extends JpaRepository<Order, Long> { ... }
                                          ↑ 이 타입이 외부 API에 등장
```

엄밀히 말하면 `OrderRepository`를 외부가 변수 타입으로 받아 메서드를 호출한다는 건 `JpaRepository`도 외부에서 보여야 한다는 뜻 → 이론적으로는 `api`가 맞을 수 있다. 하지만…

### 2.3 왜 `api`로 안 풀고 `app`에 직접 추가하는가

`api`로 풀면 **`order`의 모든 의존자가 강제로 spring-data-jpa를 끌고 오게** 된다.

- `notification`이 `order`를 참조한다면 spring-data-jpa가 따라옴.
- 미래의 `analytics` 같은 모듈이 `order`를 참조해도 따라옴.
- 그 모듈들이 실제로 JPA를 쓰지 않더라도.

이건 **"의존성을 사용하는 책임"** 을 흐리는 행위다. **데이터 액세스 계층을 의식해서 다루는 모듈만 spring-data-jpa를 명시**하는 게 깔끔하다.

```groovy
// 이 프로젝트의 선택
// app/build.gradle
dependencies {
    implementation project(':common')
    implementation project(':order')
    implementation project(':payment')
    implementation 'org.springframework.boot:spring-boot-starter-webmvc'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'  // ← 직접 명시
}
```

> "Repository를 만지는 모듈은 자기가 데이터 액세스 계층을 인지한다"는 의도.

### 2.4 `compileOnly`

```groovy
compileOnly 'org.projectlombok:lombok'
```

- 컴파일에만 보이고, 런타임 클래스패스엔 없음.
- 어노테이션 프로세서, 빌드 시점 코드 생성기, 옛 `provided` 시맨틱(서블릿 컨테이너 제공) 등에 쓴다.

### 2.5 `runtimeOnly`

```groovy
runtimeOnly 'org.postgresql:postgresql'
```

- 컴파일엔 안 보이고 런타임에만 필요.
- JDBC 드라이버, 로깅 백엔드(`logback`) 등 — 코드에서 직접 import할 필요 없는 의존성.

### 2.6 테스트 스코프

```groovy
testImplementation 'org.testcontainers:junit-jupiter:1.20.4'
testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
```

- `testImplementation`: 테스트 코드에서 import 가능. 운영 jar에는 안 들어감.
- `testRuntimeOnly`: 테스트 실행에만 필요한 런타임 의존성.

---

## 3. 이 프로젝트의 실제 적용

### 3.1 모듈별 의존성 스코프 표

| 모듈 | 의존성 | 스코프 | 이유 |
|-----|------|------|------|
| `common` | spring-data-jpa | `implementation` | OutboxEvent를 위한 JPA — common 안에서만 |
| `order` | `:common` | `implementation` | order는 common의 EventBus/OutboxEvent 사용 |
| `order` | spring-data-jpa | `implementation` | order는 자기 Repository 정의에 사용 |
| `app` | `:order` | `implementation` | E2E 통합 |
| `app` | spring-data-jpa | **`implementation`** ← **추가됨** | E2E 테스트가 Repository.findById 호출하기 위해 직접 명시 |

### 3.2 빌드 그래프

```
       common              ← 이벤트 버스, Outbox, 기반
       │ (api로 노출 안 함)
       │
       ▼
     ┌─order─ ┐ ┌─payment─┐ ┌─notification─┐
     │       │ │         │ │              │
     └───────┘ └─────────┘ └──────────────┘
       │           │              │
       └───────────┼──────────────┘
                   ▼
                 app    ← 모든 도메인 조립 + E2E 테스트
                        + spring-data-jpa 직접 명시 (Repository 사용)
                        + spring-boot-starter-webmvc (실행 가능 jar)
```

### 3.3 모듈 간 통신은 이벤트 컨트랙트만

```
common/event/contract/
  ├ OrderCreated
  ├ OrderConfirmed
  ├ OrderConfirmedPayload
  ├ PaymentRequested
  └ PaymentCompleted
```

`order`와 `payment`는 서로 직접 의존하지 않는다. **이벤트 클래스를 `common`에 둠으로써** 두 도메인 모두 같은 컨트랙트를 보고 통신할 수 있다.

```
order ──publish(OrderCreated)──> InMemoryEventBus ──> payment (subscribe)
                                                            │
                                                            ▼
order <──publish(PaymentCompleted)── InMemoryEventBus ──── payment
```

빌드 그래프에 `order ↔ payment` 의존이 없음을 보장하는 것이 모듈러 모놀리스의 핵심.

---

## 4. 트레이드오프

### 4.1 `implementation` 기본 선택의 가치

- **캡슐화** — 모듈의 내부 구현이 외부에 새지 않음.
- **빌드 속도** — 의존성 그래프가 변경됐을 때 영향받는 모듈 수 최소화.
- **리팩토링 안전** — 내부 라이브러리 교체가 외부에 충격 없음.

### 4.2 비용

- **첫 컴파일 오류는 종종 헷갈림** — "왜 이 클래스가 안 보이지?"가 스코프 문제일 수 있음.
- **명시적 의존성 선언이 늘어남** — `app` 모듈에 직접 추가하는 의존성이 많아짐.
- **`api`를 안 쓰는 게 항상 옳지는 않음** — 진짜 외부 API 시그니처에 등장하는 타입은 `api`가 맞음.

### 4.3 언제 `api`가 맞는가

```java
// 이런 경우는 api가 자연스럽다
public interface PaymentGateway {
    PaymentResult charge(PaymentRequest request);   // 이 타입이 의존자가 다뤄야 하는 시그니처
}
```

`PaymentRequest` 같은 타입이 외부 API의 일부라면, 외부 모듈도 그 타입을 직접 다뤄야 하므로 `api` 스코프가 맞다.

이 프로젝트의 `OrderRepository`는 어떤가? `JpaRepository`는 spring-data가 제공하는 **외부 라이브러리 인터페이스**고, `OrderRepository`를 다루는 외부 모듈(app)은 어차피 자기가 JPA를 사용하는 책임을 진다. 그래서 **app이 직접 명시**하는 게 맞다.

---

## 5. 자주 만나는 함정

### 5.1 함정 1 — `implementation`이라 안 보이는데 IDE는 OK

IntelliJ는 종종 같은 프로젝트의 모듈 의존성을 폭넓게 보여줘서, `implementation` 스코프로도 의존자에 자동완성이 잡힌다. **컴파일 시점에 처음으로** 진짜 오류가 드러난다.

→ `./gradlew build`를 자주 돌려라. IDE 표시를 믿지 마라.

### 5.2 함정 2 — `api`로 풀면 모든 게 풀림

컴파일 오류가 나면 일단 `api`로 바꾸는 유혹이 있다. **하지 마라.** 캡슐화가 무너지고 빌드 그래프가 점점 거미줄이 된다.

→ "어떤 모듈이 이 의존성을 진짜로 사용하는가"를 먼저 묻고, 그 모듈에 직접 추가.

### 5.3 함정 3 — Spring Boot의 transitive 의존성에 의존

`spring-boot-starter-web`을 추가하면 jackson, tomcat 등이 따라온다. 이걸 코드에서 직접 import하는 일이 잦다.

→ 직접 import할 거면 명시적으로 의존성 선언. starter는 starter일 뿐 "내가 만든 모듈의 API"가 아니다.

### 5.4 함정 4 — 테스트가 모듈 경계를 깨뜨림

```java
// payment/src/test/.../PaymentEventHandlerTest.java
import com.hybrid.order.service.OrderService;   // ❌ payment는 order를 의존하지 않음
```

테스트 코드도 모듈 경계를 지켜야 한다. 도메인 간 통합 검증은 `app/integration/`에 두는 게 맞다.

→ 단위/슬라이스 테스트는 자기 모듈 안에서 자기 책임만, 통합 테스트는 조립체 모듈에서.

---

## 6. 디버깅 도구

### 6.1 의존성 트리 보기

```bash
./gradlew :app:dependencies                    # app 모듈의 모든 의존성 트리
./gradlew :app:dependencies --configuration compileClasspath   # 컴파일 타임만
./gradlew :app:dependencies --configuration runtimeClasspath   # 런타임만
```

`compileClasspath` 출력에 `JpaRepository`가 보이면 `findById` 호출이 가능하다. 안 보이면 위 오류가 난다.

### 6.2 모듈 그래프

```bash
./gradlew projects                             # 모듈 트리
./gradlew :app:dependencyInsight --dependency spring-data-jpa
                                               # 특정 의존성이 어디서 들어왔는지
```

### 6.3 의존성 변경 후 캐시 갱신

```bash
./gradlew build --refresh-dependencies        # 메타데이터 강제 재해석
```

IntelliJ에선 추가로 `File > Invalidate Caches`.

---

## 7. 의사결정 가이드

```
새 의존성을 추가할 때:

  이 의존성을 사용하는 게 누구인가?
  ├ 이 모듈 안에서만           → implementation
  ├ 이 모듈의 공개 API에 등장   → api (드물다)
  ├ 어노테이션 프로세서 / 빌드 도구 → compileOnly
  └ 런타임에만 필요 (드라이버 등) → runtimeOnly

기존 모듈을 의존할 때:

  나는 그 모듈의 무엇을 쓰는가?
  ├ 그 모듈이 노출한 API만        → implementation project(':...')
  └ 그 모듈을 통한 transitive 의존성도 직접 다룬다
                                  → 그 transitive 의존성도 내가 직접 명시해야 함

컴파일 오류가 났을 때:

  ❌ 충동적으로 api로 바꾸지 말 것
  ✅ 어떤 모듈이 진짜로 그 클래스를 다루는지 먼저 식별
  ✅ 그 모듈에 명시적으로 추가
```

---

## 8. 한 줄 요약

> **`implementation`은 "이 의존성은 내 비즈니스다, 너에게 강요하지 않는다"의 약속**, **`api`는 "이 의존성은 우리 둘의 약속의 일부다"의 선언**.
> 멀티모듈에서는 기본을 `implementation`으로 두고, **의존성을 진짜로 사용하는 모듈에 직접 명시**한다. `api`는 진짜 공개 API에만 — 컴파일 오류를 우회하려는 도구가 아니다.
> 빌드 그래프가 곧 캡슐화 다이어그램이다.

---

## 9. 함께 보면 좋은 자료

- 공식: [Java Library Plugin — implementation vs api](https://docs.gradle.org/current/userguide/java_library_plugin.html#sec:java_library_separation)
- 공식: [Dependency Configurations](https://docs.gradle.org/current/userguide/dependency_configurations.html)
- `docs/study/event-store-vs-simple-bus.md` — 모듈 경계를 이벤트 컨트랙트로 분리하는 사상
- `docs/phase-3-tdd.md` Step 6 — ArchUnit으로 빌드 외 패키지 경계를 강제

---

## 10. 이 프로젝트에서 마주친 사례 회고

| 사례 | 증상 | 원인 | 해결 |
|------|------|------|------|
| `Phase1E2ETest`의 `findById` 컴파일 실패 | `JpaRepository`가 안 보인다는 오류 | order의 spring-data-jpa가 `implementation` 스코프 → app에 노출 안 됨 | `app/build.gradle`에 `spring-data-jpa` 직접 추가 |
| `PaymentEventHandlerTest`가 `OrderService` import 실패 | 컴파일 자체가 안 됨 | payment는 order를 의존하지 않음 (모듈 경계 위반) | 이벤트를 `common.event.contract`로 옮기고, 테스트는 `eventBus.publish(new OrderCreated(...))`로 재작성 |
| `OrderEventHandlerTest`가 같은 패턴 | 동일 | 동일 | 동일 패턴으로 PaymentCompleted를 직접 발행해 테스트 |

세 사례의 공통점: **빌드 시스템이 모듈 설계의 정직한 거울 역할을 했다**. 컴파일 오류가 나는 건 곧 설계가 일관되지 않다는 신호다. 우회보다 **올바른 모듈 위치**를 찾는 게 맞다.
