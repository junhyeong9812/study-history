# 멀티모듈에서 테스트 헬퍼 공유하기 — `java-test-fixtures` vs 별도 모듈

> 한 모듈에서 만든 테스트 베이스 클래스(`KafkaIntegrationTestBase` 같은)를 다른 모듈의 테스트가 어떻게 쓰는가.
> Gradle의 표준 해결책 세 가지를 비교하고, 이 프로젝트가 왜 어느 단계에서 무엇을 선택했는지 정리한 학습 노트.

---

## 0. 풀어야 할 문제

```
[원하는 그림]
common/src/?/.../KafkaIntegrationTestBase   ← 공통 테스트 베이스
                          │
                          ▼ 여러 모듈에서 상속
order/src/test/.../OrderConfirmOutboxTest extends KafkaIntegrationTestBase
notification/src/test/.../InboxConsumerTest extends KafkaIntegrationTestBase
app/src/test/.../Phase1E2ETest extends KafkaIntegrationTestBase
```

다른 모듈에서 같은 베이스를 써야 한다. 그런데:

```
common/src/test/.../KafkaIntegrationTestBase    ← 여기 두면?
                                                  ❌ 다른 모듈에서 안 보임
```

`implementation project(':common')`은 **`src/main`만** 노출한다. **`src/test`는 모듈 경계를 넘지 못한다** — Gradle의 기본 정책.

### 0.1 왜 그런가

테스트 코드는 **운영 jar에 들어가서는 안 되는 것들**(Mockito stub, 픽스처 데이터, Testcontainers 헬퍼 등)을 포함한다. 만약 `src/test`가 자동으로 다른 모듈에 노출된다면:

- 운영 코드와 테스트 코드의 경계가 흐려짐.
- 무거운 테스트 의존성(Testcontainers, JUnit, AssertJ)이 운영 클래스패스에 끌려옴.
- 캡슐화 의지가 약해짐.

그래서 Gradle은 의도적으로 **테스트 소스셋을 모듈 간에 자동 공유하지 않는다**. 공유하려면 명시적인 메커니즘이 필요하다.

---

## 1. 세 가지 표준 해결책

### 1.1 옵션 A — 헬퍼를 `src/main`으로 옮김 (간단, 누수 위험)

```
common/src/main/java/.../KafkaIntegrationTestBase
                              │
                              ▼
다른 모듈은 그냥 implementation project(':common')으로 본다
```

**장점**:
- 가장 단순. 추가 설정 없음.
- 모든 모듈에 자동 노출.

**단점 (큼)**:
- **테스트 헬퍼가 운영 jar에 포함됨**. 운영 환경에 Testcontainers, Mockito 등 의도치 않은 의존성 존재.
- 의존성도 `implementation 'org.testcontainers:...'`로 잡혀서 운영 jar 크기 증가.
- "테스트 코드 vs 운영 코드"의 경계가 무너짐.

> **권장 안 함**. 의도적 결정이 아니면 피해야 함.

### 1.2 옵션 B — `java-test-fixtures` 플러그인 (이 프로젝트의 선택)

Gradle의 표준 플러그인. **`src/testFixtures/java`** 라는 새 소스셋을 도입한다.

```
common/
  src/main/java/                    ← 운영 코드
  src/test/java/                    ← common 자기 테스트
  src/testFixtures/java/            ← 다른 모듈도 쓸 수 있는 테스트 헬퍼
       └ KafkaIntegrationTestBase
```

다른 모듈은 `testImplementation testFixtures(project(':common'))`로 가져온다.

```groovy
// common/build.gradle
plugins { id 'java-test-fixtures' }

dependencies {
    testFixturesApi 'org.springframework.boot:spring-boot-starter-test'
    testFixturesApi 'org.testcontainers:junit-jupiter:1.20.4'
    // ...
}

// order/build.gradle
dependencies {
    testImplementation testFixtures(project(':common'))
}
```

**장점**:
- 테스트 헬퍼임이 디렉토리 이름으로 명확.
- 운영 jar에 들어가지 않음 — Testcontainers/Mockito 의존성도 운영 클래스패스에 안 새어나감.
- Gradle 공식 메커니즘 — 빌드 도구 차원 표준.

**단점**:
- 플러그인 추가 + 별도 디렉토리 필요.
- 처음 마주치는 사람에겐 "testFixtures가 뭐지?" 학습 필요.

### 1.3 옵션 C — 별도 `test-support` 모듈

큰 프로젝트에서 자주 쓰는 패턴. **테스트 헬퍼만 들어 있는 별도 모듈**을 만든다.

