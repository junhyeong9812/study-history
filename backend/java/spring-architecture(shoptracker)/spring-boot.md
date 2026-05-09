# Spring Boot 4

> 이 프로젝트는 Spring Boot 4.0.2를 사용 (`build.gradle:3`).

## 개념

Spring Boot는 Spring Framework 위에 **자동 설정(auto-configuration)**, **스타터 의존성(starters)**, **내장 서버(embedded server)**, **운영 도구(actuator)** 를 얹은 "의견을 가진(opinionated)" 프레임워크다. "config 100줄 대신 starter 1줄" 이 모토.

## 핵심 메커니즘

### 1. 스타터 (Starter Dependencies)
관련 의존성을 한 번에 끌어오는 메타 패키지.

```groovy
implementation 'org.springframework.boot:spring-boot-starter-web'
// → spring-web, spring-webmvc, embedded Tomcat, Jackson 자동 포함
```

### 2. 자동 설정 (Auto-configuration)
- 클래스패스에 H2가 있으면 → 인메모리 H2 DataSource 자동 등록
- `DataSource` 빈이 있으면 → JdbcTemplate 자동 등록
- 클래스패스에 Tomcat이 있으면 → `EmbeddedTomcat` 등록
- 동작 원리: `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 파일이 후보 설정 클래스를 나열, 각 클래스의 `@ConditionalOn*`이 활성화 여부 결정.

### 3. 내장 서버
`java -jar app.jar` 한 줄로 Tomcat 포함 단일 실행 파일. WAR 배포 불필요.

### 4. 외부화된 설정 (Externalized Configuration)
`application.yml` / 환경변수 / 커맨드라인이 우선순위대로 병합. 프로파일(`application-prod.yml`)로 환경 분리.

### 5. Actuator
`/actuator/health`, `/metrics`, `/prometheus`, `/info` 운영 엔드포인트 자동 노출.

## Spring Boot 4 변화점

| 항목 | 변화 |
|------|------|
| Java 베이스라인 | Java 17+ → **Java 21+** |
| Jakarta EE | EE 11 |
| 모듈 분리 | `spring-boot-actuate.health.*` → `spring-boot-health` (4.A 참고) |
| MVC 테스트 | `spring-boot-webmvc-test` 별도 모듈 |
| Observability | `spring-boot-starter-opentelemetry` 단일 의존성 |
| Servlet | `org.springframework.boot.servlet.*` 정리 |

## 이 프로젝트에서

- `ShopTrackerApplication.java`의 `@SpringBootApplication` = `@Configuration + @EnableAutoConfiguration + @ComponentScan` 합성.
- 모든 모듈의 `@Service`, `@Repository`, `@RestController`가 컴포넌트 스캔으로 자동 발견.
- `application.yml` + `application-{dev,prod}.yml` 프로파일 조합.

## FastAPI 대응

| FastAPI | Spring Boot |
|---------|-------------|
| `FastAPI()` 인스턴스 | `@SpringBootApplication` |
| Uvicorn(별도 실행) | 내장 Tomcat |
| `pyproject.toml` 직접 의존성 | starter로 묶음 |
| 자동 설정 없음 (직접 wiring) | `@EnableAutoConfiguration` |
| `python -m main` | `java -jar app.jar` |
