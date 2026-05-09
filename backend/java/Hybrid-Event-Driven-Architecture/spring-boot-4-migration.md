# Spring Boot 4.0 — 모듈화로 인한 패키지·스타터 이동 정리

> 이 프로젝트가 Spring Boot 4.0을 사용하면서 **3.x와 다르게 import해야 했던 어노테이션·클래스·스타터** 변경을 정리한 학습 노트.
> 검색·공식 문서로 확인한 사실만 담는다 (추측 금지).

---

## 0. 왜 바뀌었는가 — Spring Boot 4.0 모듈화

Spring Boot 3.x까지는 거대한 두 jar가 거의 모든 자동 구성을 들고 있었다.

```
spring-boot-autoconfigure         ← 운영 자동 구성의 깡통 (수십 개 모듈 한 곳)
spring-boot-test-autoconfigure    ← 테스트 자동 구성의 깡통 (모든 슬라이스 어노테이션)
```

이게 컸던 이유는 단순함이지만, 단점도 있었다.

- 작은 앱도 거대 jar 전체를 끌고 옴.
- "어떤 모듈이 어떤 자동 구성을 책임지는가"가 코드만 봐선 안 보임.
- 모듈별 의존을 분리하기 어려움.

**Spring Boot 4.0**은 이 두 jar를 **모듈별 jar로 쪼갰다**. 패키지/스타터 이름이 일제히 바뀐 이유다.

```
[3.x]                              [4.0]
spring-boot-autoconfigure       →  spring-boot-<module>      (예: spring-boot-webmvc, spring-boot-data-jpa)
spring-boot-test-autoconfigure  →  spring-boot-<module>-test (예: spring-boot-webmvc-test)
spring-boot-starter-web         →  spring-boot-starter-webmvc
spring-boot-starter-test        →  여전히 존재 (일반 베이스) + spring-boot-starter-<module>-test 추가
```

**일반 패키지 패턴**:

```
[3.x] org.springframework.boot.test.autoconfigure.<area>.<class>
[4.0] org.springframework.boot.<module>.test.autoconfigure.<class>
```

> "자동 구성과 그 자동 구성의 테스트 지원이 같은 모듈로 묶인다"는 사상.

---

## 1. 테스트 어노테이션 패키지 이동

### 1.1 변경된 것들

| 어노테이션 | Spring Boot 3.x | Spring Boot 4.0 |
|----------|----------------|----------------|
| `@WebMvcTest` | `org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest` | **`org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest`** |
| `@DataJpaTest` | `org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest` | **`org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest`** |
| `@JsonTest` (예상) | `org.springframework.boot.test.autoconfigure.json.JsonTest` | `org.springframework.boot.<json-module>.test.autoconfigure.JsonTest` |
| `@JdbcTest` (예상) | `org.springframework.boot.test.autoconfigure.jdbc.JdbcTest` | `org.springframework.boot.<jdbc-module>.test.autoconfigure.JdbcTest` |

> 표의 (예상)으로 표시한 것들은 같은 모듈화 패턴이 일관 적용됐다는 전제이지만, 실제 사용 시점에 IDE 자동완성으로 정확한 경로를 확인하길 권장.

### 1.2 변경 안 된 것들

| 어노테이션 | 위치 |
|----------|------|
| `@SpringBootTest` | `org.springframework.boot.test.context.SpringBootTest` (코어 잔류) |
| `@SpringBootApplication` | `org.springframework.boot.autoconfigure.SpringBootApplication` |
| `@AutoConfigureMockMvc` | (코어 잔류로 추정) |

`@SpringBootTest`처럼 모듈 비종속인 코어 어노테이션은 그대로다.

---

## 2. `@MockBean` → `@MockitoBean` (별개의 변경)

이건 4.0 모듈화와 별개 이슈지만 같이 마주치는 변경.

| | Spring Boot ≤ 3.3 | Spring Boot 3.4+ / 4.0 |
|--|------------------|----------------------|
| 어노테이션 | `@MockBean` | `@MockitoBean` |
| 패키지 | `org.springframework.boot.test.mock.mockito.MockBean` | `org.springframework.test.context.bean.override.mockito.MockitoBean` |
| 소속 | Spring Boot Test | **Spring Framework Test** (코어로 승격) |
| 4.0 상태 | **제거** | 표준 |

### 2.1 새 패밀리

Spring 6.2부터 `org.springframework.test.context.bean.override` 패키지에 새 어노테이션 패밀리:

| 어노테이션 | 용도 |
|-----------|------|
| `@MockitoBean` | Mockito mock으로 빈 교체 (`@MockBean` 대체) |
| `@MockitoSpyBean` | Mockito spy (`@SpyBean` 대체) |
| `@TestBean` | 사용자가 직접 만든 빈으로 교체 |

### 2.2 왜 옮겼나

- **Spring Boot 종속 제거** — 일반 Spring Framework 프로젝트도 사용 가능.
- **컨텍스트 캐시 효율 개선** — `@MockBean`이 컨텍스트를 무효화하던 문제 완화.
- **확장성** — Mockito 외 다른 모킹 전략 통합 가능.