```
hybrid-event-driven/
  ├ common/
  ├ order/
  ├ payment/
  ├ notification/
  ├ app/
  └ test-support/                   ← 새 모듈
      └ src/main/java/.../KafkaIntegrationTestBase
```

```groovy
// test-support/build.gradle
dependencies {
    api 'org.springframework.boot:spring-boot-starter-test'
    api 'org.testcontainers:junit-jupiter:1.20.4'
}

// order/build.gradle
dependencies {
    testImplementation project(':test-support')
}
```

> 주의: `test-support`는 운영 jar로 패키징되지 않도록 **메인 앱(`app`)이 의존하지 않게** 관리해야 한다.

**장점**:
- 가장 명확한 분리 — 한 모듈이 통째로 테스트 지원만 담당.
- 테스트 헬퍼가 점점 늘어나도 자기만의 패키지 구조·의존성·빌드 설정 가능.
- 다른 프로젝트로 분리 가능.

**단점**:
- 모듈이 하나 늘어남. 작은 프로젝트엔 과함.
- `testImplementation`으로 명시적으로 가져가야 함 (실수로 운영에 의존하면 위험).

---

## 2. `java-test-fixtures` 자세히 보기

이 프로젝트의 선택이라 좀 더 깊이 다룬다.

### 2.1 적용

```groovy
// common/build.gradle
plugins {
    id 'java'
    id 'java-test-fixtures'
}
```

이 플러그인은 자동으로:
- `src/testFixtures/java/`, `src/testFixtures/resources/` 디렉토리 생성(존재하면 사용).
- `testFixtures` 소스셋 등록.
- 의존성 컨피규레이션 5개 추가.

### 2.2 의존성 컨피규레이션

| 컨피규레이션 | 의미 | 비유 |
|------------|------|------|
| `testFixturesApi` | testFixtures가 사용 + **외부에 노출** | `api`의 testFixtures 버전 |
| `testFixturesImplementation` | testFixtures가 사용, 외부엔 노출 안 됨 | `implementation`의 testFixtures 버전 |
| `testFixturesCompileOnly` | 컴파일 시점만 | `compileOnly` 짝 |
| `testFixturesRuntimeOnly` | 런타임만 | `runtimeOnly` 짝 |
| `testFixturesAnnotationProcessor` | 어노테이션 프로세서 | |

이 프로젝트의 `common/build.gradle`:

```groovy
testFixturesApi 'org.springframework.boot:spring-boot-starter-test'
testFixturesApi 'org.testcontainers:junit-jupiter:1.20.4'
testFixturesApi 'org.testcontainers:postgresql:1.20.4'
testFixturesApi 'org.testcontainers:kafka:1.20.4'
```

`api`로 노출하는 이유: 사용자(`order`, `payment` 등)도 `@SpringBootTest`, `@Testcontainers`, `KafkaContainer` 등을 직접 import해야 하기 때문. **테스트 헬퍼의 시그니처에 등장하는 타입은 의존자가 다뤄야 한다 → api**.

### 2.3 사용자 측 선언

```groovy
// order/build.gradle, payment/build.gradle, notification/build.gradle, app/build.gradle
dependencies {
    testImplementation testFixtures(project(':common'))
}
```

`testFixtures(...)` 헬퍼는 Gradle 빌트인. "이 모듈의 testFixtures 결과물을 가져와라"는 의미.

### 2.4 동시 의존 가능

같은 모듈을 두 가지 방식으로 의존할 수 있다.

```groovy
// order/build.gradle
implementation project(':common')                            // 운영 코드
testImplementation testFixtures(project(':common'))          // 테스트 헬퍼
```

별개의 컨피규레이션이라 충돌 없음.

---

## 3. 의사결정 가이드

```
공유할 테스트 헬퍼가 생겼다
       │
       ├ 헬퍼가 1~3개, 작고 거의 안 늘 것
       │   → 옵션 B (java-test-fixtures)
       │
       ├ 헬퍼가 점점 늘고 있다 (5+)
       │   → 옵션 B 유지 가능, 또는 옵션 C로 졸업
       │
       ├ 헬퍼가 자기만의 빌드 설정·하위 의존성을 가짐
       │   (예: 별도 Mock 서버 띄우기, 자체 테스트 데이터 생성기)
       │   → 옵션 C (test-support 모듈)
       │
       ├ 여러 프로젝트(저장소)에 걸쳐 공유하고 싶음
       │   → 옵션 C + 결과물을 Maven 저장소에 발행
       │
       └ 헬퍼가 운영 코드와 어차피 같이 쓰여야 함 (드물다)
           → 옵션 A (src/main으로 이동) — 의도적 결정인 경우만
```

