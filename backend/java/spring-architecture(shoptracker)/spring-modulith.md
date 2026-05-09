# Spring Modulith

> 모듈러 모놀리스를 만드는 도구. 모듈 경계 검증 + 이벤트 지속성 + 문서 자동 생성.

## 개념

마이크로서비스 없이 **하나의 배포 단위 안에 잘 분리된 모듈** 을 두는 아키텍처가 "모듈러 모놀리스". Spring Modulith가 이걸 강제하고 검증한다.

이 프로젝트의 톱레벨 패키지 = 모듈:
- `subscription`, `orders`, `payments`, `shipping`, `tracking`, `shared`

각 모듈의 **공개 API** 는 톱레벨 패키지에 둔 클래스, **내부 구현** 은 하위 패키지(`internal`, `domain`, `adapter` 등)에 둔다. 다른 모듈이 내부 구현을 import하면 컴파일은 되지만 검증 테스트가 실패한다.

## 핵심 기능

### 1. 모듈 경계 검증

```java
// src/test/java/com/shoptracker/ModuleStructureTest.java
@Test
void verifyModuleStructure() {
    ApplicationModules modules = ApplicationModules.of(ShopTrackerApplication.class);
    modules.verify();
}
```

이게 검증하는 것:
- 모듈 간 순환 의존성 (A→B→A 금지)
- 다른 모듈의 `internal` 패키지 직접 import 금지
- `@NamedInterface`로 명시한 공개 패키지만 외부에서 사용

이 프로젝트의 `payments/internal/PaymentsPolicyConfig`는 다른 모듈에서 import 못 함. 의도된 캡슐화.

### 2. `@ApplicationModuleListener`

```java
@Component
public class PaymentEventHandler {
    @ApplicationModuleListener
    public void on(OrderCreatedEvent event) { ... }
}
```

기본 `@EventListener` 위에 다음을 자동 적용:
- `@TransactionalEventListener(AFTER_COMMIT)`: 발행자 커밋 후 실행
- `@Async`: 별도 스레드 (트랜잭션 격리)
- `@Transactional(REQUIRES_NEW)`: 리스너 자체 트랜잭션
- **이벤트 지속성**: 실패 시 재시도 가능

### 3. 이벤트 지속성 (Event Publication Registry)

`spring-modulith-events-jpa` 같은 모듈을 추가하면 발행된 이벤트가 `event_publication` 테이블에 저장된다.

```
이벤트 발행
  → event_publication 에 (id, event_type, listener, serialized_event, publication_date) INSERT
  → 리스너 호출
  → 성공: completion_date 업데이트
  → 실패: completion_date NULL 유지 → 재시도 가능
```

JVM이 이벤트 발행 직후 죽어도 트랜잭션이 커밋되었다면 이벤트가 DB에 남음. 재기동 시 미완료 이벤트를 재발행할 수 있다.

이 프로젝트는 현재 `spring-modulith-events-api` 만 의존성에 두고 있어 인메모리 발행. 운영용으로는 `spring-modulith-events-jpa` 추가 권장.

### 4. Scenario 테스트

```java
// PaymentsModuleTest.java
@ApplicationModuleTest
class PaymentsModuleTest {
    @Test
    void orderCreated_triggersPayment(Scenario scenario) {
        scenario.publish(new OrderCreatedEvent(...))
            .andWaitForEventOfType(PaymentApprovedEvent.class)
            .toArriveAndVerify(event -> { ... });
    }
}
```

- `@ApplicationModuleTest`: 해당 모듈만 부팅, 의존 모듈은 mock.
- `Scenario.publish().andWaitForEventOfType()`: 이벤트 흐름을 시나리오로 검증.

### 5. 문서 자동 생성

```java
new Documenter(modules)
    .writeModulesAsPlantUml()
    .writeIndividualModulesAsPlantUml();
```

`build/spring-modulith-docs/` 에 PlantUML 다이어그램 생성. 모듈 관계도가 코드에서 자동으로 도출된다.

## 모듈 메타데이터

`package-info.java` 로 모듈 이름/설명 명시:
```java
@ApplicationModule(
    displayName = "주문 모듈",
    allowedDependencies = {"shared"}
)
package com.shoptracker.orders;
```

## FastAPI 대응

| FastAPI 환경 | Spring Modulith |
|--------------|-----------------|
| 모듈 검증 = `grep` 수동 | `ApplicationModules.verify()` 자동 |
| 이벤트 영속성 = 직접 구현 | 내장 (jpa/jdbc/mongodb) |
| 모듈 다이어그램 = 직접 작성 | PlantUML 자동 |
| 시나리오 테스트 = pytest fixture 조립 | `Scenario` API |

이 프로젝트가 Spring으로 옮긴 가장 큰 이득 중 하나가 Modulith. FastAPI 버전에서 직접 만들어야 했던 outbox/event_bus/모듈 검증을 모두 프레임워크가 제공.