---

## 3. 스타터 이름 변경

### 3.1 정식 스타터

| 3.x | 4.0 | 비고 |
|-----|-----|------|
| `spring-boot-starter-web` | **`spring-boot-starter-webmvc`** | Servlet 기반 MVC 명시 |
| `spring-boot-starter-data-jpa` | 동일 | |
| `spring-boot-starter-actuator` | 동일 | |
| `spring-boot-starter` | 동일 | 코어 |

### 3.2 새 패턴 — 모듈별 테스트 스타터

각 정식 스타터마다 짝이 되는 테스트 스타터가 추가됐다.

```
spring-boot-starter-webmvc       ↔  spring-boot-starter-webmvc-test
spring-boot-starter-data-jpa     ↔  spring-boot-starter-data-jpa-test (예상)
spring-boot-starter-data-redis   ↔  spring-boot-starter-data-redis-test (예상)
```

**기본 사용법**:

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webmvc'

    // 일반 테스트 베이스 (JUnit, AssertJ, Mockito) — 그대로 유지
    testImplementation 'org.springframework.boot:spring-boot-starter-test'

    // WebMvc 테스트(@WebMvcTest, MockMvc) 전용 — 새로 필요
    testImplementation 'org.springframework.boot:spring-boot-starter-webmvc-test'
}
```

`spring-boot-starter-test`는 4.0에서도 살아남았다 — 다만 슬라이스 테스트 기능은 모듈별 스타터로 분리됐다.

---

## 4. 기타 변경

### 4.1 Jackson 관련 어노테이션 이름 변경

| 3.x | 4.0 |
|-----|-----|
| `@JsonComponent` | `@JacksonComponent` |
| `@JsonMixin` | `@JacksonMixin` |
| `JsonObjectSerializer` | `ObjectValueSerializer` |
| `JsonValueDeserializer` | `ObjectValueDeserializer` |
| `Jackson2ObjectMapperBuilderCustomizer` | `JsonMapperBuilderCustomizer` |

"Json"은 표준 표기, "Jackson"은 라이브러리 이름. 표준과 라이브러리를 구별하려는 의도로 보인다.

### 4.2 코어 클래스 이동

| 클래스 | 3.x | 4.0 |
|------|-----|-----|
| `BootstrapRegistry` | `org.springframework.boot` | `org.springframework.boot.bootstrap` |
| `EnvironmentPostProcessor` | `org.springframework.boot.env` | `org.springframework.boot` |
| `@PropertyMapping` | `org.springframework.boot.test.autoconfigure.properties` | `org.springframework.boot.test.context` |
| `EntityScan` | (구 위치) | `org.springframework.boot.persistence.autoconfigure` |
| `TestRestTemplate` | (구 위치) | `org.springframework.boot.resttestclient` (deprecated 예정) |

### 4.3 `RestTestClient` 추가

기존 `TestRestTemplate`을 대체할 새 fluent API. `MockMvc`와 실제 서버 양쪽에서 동작.

```java
RestTestClient client = RestTestClient.bindToController(new OrderController(...));

client.post().uri("/api/orders")
    .contentType(MediaType.APPLICATION_JSON)
    .body("{...}")
    .exchange()
    .expectStatus().isCreated();
```

---

## 5. 이 프로젝트에서 적용한 변경

### 5.1 빌드 파일

```diff
// order/build.gradle, payment/build.gradle, notification/build.gradle, app/build.gradle
dependencies {
-   implementation 'org.springframework.boot:spring-boot-starter-web'
+   implementation 'org.springframework.boot:spring-boot-starter-webmvc'
+
+   testImplementation 'org.springframework.boot:spring-boot-starter-webmvc-test'
}
```

`common/build.gradle`은 web을 안 쓰므로 변경 없음.
root `build.gradle`의 `spring-boot-starter-test`는 그대로 유지 (일반 테스트 베이스).

### 5.2 테스트 코드 import

```diff
// order/src/test/java/com/hybrid/order/web/OrderControllerTest.java
- import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
+ import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
- import org.springframework.boot.test.mock.mockito.MockBean;
+ import org.springframework.test.context.bean.override.mockito.MockitoBean;

  @WebMvcTest(OrderController.class)
  class OrderControllerTest {
      @Autowired MockMvc mockMvc;
-     @MockBean OrderService orderService;
+     @MockitoBean OrderService orderService;
  }
```

```diff
// notification/src/test/java/com/hybrid/notification/inbox/InboxConsumerTest.java
- import org.springframework.boot.test.mock.mockito.MockBean;
+ import org.springframework.test.context.bean.override.mockito.MockitoBean;