---

## 4. 이 프로젝트의 진화 시나리오

### 4.1 현재 — 옵션 B

```
common/src/testFixtures/java/com/hybrid/common/support/
  └ KafkaIntegrationTestBase
```

Phase 1/2/3 통합 테스트가 모두 이 베이스를 상속.
헬퍼가 1개라 옵션 B가 적절.

### 4.2 가까운 미래 — 헬퍼 추가

Phase 2/3 진행하면 헬퍼가 늘 가능성:

```
common/src/testFixtures/java/com/hybrid/common/support/
  ├ KafkaIntegrationTestBase
  ├ KafkaTestProducer        ← Phase 2에서 도입
  ├ KafkaTestConsumer
  └ KafkaProducerStub        ← 실패 주입
```

이 정도까지는 옵션 B로 충분.

### 4.3 만약 다음이 생긴다면 → 옵션 C로 이행

```
common/src/testFixtures/...
  ├ KafkaIntegrationTestBase
  ├ KafkaTestProducer
  ├ KafkaTestConsumer
  ├ KafkaProducerStub
  ├ ArchUnitRules            ← Phase 3 ArchUnit 룰 모음
  ├ ChaosScenarioRunner      ← 부하·장애 시뮬레이터
  ├ FakeEmailGateway         ← Phase 3 외부 시스템 페이크
  ├ TestDataBuilder          ← 도메인 객체 빌더
  └ ...
```

이 시점이면 **`test-support` 모듈**로 분리한다:

```
hybrid-event-driven/
  ├ common/
  ├ order/
  ├ payment/
  ├ notification/
  ├ app/
  └ test-support/             ← 신규
       ├ build.gradle
       └ src/main/java/.../
```

이행 비용은 작다 — 디렉토리 이동 + build.gradle 수정. 미리 준비할 건 없음.

### 4.4 옵션 C로 이행 시 빌드 변경

```groovy
// settings.gradle
include 'common', 'order', 'payment', 'notification', 'app', 'test-support'

// test-support/build.gradle
dependencies {
    api 'org.springframework.boot:spring-boot-starter-test'
    api 'org.testcontainers:junit-jupiter:1.20.4'
    // ...
    api project(':common')   // 도메인 클래스 참조 가능
}

// order/build.gradle
dependencies {
    implementation project(':common')
    // testImplementation testFixtures(project(':common'))   ← 제거
    testImplementation project(':test-support')               ← 추가
}

// common/build.gradle
// java-test-fixtures 플러그인과 testFixturesApi 의존성 제거
```

---

## 5. 자주 만나는 함정

### 5.1 함정 1 — testFixtures 의존성을 `implementation`으로 잡음

```groovy
// ❌ 잘못된 사용
testFixturesImplementation 'org.testcontainers:junit-jupiter:1.20.4'
```

이러면 다른 모듈이 `testFixtures(project(':common'))`로 가져왔을 때 **`@Container`, `KafkaContainer`** 같은 타입이 안 보인다. 사용자 측이 실제로 import해야 하는 타입은 `testFixturesApi`로 노출해야 한다.

규칙: **헬퍼 클래스의 public 시그니처에 등장하는 타입은 `testFixturesApi`**, 내부 구현용은 `testFixturesImplementation`.

### 5.2 함정 2 — `test-support` 모듈을 `app`이 implementation으로 의존

```groovy
// app/build.gradle
implementation project(':test-support')   // ❌ 운영 jar에 테스트 헬퍼 들어감
```

테스트 헬퍼 모듈은 **반드시 `testImplementation`** 으로만. 사고 방지를 위해 `test-support`의 패키지 이름에 `test`를 명시적으로 두면 좋다 (`com.hybrid.testsupport.*`).

### 5.3 함정 3 — IDE가 testFixtures 디렉토리를 못 인식

특히 IntelliJ에서 처음 도입 시:
- `src/testFixtures/java`가 회색(인덱싱 안 됨) → Gradle Reload 필요.
- 자동완성이 안 됨 → File > Invalidate Caches.

```bash
./gradlew --refresh-dependencies build
# IDE: View > Tool Windows > Gradle > Refresh
```

### 5.4 함정 4 — 순환 의존

```
test-support → common      (도메인 객체 빌더 만들려고)
common  → test-support  (??)   ← ❌ 순환
```

`common`이 `test-support`를 의존하는 일은 없어야 한다. test-support는 항상 **단방향, 도메인 모듈에 종속**되는 위치.

### 5.5 함정 5 — testFixtures가 다른 모듈의 testFixtures를 의존

```groovy
// notification/build.gradle (testFixtures 사용 시)
testFixturesImplementation testFixtures(project(':common'))
```

가능하다. 다만 빌드 그래프가 복잡해지므로 헬퍼가 진짜 단계별로 쌓일 때만 사용.

---

## 6. 비교 — 다른 도구·환경에서의 같은 패턴

### 6.1 Maven에서

Maven은 `<scope>test</scope>`만 있고 testFixtures 같은 표준 메커니즘이 약하다. 보통:

- `<type>test-jar</type>` (테스트 jar 별도 발행) — 가능하지만 보일러플레이트 큼.
- 별도 `*-tests` 모듈 — Maven에선 사실상 표준.

Gradle의 `java-test-fixtures`는 Maven 사용자에겐 일종의 부러움 대상.

### 6.2 Kotlin Multiplatform / 안드로이드

`androidTest` 소스셋이 비슷한 위치 — 메인과 분리된 별도 소스셋 + 자체 의존성 컨피규레이션. 사상은 같다.

### 6.3 다른 빌드 도구 — Bazel / Buck

세분화된 타겟 단위로 의존성 관리 → testFixtures 개념이 자연스럽게 표현됨 (`java_library`로 테스트 헬퍼 타겟 만들고, 다른 `java_test`가 deps로 가져감).

---

## 7. 한 줄 요약

> **Gradle은 모듈 간 테스트 소스 공유를 의도적으로 막아 운영 jar 누수를 방지한다.** 공유가 필요하면 `java-test-fixtures` 플러그인으로 별도 소스셋(`src/testFixtures/java`)을 만들어 명시적으로 노출하는 게 표준이다. 헬퍼가 점점 늘어 자기 빌드 설정과 하위 의존성을 가져야 할 정도가 되면 별도 `test-support` 모듈로 졸업한다.
> **이 프로젝트의 위치**: 헬퍼 1개(`KafkaIntegrationTestBase`) → testFixtures로 충분. Phase 3에서 ArchUnit·Chaos·Fake 등이 늘면 `test-support`로 이행.

---

## 8. 함께 보면 좋은 자료

- 공식: [Gradle Java Test Fixtures](https://docs.gradle.org/current/userguide/java_testing.html#sec:java_test_fixtures)
- 공식: [Java Library Plugin — api/implementation](https://docs.gradle.org/current/userguide/java_library_plugin.html)
- `docs/study/gradle-dependency-scopes.md` — testFixturesApi vs testFixturesImplementation의 사상적 뿌리
- `docs/study/testcontainers.md` — `KafkaIntegrationTestBase`가 무엇을 하는지
- `docs/study/spring-test-property-injection.md` — 베이스 클래스가 동적 프로퍼티를 어떻게 주입하는지

---

## 9. 이 프로젝트에서 마주친 사례 회고

| 사례 | 증상 | 원인 | 해결 |
|------|------|------|------|
| `OrderConfirmOutboxTest`가 `KafkaIntegrationTestBase`를 못 찾음 | "심볼 'KafkaIntegrationTestBase'을(를) 해결할 수 없습니다" | `common/src/test/`에 있어 다른 모듈에 노출 안 됨 | `common/src/testFixtures/`로 이동 + `java-test-fixtures` 플러그인 + 의존자에 `testImplementation testFixtures(project(':common'))` |

이 한 사례가 모듈러 모놀리스의 정직함을 또 한 번 보여줬다 — **빌드 그래프가 곧 캡슐화 다이어그램**이라는 사실. 테스트 헬퍼도 명시적으로 노출하지 않으면 새지 않는다.

---

## 10. 실무 체크리스트

새 프로젝트에서 테스트 헬퍼 공유를 도입할 때 자문할 것:

- [ ] 헬퍼는 한 곳에서 정의되어 모든 모듈이 같은 기반을 쓰는가?
- [ ] 헬퍼가 운영 jar에 새어나가지 않는가? (jar 압축 해제해서 직접 확인 가능)
- [ ] 헬퍼의 의존성(Mockito, Testcontainers 등)이 운영 클래스패스에 없는가?
- [ ] 헬퍼의 시그니처에 등장하는 타입이 `api`로 노출되어 있어 의존자가 자연스럽게 import할 수 있는가?
- [ ] testFixtures 디렉토리가 IDE에 잘 인식되는가? (Gradle reload + caches 정리)
- [ ] 헬퍼가 점점 늘면 `test-support` 모듈로 졸업할 시점인가?