- @MockBean NotificationService notificationService;
+ @MockitoBean NotificationService notificationService;
```

### 5.3 영향 받지 않은 것

- `@SpringBootTest`, `@Container`, `@Testcontainers`, `@DynamicPropertySource` 등은 변경 없음.
- 도메인 코드(`@Entity`, `@Service`, `@Component`, `@Transactional`) 변경 없음.
- `MockMvc`, `MockMvcRequestBuilders`, `MockMvcResultMatchers` 등 import 변경 없음.

---

## 6. 마이그레이션 작업 순서

3.x 프로젝트를 4.0으로 옮길 때 권장 순서:

1. **빌드 의존성 우선**: `spring-boot-starter-web` → `spring-boot-starter-webmvc`로 변경하고 모듈별 테스트 스타터 추가.
2. **컴파일 오류 따라가기**: import unresolved 에러를 IDE가 잡아주면 새 경로로 교체. `@MockBean` → `@MockitoBean`도 같이.
3. **OpenRewrite 자동화 검토**: Spring 팀이 제공하는 `org.openrewrite.java.spring.boot4.UpgradeSpringBoot_4_0` 레시피로 자동 마이그레이션 가능 (수동 작업 90%↑ 절감).
4. **테스트 실행**: 컴파일 통과 후 실제 동작 확인. `@WebMvcTest`의 시큐리티 동작이 미묘하게 바뀌었다는 보고가 있어 (GitHub Issue #36423) 슬라이스 테스트 회귀가 있으면 별도 점검.

---

## 7. 자주 마주칠 함정

### 7.1 IDE 자동 import가 옛 경로 추천

특히 IntelliJ가 캐시된 인덱스를 기준으로 옛 경로(`boot.test.autoconfigure.web.servlet.WebMvcTest`)를 추천하는 경우가 있다. 새로 의존성을 추가한 직후엔:

```
File > Invalidate Caches > Invalidate and Restart
또는
./gradlew build --refresh-dependencies
```

### 7.2 `spring-boot-starter-test`만 추가하고 `spring-boot-starter-webmvc-test`를 빼먹음

`@WebMvcTest`만 쓰고 싶다면 둘 다 필요하다. **일반 테스트 베이스(`spring-boot-starter-test`) + 모듈별 슬라이스 스타터(`spring-boot-starter-webmvc-test`)** 조합.

### 7.3 `@MockBean` 자동 import가 깨짐

Spring Boot 4.0에서 `@MockBean`이 **완전히 제거**됐기 때문에 IDE 자동 import에서 안 나온다. `@MockitoBean`으로 직접 입력해야 함. import 경로도 `org.springframework.test.context.bean.override.mockito` (boot가 아니라 test).

---

## 8. 한 줄 요약

> **Spring Boot 4.0은 거대 jar(`spring-boot-autoconfigure`, `spring-boot-test-autoconfigure`)를 모듈별로 쪼갠 대규모 모듈화 릴리스다.** 결과로 `@WebMvcTest`처럼 자주 쓰는 테스트 어노테이션의 패키지가 `boot.<module>.test.autoconfigure`로 옮겨졌고, `spring-boot-starter-web`이 `spring-boot-starter-webmvc`로 이름이 바뀌었으며, 모듈별 테스트 스타터(`spring-boot-starter-webmvc-test` 등)가 새로 도입됐다. `@MockBean`은 별개 이슈로 제거되어 `@MockitoBean`이 표준이 되었다.

---

## 9. 함께 보면 좋은 자료

- 공식 [Spring Boot 4.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide)
- 공식 블로그 [Modularizing Spring Boot (2025-10-28)](https://spring.io/blog/2025/10/28/modularizing-spring-boot/)
- [What's New for Testing in Spring Boot 4 and Spring Framework 7 — rieckpil.de](https://rieckpil.de/whats-new-for-testing-in-spring-boot-4-0-and-spring-framework-7/)
- [Spring Boot 4 Modularization: Fix Auto-Configuration Issues After Upgrading — danvega.dev](https://www.danvega.dev/blog/spring-boot-4-modularization)
- API 문서:
  - [@WebMvcTest (Spring Boot 4.0)](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/webmvc/test/autoconfigure/WebMvcTest.html)
  - [@DataJpaTest (Spring Boot 4.0)](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/data/jpa/test/autoconfigure/DataJpaTest.html)
  - [@SpringBootTest (Spring Boot 4.0)](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/test/context/SpringBootTest.html)

---

## 10. 학습 노트의 자기 정정

> 이 프로젝트의 페이즈 문서를 처음 작성할 때 **`@WebMvcTest`의 import 경로를 Spring Boot 3.x 기준으로 잘못 적었다**. 사용자가 IDE에서 "심볼 'web'을 해결할 수 없습니다" 오류를 보고 지적해주어 검색으로 확인 후 수정했다.
>
> 교훈: **최신 라이브러리(특히 메이저 버전이 막 올라간 경우)의 패키지/어노테이션 위치는 추측 금지, 무조건 공식 API 문서 또는 마이그레이션 가이드로 확인.** Java/Spring 생태계는 모듈화·deprecation·rename이 자주 일어나는 편이라 학습 시점의 정보가 빠르게 낡는다.
