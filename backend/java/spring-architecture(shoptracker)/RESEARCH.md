# Spring 아키텍처 설계 철학 (2026년 2월 개정판)

## 목차

1. [개요](#개요)
2. [창시자의 설계 철학](#창시자의-설계-철학)
3. [핵심 설계 원칙](#핵심-설계-원칙)
4. [권장 아키텍처 패턴](#권장-아키텍처-패턴)
5. [의존성 주입 (Dependency Injection)](#의존성-주입-dependency-injection)
6. [Clean Architecture & Hexagonal Architecture 적용](#clean-architecture--hexagonal-architecture-적용)
7. [CQRS & Event-Driven Architecture](#cqrs--event-driven-architecture)
8. [Spring Modulith — 모듈러 모놀리스](#spring-modulith--모듈러-모놀리스)
9. [프로젝트 구조](#프로젝트-구조)
10. [마이크로서비스 아키텍처](#마이크로서비스-아키텍처)
11. [프로덕션 배포 & Observability](#프로덕션-배포--observability)
12. [Spring AI — AI 통합](#spring-ai--ai-통합)
13. [2026년 생태계 변화 요약](#2026년-생태계-변화-요약)
14. [참고 자료](#참고-자료)

---

## 개요

Spring Framework는 2002년 Rod Johnson이 "Expert One-on-One J2EE Design and Development"에서 발표한 코드를 기반으로 탄생한 Java/JVM 생태계의 대표 프레임워크입니다. Spring Boot는 Spring 위에 "opinionated" 설정을 얹어 생산성을 극대화한 프로젝트로, 2014년 첫 릴리스 이후 Java 엔터프라이즈 개발의 사실상 표준이 되었습니다.

Spring의 핵심 철학은 **복잡한 엔터프라이즈 개발을 단순하게 만드는 것**입니다. EJB의 무거움에 대한 대안으로 시작하여, IoC(Inversion of Control)와 DI(Dependency Injection)를 통해 POJO 기반 개발을 실현했습니다.

> "I wrote a framework that introduced the core ideas of Spring: dependency injection, simplified JDBC, unchecked exceptions were in the first framework."
>
> — Rod Johnson, InfoQ Podcast

**출처**: [InfoQ - Rod Johnson Chats about the Spring Framework Early Days](https://www.infoq.com/podcasts/johnson-spring-framework/)

### 2026년 2월 기준 현황

| 항목 | 상태 |
|------|------|
| **Spring Boot** | 4.0.2 (2026-01-23) |
| **Spring Framework** | 7.0.x |
| **Java 지원** | JDK 17+ (권장 JDK 21/25) |
| **Jakarta EE** | 11 (javax.* → jakarta.* 완전 전환) |
| **Hibernate** | 7.x (JPA 3.2) |
| **Spring AI** | 2.0.0 (M2 진행 중) |
| **Spring Modulith** | 2.0.x |
| **포지션** | Java 웹 프레임워크 1위, 엔터프라이즈 사실상 표준 |

> ⚠️ **Spring Boot 3.5.x는 2026년 6월까지 오픈소스 지원**. 새 프로젝트는 **Spring Boot 4.0 + JDK 21**로 시작할 것을 권장합니다.

**출처**: [Spring Boot 4.0 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes), [InfoQ - Spring Framework 7 and Spring Boot 4](https://www.infoq.com/news/2025/11/spring-7-spring-boot-4/)

---

## 창시자의 설계 철학

### Rod Johnson — "EJB 없는 J2EE"

Rod Johnson은 2002년 저서에서 J2EE(현 Jakarta EE)의 과도한 복잡성을 비판하고, POJO 기반의 경량 프레임워크를 제안했습니다. 이것이 Spring Framework의 시작입니다.

> "Spring is designed so that applications built with it depend on as few of its APIs as possible. Most business objects in Spring applications have no dependency on Spring."

**출처**: [TheServerSide - Introduction to the Spring Framework](https://www.theserverside.com/news/1364527/Introduction-to-the-Spring-Framework)

Rod Johnson의 비전은 세 가지로 요약됩니다:

1. **모듈화(Modular)**: 프레임워크의 필요한 부분만 사용
2. **애플리케이션 서버 독립적**: WebLogic, JBoss 같은 무거운 서버 불필요
3. **단순함(Simple)**: 트랜잭션 설정, 의존성 주입을 직관적으로

> "Rod Johnson's vision was not just to reduce the complexity of J2EE but to create a tool that was: Modular, Application Server Independent, Simple."

**출처**: [Medium - Inside the Spring Framework](https://medium.com/@well-araujo/inside-the-spring-framework-the-foundation-of-modern-java-development-85e190b3ec0d)

### Jürgen Höller — Spring의 지속적 진화

Rod Johnson이 Spring의 창시자라면, Jürgen Höller는 Spring의 실질적 기술 리더로서 20년 이상 프레임워크를 이끌어 왔습니다.

> "Juergen stepped up and is an amazing engineer — he continues to lead Spring to this day and his contribution was incredible."
>
> — Rod Johnson, InfoQ Podcast

**출처**: [InfoQ - Rod Johnson Chats about the Spring Framework Early Days](https://www.infoq.com/podcasts/johnson-spring-framework/)

### Spring의 핵심 삼각형 (The Spring Triangle)

Rod Johnson이 2003년 말에 확립한 Spring의 세 가지 핵심 기둥:

1. **Dependency Injection**: 객체 생성과 의존성 관리를 프레임워크에 위임
2. **AOP (Aspect-Oriented Programming)**: 횡단 관심사(트랜잭션, 보안 등) 분리
3. **Portable Service Abstractions**: 벤더 독립적인 서비스 추상화

> "We had already established the Spring triangle of dependency injection, AOP and portable service abstractions — that was well established by the end of 2003."
>
> — Rod Johnson

**출처**: [InfoQ - Rod Johnson Chats about the Spring Framework Early Days](https://www.infoq.com/podcasts/johnson-spring-framework/)

---

## 핵심 설계 원칙

### 1. IoC (Inversion of Control) & Dependency Injection

Spring의 가장 근본적인 원칙입니다. "Don't call me, I'll call you" — 할리우드 원칙으로도 알려져 있습니다.

> "Spring is most closely identified with a flavor of Inversion of Control known as Dependency Injection — a name coined by Martin Fowler, Rod Johnson and the PicoContainer team in late 2003."

**출처**: [TheServerSide - Introduction to the Spring Framework](https://www.theserverside.com/news/1364527/Introduction-to-the-Spring-Framework)

```java
// ✅ 생성자 주입 (권장)
@Service
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    // Spring Boot 4에서는 단일 생성자 시 @Autowired 불필요
    public UserService(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }

    public User createUser(CreateUserCommand command) {
        // 비즈니스 로직
        String encoded = passwordEncoder.encode(command.password());
        return userRepository.save(new User(command.email(), encoded));
    }
}

// ⚠️ 필드 주입은 권장하지 않음
@Service
public class BadService {
    @Autowired  // ❌ 테스트가 어려워지고, 불변성을 보장할 수 없음
    private UserRepository userRepository;
}
```

> ⚠️ **2026년 권장**: 생성자 주입(Constructor Injection)을 사용하세요. 필드 주입(@Autowired)보다 테스트하기 쉽고, 의존성이 명시적이며, 불변성을 보장합니다.

### 2. Convention over Configuration (관례 우선 설정)

Spring Boot의 핵심 철학입니다. 합리적인 기본값을 제공하되, 필요할 때 커스터마이징할 수 있습니다.

```java
// Spring Boot 4 — 최소 설정으로 웹 애플리케이션 시작
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}

// application.yml — 필요한 것만 오버라이드
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
  jpa:
    hibernate:
      ddl-auto: validate
```

### 3. 모듈화 & 자동 설정 (Spring Boot 4의 모듈화 혁신)

Spring Boot 4.0의 가장 큰 변화 중 하나는 `spring-boot-autoconfigure` JAR의 모듈화입니다.

> "The entire Spring Boot codebase has been completely modularized, providing smaller and more focused JARs. Each technology gets its own module and starter."

**출처**: [Medium - Spring Boot 4.0: The Future of Java Development](https://medium.com/@vinodbokare0588/spring-boot-4-0-the-future-of-java-development-is-here-9071fdc66e36)

모듈화의 이점:
- 빌드 속도 향상
- 애플리케이션 시작 시간 단축
- 필요한 것만 포함 (불필요한 자동 설정 제거)
- GraalVM 네이티브 이미지 최적화 개선

### 4. JSpecify를 통한 Null Safety 표준화

Spring Framework 7에서 가장 주목할 변화 중 하나입니다.

> "Spring Framework 7 completes the migration to standardized JSpecify annotations for null safety across the Spring portfolio."

**출처**: [InfoQ - Spring Framework 7 and Spring Boot 4](https://www.infoq.com/news/2025/11/spring-7-spring-boot-4/)

```java
import org.jspecify.annotations.NonNull;
import org.jspecify.annotations.Nullable;

@Service
public class UserService {
    public @NonNull User findById(long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }

    public @Nullable User findByEmail(@NonNull String email) {
        return userRepository.findByEmail(email).orElse(null);
    }
}
```

JSpecify의 이점:
- IDE(IntelliJ 2025.3+)에서 컴파일 타임에 null 오류 감지
- Kotlin 2와의 자동 상호운용 (JSpecify → Kotlin nullability)
- 통일된 표준 (기존 @Nullable, @NonNull, @NotNull 등의 혼란 해소)

### 5. API 버전 관리 (Spring Framework 7 신기능)

Spring Framework 7.0에서 REST API 버전 관리가 프레임워크에 내장되었습니다.

> "The new API versioning feature is now available in Spring MVC and Spring WebFlux. The framework supports path, header, query parameter, and media type versioning strategies."

**출처**: [InfoQ - Spring Framework 7 and Spring Boot 4](https://www.infoq.com/news/2025/11/spring-7-spring-boot-4/)

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping(value = "/{id}", version = "1.0")
    public UserV1Response getUserV1(@PathVariable Long id) {
        return new UserV1Response(id, "John");
    }

    @GetMapping(value = "/{id}", version = "2.0")
    public UserV2Response getUserV2(@PathVariable Long id) {
        return new UserV2Response(id, "John", "john@example.com", List.of("ADMIN"));
    }
}

// 설정: path 기반 버전 관리
@Configuration
public class ApiConfig implements WebMvcConfigurer {
    @Override
    public void configureApiVersioning(ApiVersionConfigurer configurer) {
        configurer.usePathSegment(1); // /v1/api/users, /v2/api/users
    }
}
```

### 6. 내장 Resilience (복원력)

Spring Boot 4는 retry와 동시성 제한을 프레임워크 수준에서 지원합니다.

```java
@Service
public class ExternalApiService {
    // Spring Framework 7의 내장 resilience
    @Retryable(maxAttempts = 3, backoff = @Backoff(delay = 1000))
    public ExternalData fetchData(String id) {
        return restClient.get()
            .uri("/external/{id}", id)
            .retrieve()
            .body(ExternalData.class);
    }
}
```

---

## 권장 아키텍처 패턴

### 레이어드 아키텍처 (Layered Architecture)

Spring 커뮤니티에서 가장 전통적이고 널리 사용되는 구조입니다.

```
┌─────────────────────────────────────┐
│        Presentation Layer           │  ← @RestController, DTO
│        (Controllers)                │
├─────────────────────────────────────┤
│        Application Layer            │  ← @Service, Use Cases
│        (Services)                   │
├─────────────────────────────────────┤
│         Domain Layer                │  ← Entity, Value Objects
│        (Domain Model)               │
├─────────────────────────────────────┤
│       Infrastructure Layer          │  ← @Repository, Config
│    (Repositories / Adapters)        │
└─────────────────────────────────────┘
```

### 각 레이어의 책임과 Spring 어노테이션

| 레이어 | Spring 어노테이션 | 책임 | 예시 |
|--------|-------------------|------|------|
| **Presentation** | `@RestController` | HTTP 처리, 요청 검증, 응답 형성 | `UserController` |
| **Application** | `@Service` | 비즈니스 로직 조율, 트랜잭션 관리 | `UserService` |
| **Domain** | (POJO) | 도메인 엔티티, 비즈니스 규칙 | `User`, `Order` |
| **Infrastructure** | `@Repository`, `@Configuration` | DB 접근, 외부 서비스 | `JpaUserRepository` |

### 단일 책임 원칙 (SRP)

```java
// ❌ 나쁜 예: 컨트롤러에 모든 로직
@RestController
@RequestMapping("/api/users")
public class UserController {
    @Autowired private UserRepository repo;
    @Autowired private PasswordEncoder encoder;

    @PostMapping
    public User createUser(@RequestBody CreateUserRequest request) {
        // 검증, 해싱, 저장, 이메일 발송 모두 여기에...
        if (repo.existsByEmail(request.email())) {
            throw new ResponseStatusException(HttpStatus.CONFLICT);
        }
        String hashed = encoder.encode(request.password());
        User user = new User(request.email(), hashed);
        User saved = repo.save(user);
        emailService.sendWelcome(saved.getEmail());
        return saved;
    }
}

// ✅ 좋은 예: 레이어 분리
@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        UserResponse response = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}
```

---

## 의존성 주입 (Dependency Injection)

### Spring IoC 컨테이너

Spring의 DI는 프레임워크의 핵심이자 가장 강력한 기능입니다.

> "Dependency Injection is a form of push configuration; the container 'pushes' dependencies into application objects at runtime. This is the opposite of traditional pull configuration."

**출처**: [Professional Java Development with the Spring Framework - Rod Johnson](https://iamgodsom.wordpress.com/wp-content/uploads/2014/08/wrox-professional-java-development-with-the-spring-framework.pdf)

#### 주입 방식 비교 (2026년 권장)

```java
// ✅ 1. 생성자 주입 (강력히 권장)
// - 불변 보장 (final 필드)
// - 테스트 용이 (new로 직접 생성 가능)
// - 순환 참조 방지 (컴파일 타임에 감지)
@Service
public class OrderService {
    private final OrderRepository orderRepo;
    private final PaymentService paymentService;
    private final EventPublisher eventPublisher;

    // Spring Boot 4: 단일 생성자면 @Autowired 불필요
    public OrderService(OrderRepository orderRepo,
                        PaymentService paymentService,
                        EventPublisher eventPublisher) {
        this.orderRepo = orderRepo;
        this.paymentService = paymentService;
        this.eventPublisher = eventPublisher;
    }
}

// ✅ 2. Lombok + 생성자 주입 (보일러플레이트 제거)
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepo;
    private final PaymentService paymentService;
    private final EventPublisher eventPublisher;
}

// ❌ 3. 필드 주입 (비추천)
@Service
public class OrderService {
    @Autowired private OrderRepository orderRepo;  // 테스트 어려움
}
```

#### Bean 스코프

| 스코프 | 설명 | 사용 시점 |
|--------|------|-----------|
| `singleton` (기본값) | 컨테이너당 하나의 인스턴스 | 대부분의 서비스 |
| `prototype` | 요청 시마다 새 인스턴스 | 상태를 가진 빈 |
| `request` | HTTP 요청당 하나 | 요청별 컨텍스트 |
| `session` | HTTP 세션당 하나 | 세션별 상태 |

#### Java Config 기반 설정 (Spring Boot 4 스타일)

```java
@Configuration
public class AppConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    @Profile("production")
    public DataSource productionDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://prod-db:5432/myapp");
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        return new HikariDataSource(config);
    }
}
```

#### 조건부 Bean 등록

```java
@Configuration
public class ConditionalConfig {

    @Bean
    @ConditionalOnProperty(name = "cache.type", havingValue = "redis")
    public CacheManager redisCacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory).build();
    }

    @Bean
    @ConditionalOnMissingBean(CacheManager.class)
    public CacheManager defaultCacheManager() {
        return new ConcurrentMapCacheManager();
    }
}
```

---

## Clean Architecture & Hexagonal Architecture 적용

### Spring에서의 Hexagonal Architecture

2025~2026년 Spring 커뮤니티에서 Hexagonal Architecture(Ports & Adapters)가 매우 활발하게 논의되고 있습니다.

> "Hexagonal Architecture, also known as Ports and Adapters, isolates business logic from external dependencies like databases or UI, promoting testability and flexibility."

**출처**: [Medium - Hexagonal Architecture in Spring Boot: Clean Code vs. Spring's Opinion](https://medium.com/@ntiinsd/hexagonal-architecture-in-spring-boot-clean-code-vs-springs-opinion-the-ultimate-showdown-cc00883c0863)

```
                    ┌─────────────────────────────┐
                    │      Driving Adapters        │
                    │  (REST Controller, CLI, Test) │
                    └──────────┬──────────────────┘
                               │
                    ┌──────────▼──────────────────┐
                    │      Input Ports             │
                    │  (Use Case Interfaces)       │
                    ├─────────────────────────────┤
                    │      Application Core        │
                    │  ┌───────────────────────┐  │
                    │  │    Domain Entities     │  │
                    │  │    Business Rules      │  │
                    │  │    Value Objects        │  │
                    │  └───────────────────────┘  │
                    ├─────────────────────────────┤
                    │      Output Ports            │
                    │  (Repository Interfaces)     │
                    └──────────┬──────────────────┘
                               │
                    ┌──────────▼──────────────────┐
                    │      Driven Adapters         │
                    │  (JPA, Redis, Kafka, HTTP)    │
                    └─────────────────────────────┘
```

### 핵심 원칙: 의존성은 항상 안쪽을 향한다

> "Your domain and application code should know nothing about Spring, HTTP, or JPA. They're just Java."

**출처**: [Package by Layer vs Package by Feature](https://jshingler.github.io/blog/2025/10/25/package-by-feature-vs-clean-architecture/)

### 구현 예시

```java
// ===== DOMAIN LAYER (순수 Java, Spring 의존성 없음) =====

// domain/model/Order.java
public class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private OrderStatus status;
    private final List<OrderItem> items;
    private Money totalAmount;

    public void place() {
        if (items.isEmpty()) {
            throw new OrderHasNoItemsException();
        }
        this.status = OrderStatus.PLACED;
        this.totalAmount = calculateTotal();
    }

    private Money calculateTotal() {
        return items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// domain/port/out/OrderRepository.java (Output Port)
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(OrderId id);
}

// domain/port/in/PlaceOrderUseCase.java (Input Port)
public interface PlaceOrderUseCase {
    OrderId placeOrder(PlaceOrderCommand command);
}

// ===== APPLICATION LAYER (Use Case 구현) =====

// application/service/PlaceOrderService.java
// 주의: @Service는 인프라 레이어에서 등록해도 됨
public class PlaceOrderService implements PlaceOrderUseCase {
    private final OrderRepository orderRepository;
    private final EventPublisher eventPublisher;

    public PlaceOrderService(OrderRepository orderRepository,
                              EventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(command.customerId(), command.items());
        order.place();
        Order saved = orderRepository.save(order);
        eventPublisher.publish(new OrderPlacedEvent(saved.getId()));
        return saved.getId();
    }
}

// ===== INFRASTRUCTURE LAYER (Driven Adapter) =====

// infrastructure/persistence/JpaOrderRepository.java
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final SpringDataOrderRepository springDataRepo;
    private final OrderMapper mapper;

    public JpaOrderRepository(SpringDataOrderRepository springDataRepo,
                               OrderMapper mapper) {
        this.springDataRepo = springDataRepo;
        this.mapper = mapper;
    }

    @Override
    public Order save(Order order) {
        OrderEntity entity = mapper.toEntity(order);
        OrderEntity saved = springDataRepo.save(entity);
        return mapper.toDomain(saved);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return springDataRepo.findById(id.value())
            .map(mapper::toDomain);
    }
}

// ===== PRESENTATION LAYER (Driving Adapter) =====

// presentation/api/OrderController.java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final PlaceOrderUseCase placeOrderUseCase;

    public OrderController(PlaceOrderUseCase placeOrderUseCase) {
        this.placeOrderUseCase = placeOrderUseCase;
    }

    @PostMapping
    public ResponseEntity<OrderIdResponse> placeOrder(
            @Valid @RequestBody PlaceOrderRequest request) {
        PlaceOrderCommand command = request.toCommand();
        OrderId orderId = placeOrderUseCase.placeOrder(command);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(new OrderIdResponse(orderId.value()));
    }
}
```

### 실용적 조언: 모든 곳에 적용하지 말 것

> "You don't earn the right to a Clean Architecture until you have complexity to clean. Begin with package-by-feature. Refactor toward layers as needed."

**출처**: [Package by Layer vs Package by Feature](https://jshingler.github.io/blog/2025/10/25/package-by-feature-vs-clean-architecture/)

복잡한 비즈니스 로직이 있는 핵심 도메인에는 Hexagonal Architecture를, 단순 CRUD 모듈에는 간단한 레이어드 구조를 적용하는 것이 실용적입니다.

---

## CQRS & Event-Driven Architecture

### CQRS (Command Query Responsibility Segregation)

Spring 생태계에서는 Axon Framework와 Spring Modulith를 활용한 CQRS 구현이 주류입니다.

> "CQRS separates your application's command operations (data modifications) from query operations (data retrieval)."

**출처**: [SpringFuse - Implementing CQRS in Spring Boot Applications](https://www.springfuse.com/implementing-cqrs-spring-boot-applications/)

#### Axon Framework 기반 CQRS

```java
// Command (쓰기 측)
@Aggregate
public class OrderAggregate {
    @AggregateIdentifier
    private String orderId;
    private OrderStatus status;

    @CommandHandler
    public OrderAggregate(CreateOrderCommand command) {
        AggregateLifecycle.apply(new OrderCreatedEvent(
            command.getOrderId(),
            command.getCustomerId(),
            command.getItems()
        ));
    }

    @EventSourcingHandler
    public void on(OrderCreatedEvent event) {
        this.orderId = event.getOrderId();
        this.status = OrderStatus.CREATED;
    }
}

// Query (읽기 측)
@Component
public class OrderProjection {
    private final OrderReadRepository readRepository;

    @EventHandler
    public void on(OrderCreatedEvent event) {
        OrderReadModel model = new OrderReadModel(
            event.getOrderId(),
            event.getCustomerId(),
            "CREATED"
        );
        readRepository.save(model);
    }

    @QueryHandler
    public OrderReadModel handle(FindOrderQuery query) {
        return readRepository.findById(query.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(query.getOrderId()));
    }
}
```

#### Spring Modulith 기반 CQRS (2025~2026 트렌드)

Spring Modulith는 모듈 간 이벤트 기반 통신을 통해 CQRS를 깔끔하게 구현할 수 있습니다.

> "Spring Modulith helps us structure Spring Boot applications into clear and loosely connected modules. It encourages modeling each module around a specific business area."

**출처**: [Baeldung - Implementing CQRS with Spring Modulith](https://www.baeldung.com/spring-modulith-cqrs)

```java
// Command 측 — ticket 모듈
@Service
@Transactional
public class TicketBookingService {
    private final TicketRepository ticketRepo;
    private final ApplicationEventPublisher eventPublisher;

    public TicketBookingService(TicketRepository ticketRepo,
                                 ApplicationEventPublisher eventPublisher) {
        this.ticketRepo = ticketRepo;
        this.eventPublisher = eventPublisher;
    }

    public void bookTicket(BookTicketCommand command) {
        Ticket ticket = Ticket.book(command.movieId(), command.seatNumber());
        ticketRepo.save(ticket);
        eventPublisher.publishEvent(
            new TicketBookedEvent(ticket.getId(), command.movieId(), command.seatNumber())
        );
    }
}

// Query 측 — movie 모듈 (비동기 이벤트 수신)
@Service
public class MovieQueryService {
    private final MovieReadRepository readRepo;

    @ApplicationModuleListener  // Spring Modulith의 비동기 이벤트 리스너
    public void on(TicketBookedEvent event) {
        readRepo.markSeatAsBooked(event.movieId(), event.seatNumber());
    }

    public MovieSeatAvailability getSeatAvailability(Long movieId) {
        return readRepo.findSeatAvailability(movieId);
    }
}
```

### Event-Driven Architecture

Spring 생태계에서 이벤트 기반 아키텍처는 Kafka, RabbitMQ 등과의 통합으로 구현됩니다.

```java
// Spring ApplicationEvent (도메인 이벤트)
public record OrderPlacedEvent(
    String orderId,
    String customerId,
    Instant occurredAt
) {}

// 이벤트 발행
@Service
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order placeOrder(PlaceOrderCommand command) {
        Order order = orderRepository.save(createOrder(command));
        eventPublisher.publishEvent(
            new OrderPlacedEvent(order.getId(), command.customerId(), Instant.now())
        );
        return order;
    }
}

// 이벤트 핸들러 (비동기)
@Component
public class OrderEventHandler {

    @Async
    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // 이메일 발송, 재고 업데이트 등
        notificationService.sendOrderConfirmation(event.orderId());
    }
}
```

### CQRS 도입 시점 판단

| 상황 | CQRS 필요? |
|------|-----------|
| 단순 CRUD, 읽기/쓰기 비율 비슷 | ❌ 불필요 |
| 읽기 >> 쓰기, 별도 스케일링 필요 | ✅ 고려 |
| 감사 로그, 이벤트 이력 필수 | ✅ 고려 (Event Sourcing 함께) |
| 복잡한 비즈니스 워크플로우 | ✅ 고려 (Saga 패턴 함께) |
| 소규모 팀, MVP 단계 | ❌ 오버엔지니어링 주의 |

---

## Spring Modulith — 모듈러 모놀리스

### 2025~2026년 가장 주목받는 아키텍처 패턴

Spring Modulith는 마이크로서비스의 복잡성 없이 모듈화의 이점을 얻는 "모듈러 모놀리스" 아키텍처를 지원합니다. Spring Boot 4와 함께 Spring Modulith 2.0이 릴리스되었습니다.

> "At the start of any project, teams are still learning the domain and the broader ecosystem. For this reason, a project should not begin with microservices. Most teams are better off starting with a monolithic application, specifically a modular monolith."

**출처**: [Medium - Modular Monolith with Spring Boot — Spring Modulith](https://senoritadeveloper.medium.com/modular-monolith-with-spring-boot-spring-modulith-6687c234daab)

### 핵심 개념

```
com.example.myapp/
├── MyApplication.java          # @SpringBootApplication (루트)
├── order/                       # order 모듈 (API 패키지)
│   ├── OrderService.java        # 외부에 노출되는 API
│   └── internal/                # 내부 구현 (다른 모듈에서 접근 불가)
│       ├── OrderRepository.java
│       └── OrderEntity.java
├── inventory/                   # inventory 모듈
│   ├── InventoryService.java
│   └── internal/
│       └── ...
└── payment/                     # payment 모듈
    ├── PaymentService.java
    └── internal/
        └── ...
```

### 모듈 경계 검증

```java
// Spring Modulith의 핵심: 아키텍처 검증 테스트
@Test
void verifyModuleStructure() {
    ApplicationModules modules = ApplicationModules.of(MyApplication.class);
    modules.verify();  // 모듈 경계 위반 시 테스트 실패!
}

// 모듈 문서 자동 생성
@Test
void createModuleDocumentation() {
    ApplicationModules modules = ApplicationModules.of(MyApplication.class);
    new Documenter(modules).writeDocumentation();
}
```

> ⚠️ Spring Modulith 2.0부터 `spring.modulith.runtime.verification-enabled=true`로 런타임에서도 모듈 경계를 강제할 수 있습니다.

### 이벤트 기반 모듈 간 통신

```java
// order 모듈에서 이벤트 발행
@Service
@Transactional
public class OrderService {
    private final ApplicationEventPublisher events;

    public void completeOrder(OrderId id) {
        // ... 주문 처리 로직
        events.publishEvent(new OrderCompleted(id));
    }
}

// payment 모듈에서 이벤트 수신 (비동기, 트랜잭션 분리)
@Service
public class PaymentService {

    @ApplicationModuleListener  // 자동 재시도 + 이벤트 지속성
    void on(OrderCompleted event) {
        processPayment(event.orderId());
    }
}
```

Spring Modulith의 이벤트 기능:
- **이벤트 지속성**: JPA/JDBC 기반 이벤트 저장 (실패 시 자동 재시도)
- **이벤트 외부화**: Kafka, RabbitMQ, AMQP 등으로 자동 전달
- **모듈 통합 테스트**: `@ApplicationModuleTest`로 모듈 단위 테스트

**출처**: [Baeldung - Introduction to Spring Modulith](https://www.baeldung.com/spring-modulith), [JetBrains - Building Modular Monoliths with Kotlin and Spring](https://blog.jetbrains.com/kotlin/2026/02/building-modular-monoliths-with-kotlin-and-spring/)

---

## 프로젝트 구조

### Package by Layer (소규모 프로젝트)

```
com.example.myapp/
├── MyApplication.java
├── config/
│   ├── SecurityConfig.java
│   └── WebConfig.java
├── controller/
│   ├── UserController.java
│   └── OrderController.java
├── service/
│   ├── UserService.java
│   └── OrderService.java
├── repository/
│   ├── UserRepository.java
│   └── OrderRepository.java
├── model/
│   ├── User.java
│   └── Order.java
├── dto/
│   ├── UserRequest.java
│   └── UserResponse.java
└── exception/
    ├── GlobalExceptionHandler.java
    └── ResourceNotFoundException.java
```

> "For smaller apps, Layer-based; for larger or team projects, Feature-based."

**출처**: [Medium - Spring Boot Project Structure Best Practices](https://rifaiio.medium.com/spring-boot-project-structure-best-practices-layer-based-vs-feature-based-explained-simply-4a9002f3cff0)

### Package by Feature (중규모 프로젝트)

```
com.example.myapp/
├── MyApplication.java
├── shared/
│   ├── config/
│   ├── exception/
│   └── security/
├── user/
│   ├── UserController.java
│   ├── UserService.java
│   ├── UserRepository.java
│   ├── User.java
│   └── dto/
│       ├── CreateUserRequest.java
│       └── UserResponse.java
├── order/
│   ├── OrderController.java
│   ├── OrderService.java
│   ├── OrderRepository.java
│   ├── Order.java
│   └── dto/
└── payment/
    └── (동일 구조)
```

### Hexagonal + Feature 하이브리드 (대규모/엔터프라이즈)

2025~2026년에 가장 활발하게 논의되는 구조입니다.

> "This hybrid structure is exactly what many experienced Spring Boot engineers (Dan Vega included) use for production systems."

**출처**: [Package by Layer vs Package by Feature](https://jshingler.github.io/blog/2025/10/25/package-by-feature-vs-clean-architecture/)

```
com.example.myapp/
├── MyApplication.java
├── shared/
│   ├── config/
│   │   ├── SecurityConfig.java
│   │   └── OpenTelemetryConfig.java
│   └── common/
│       └── Money.java
├── order/
│   ├── domain/                    # 🎯 순수 비즈니스 로직 (Spring 의존성 없음)
│   │   ├── Order.java             # Aggregate Root
│   │   ├── OrderItem.java         # Entity
│   │   ├── OrderStatus.java       # Value Object
│   │   └── OrderRepository.java   # Output Port (Interface)
│   ├── application/               # 🔄 Use Cases
│   │   ├── port/
│   │   │   └── in/
│   │   │       └── PlaceOrderUseCase.java  # Input Port
│   │   └── service/
│   │       └── PlaceOrderService.java
│   ├── adapter/                   # 🔧 외부 연결
│   │   ├── in/
│   │   │   └── web/
│   │   │       ├── OrderController.java
│   │   │       └── OrderRequest.java
│   │   └── out/
│   │       └── persistence/
│   │           ├── JpaOrderRepository.java
│   │           ├── OrderEntity.java
│   │           └── OrderMapper.java
│   └── infrastructure/
│       └── config/
│           └── OrderModuleConfig.java
├── customer/
│   ├── (동일 구조)
└── payment/
    ├── (동일 구조)
```

### Spring Modulith 기반 구조

```
com.example.myapp/
├── MyApplication.java
├── order/                          # order 모듈 (API 노출 패키지)
│   ├── OrderService.java           # 공개 API
│   ├── OrderCompleted.java         # 도메인 이벤트 (공개)
│   └── internal/                   # 내부 구현 (모듈 외부 접근 불가)
│       ├── OrderController.java
│       ├── OrderRepository.java
│       ├── OrderEntity.java
│       └── OrderConfig.java
├── inventory/
│   ├── InventoryService.java
│   └── internal/
│       └── ...
└── notification/
    ├── NotificationService.java
    └── internal/
        └── ...
```

### 환경 설정 관리

```yaml
# application.yml — Spring Boot 4 스타일
spring:
  application:
    name: myapp
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
  jpa:
    open-in-view: false  # ⚠️ 프로덕션에서는 반드시 false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        default_batch_fetch_size: 100

# Observability (Spring Boot 4)
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  tracing:
    sampling:
      probability: 1.0  # 개발: 1.0, 프로덕션: 0.1

---
# 프로덕션 프로파일
spring:
  config:
    activate:
      on-profile: production
  datasource:
    url: ${DATABASE_URL}
  jpa:
    hibernate:
      ddl-auto: none
```

---

## 마이크로서비스 아키텍처

### Spring Cloud 생태계

Spring Boot + Spring Cloud는 Java 마이크로서비스의 사실상 표준입니다.

| 구성요소 | Spring Cloud 프로젝트 | 2026년 상태 |
|----------|----------------------|------------|
| **서비스 디스커버리** | Spring Cloud Netflix Eureka / Consul | Kubernetes 네이티브 디스커버리 선호 추세 |
| **API 게이트웨이** | Spring Cloud Gateway | Reactive 기반, 주요 선택지 유지 |
| **설정 관리** | Spring Cloud Config | Kubernetes ConfigMap/Secret 대체 추세 |
| **서킷 브레이커** | Spring Cloud Circuit Breaker (Resilience4j) | 활발히 사용 |
| **메시징** | Spring Cloud Stream (Kafka/RabbitMQ) | 활발히 사용 |
| **분산 추적** | Micrometer Tracing + OpenTelemetry | Spring Boot 4 내장 |

### Spring Boot 4 + Kubernetes

```yaml
# Kubernetes 배포 예시
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: order-service
          image: myapp/order-service:latest
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "production,k8s"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
```

### GraalVM 네이티브 이미지 (Spring Boot 4 강화)

Spring Boot 4는 GraalVM 24와의 완전한 정렬을 제공합니다.

```bash
# Maven으로 네이티브 이미지 빌드
./mvnw -Pnative native:compile

# 결과: 네이티브 바이너리
# 시작 시간: ~50ms (JVM: ~2-3초)
# 메모리: ~100MB (JVM: ~300-500MB)
```

### 2025~2026 마이크로서비스 트렌드

| 영역 | 트렌드 |
|------|--------|
| **아키텍처** | Modular Monolith → 필요 시 Microservices 추출 |
| **메시징** | Spring Cloud Stream + Kafka/RabbitMQ |
| **컨테이너** | Kubernetes + GraalVM 네이티브 이미지 |
| **관찰성** | OpenTelemetry (Spring Boot 4 내장 지원) |
| **패턴** | CQRS, Saga, Event-Driven, Outbox |
| **AI 서빙** | Spring AI로 LLM 통합 |
| **서비스 메시** | Istio / Linkerd (Spring에서 직접 지원 약화 추세) |

---

## 프로덕션 배포 & Observability

### Virtual Threads (Java 21+)

Spring Boot 4에서 Virtual Threads를 쉽게 활성화할 수 있습니다.

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true  # 모든 요청 처리에 Virtual Thread 사용
```

```java
// Virtual Thread 활성화 시, 블로킹 I/O도 효율적으로 처리
@Service
public class ReportService {
    // 이 메서드가 DB를 호출해도 Virtual Thread에서 실행
    // 수천 개의 동시 요청도 적은 OS 스레드로 처리 가능
    public Report generateReport(Long userId) {
        User user = userRepo.findById(userId).orElseThrow();
        List<Order> orders = orderRepo.findByUserId(userId);
        return Report.from(user, orders);
    }
}
```

### Observability (Spring Boot 4 — OpenTelemetry 통합)

Spring Boot 4의 가장 중요한 프로덕션 기능 중 하나입니다.

> "The new spring-boot-starter-opentelemetry is the official Spring team solution. It leverages the modularization work in Spring Boot 4 to give you focused observability support without the overhead of the entire Actuator module."

**출처**: [Dan Vega - OpenTelemetry with Spring Boot 4](https://www.danvega.dev/blog/2025/12/23/opentelemetry-spring-boot)

```xml
<!-- Spring Boot 4: 단일 의존성으로 전체 관찰성 설정 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-opentelemetry</artifactId>
</dependency>
```

```yaml
# application.yml — OpenTelemetry 설정
spring:
  application:
    name: order-service

management:
  otlp:
    metrics:
      export:
        url: http://otel-collector:4318/v1/metrics
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
  tracing:
    sampling:
      probability: 0.1  # 프로덕션: 10% 샘플링
```

```java
// @Observed 어노테이션으로 자동 계측
@Service
public class PaymentService {

    @Observed(name = "payment.process",
              contextualName = "process-payment")
    public PaymentResult processPayment(String orderId) {
        // Spring이 자동으로 Timer + Counter + Trace span 생성
        return executePayment(orderId);
    }
}

// 수동 Observation API
@Service
public class CustomObservabilityService {
    private final ObservationRegistry observationRegistry;

    public void doWork() {
        Observation.createNotStarted("custom-operation", observationRegistry)
            .lowCardinalityKeyValue("type", "batch")
            .observe(() -> {
                // 관찰 대상 코드
            });
    }
}
```

### SSL 인증서 모니터링 (Spring Boot 4 신기능)

```yaml
# Actuator에서 SSL 인증서 만료 모니터링
management:
  endpoints:
    web:
      exposure:
        include: health,ssl
```

```json
{
  "ssl": {
    "status": "UP",
    "details": {
      "validSSL": [{
        "bundle": "my-server-cert",
        "expiresIn": "59 days",
        "expiryDate": "2026-04-18T12:00:00Z"
      }]
    }
  }
}
```

### 데이터베이스 연결 풀 설정 (HikariCP)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000        # 5분
      max-lifetime: 1800000       # 30분
      connection-timeout: 20000   # 20초
      leak-detection-threshold: 60000  # 1분 (개발용)
```

---

## Spring AI — AI 통합

### 2025~2026년 가장 빠르게 성장하는 Spring 프로젝트

Spring AI 1.0 GA가 2025년 5월에 릴리스되었으며, 2026년 초 현재 2.0 마일스톤이 진행 중입니다.

> "Spring AI addresses the fundamental challenge of AI integration: Connecting your enterprise Data and APIs with the AI Models."

**출처**: [GitHub - spring-projects/spring-ai](https://github.com/spring-projects/spring-ai)

### 핵심 기능

```java
// ChatClient — LLM 통합의 핵심
@Service
public class AiAssistantService {
    private final ChatClient chatClient;

    public AiAssistantService(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("You are a helpful assistant for our e-commerce platform.")
            .build();
    }

    public String askQuestion(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }

    // 구조화된 출력 (POJO 매핑)
    public ProductRecommendation getRecommendation(String userPreference) {
        return chatClient.prompt()
            .user("Recommend a product for: " + userPreference)
            .call()
            .entity(ProductRecommendation.class);
    }
}
```

### Spring AI 생태계 (2026년 2월)

| 기능 | 상태 |
|------|------|
| **모델 지원** | OpenAI, Anthropic, Google, Amazon Bedrock, Ollama 등 |
| **RAG** | Vector Store API (PGVector, Milvus, Pinecone 등) |
| **Function Calling / Tool Use** | GA |
| **MCP (Model Context Protocol)** | Spring AI 2.0에서 강화 |
| **Structured Output** | POJO 자동 매핑 |
| **Agentic Workflows** | Spring AI Agent Clients (2026) |
| **Memory / Chat History** | 압축, 보존 정책 지원 |

**출처**: [InfoQ - Spring AI 1.0 Released](https://www.infoq.com/news/2025/05/spring-ai-1-0-streamlines-apps/), [Spring I/O 2026 - The Spring AI Ecosystem](https://2026.springio.net/sessions/the-spring-ai-ecosystem-in-2026-from-foundations-to-agents/)

### Rod Johnson의 Embabel 프로젝트

Spring 창시자 Rod Johnson이 2025년 JVM 기반 AI 에이전트 플랫폼 Embabel을 발표했습니다.

> "Not since I founded the Spring Framework have I been so convinced that a new project is needed. Not since I pioneered Dependency Injection and the other core Spring concepts, have I been so convinced that a new programming model is needed."

**출처**: [Medium - Rod Johnson, Embabel: A New Agent Platform For the JVM](https://medium.com/@springrod/embabel-a-new-agent-platform-for-the-jvm-1c83402e0014)

---

## 2026년 생태계 변화 요약

### 주요 변경사항 타임라인

| 항목 | 이전 | 2026년 현재 |
|------|------|------------|
| **Spring Boot** | 3.x | **4.0.2** (모듈화된 autoconfigure) |
| **Spring Framework** | 6.x | **7.0.x** (API 버전 관리, JSpecify) |
| **Java 최소** | JDK 17 | **JDK 17** (권장 21/25) |
| **Jakarta EE** | 10 | **11** |
| **Hibernate** | 6.x | **7.x** (JPA 3.2) |
| **Jackson** | 2.x | **3.x** |
| **Null Safety** | @Nullable (혼용) | **JSpecify** (표준화) |
| **HTTP 클라이언트** | RestTemplate | **RestClient / HTTP Interface Client** |
| **관찰성** | Micrometer + 외부 OTel | **spring-boot-starter-opentelemetry** |
| **모듈 아키텍처** | 모놀리스 vs 마이크로서비스 | **Spring Modulith (모듈러 모놀리스)** |
| **AI 통합** | 없음 | **Spring AI 1.0+ GA** |
| **빌드 도구** | Gradle 8 / Maven | **Gradle 9** 지원 추가 |

### 새롭게 부상한 도구/패턴

| 도구/패턴 | 설명 |
|-----------|------|
| **Spring Modulith 2.0** | 모듈러 모놀리스, 이벤트 지속성, 관찰성 |
| **Spring AI** | LLM 통합, RAG, MCP, Agentic workflows |
| **spring-boot-starter-opentelemetry** | 단일 의존성으로 전체 관찰성 |
| **RestClient** | RestTemplate 대체, 동기 HTTP 클라이언트 |
| **HTTP Interface Client** | 인터페이스만으로 HTTP 클라이언트 생성 |
| **JSpecify** | 표준화된 null safety 어노테이션 |
| **API Versioning** | Spring MVC/WebFlux 내장 API 버전 관리 |
| **Virtual Threads** | JDK 21+ 가상 스레드, 설정 한 줄로 활성화 |
| **GraalVM 네이티브** | Spring Boot 4에서 더욱 강화 |
| **jMolecules** | 아키텍처 역할 어노테이션 (@Command, @DomainEvent 등) |
| **ArchUnit** | 아키텍처 규칙 테스트 |
| **Embabel** | Rod Johnson의 JVM AI 에이전트 플랫폼 |

### RestTemplate → RestClient 마이그레이션

> Spring Framework 7.1 (2026년 11월 예상)에서 RestTemplate이 deprecated 예정이며, Spring Framework 8에서 제거될 예정입니다.

```java
// ❌ RestTemplate (deprecated 예정)
RestTemplate restTemplate = new RestTemplate();
ResponseEntity<User> response = restTemplate.exchange(
    "/api/users/{id}", HttpMethod.GET, null,
    User.class, userId
);

// ✅ RestClient (Spring Framework 7 권장)
RestClient restClient = RestClient.create();
User user = restClient.get()
    .uri("/api/users/{id}", userId)
    .retrieve()
    .body(User.class);

// ✅ HTTP Interface Client (인터페이스만으로 클라이언트 생성)
public interface UserClient {
    @GetExchange("/api/users/{id}")
    User getUser(@PathVariable Long id);

    @PostExchange("/api/users")
    User createUser(@RequestBody CreateUserRequest request);
}
```

**출처**: [InfoQ - Spring Framework 7 and Spring Boot 4](https://www.infoq.com/news/2025/11/spring-7-spring-boot-4/)

---

## 참고 자료

### 공식 자료

| 자료 | URL |
|------|-----|
| Spring 공식 사이트 | https://spring.io |
| Spring Boot 공식 문서 | https://docs.spring.io/spring-boot/docs/current/reference/html/ |
| Spring Boot GitHub | https://github.com/spring-projects/spring-boot |
| Spring Framework GitHub | https://github.com/spring-projects/spring-framework |
| Spring Initializr | https://start.spring.io |
| Spring Boot 4.0 Release Notes | https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes |
| Spring Modulith 문서 | https://docs.spring.io/spring-modulith/reference/ |
| Spring AI GitHub | https://github.com/spring-projects/spring-ai |

### Spring Boot 4 & Spring Framework 7

| 자료 | URL |
|------|-----|
| InfoQ - Spring 7 & Boot 4 릴리스 | https://www.infoq.com/news/2025/11/spring-7-spring-boot-4/ |
| JetBrains - Spring Boot 4 | https://blog.jetbrains.com/idea/2025/11/spring-boot-4/ |
| Baeldung - Spring Boot 4 & Framework 7 | https://www.baeldung.com/spring-boot-4-spring-framework-7 |
| Coding Shuttle - Spring Boot 4 Complete Guide | https://www.codingshuttle.com/blogs/spring-boot-4-a-complete-guide-to-new-features-improvements-and-why-you-should-upgrade/ |
| Loiane - Spring Boot 4 Key Features | https://loiane.com/2025/08/spring-boot-4-spring-framework-7-key-features/ |

### 창시자 & 설계 철학

| 자료 | URL |
|------|-----|
| InfoQ - Rod Johnson Podcast | https://www.infoq.com/podcasts/johnson-spring-framework/ |
| Rod Johnson - Embabel (Medium) | https://medium.com/@springrod/embabel-a-new-agent-platform-for-the-jvm-1c83402e0014 |
| Spring Framework Wikipedia | https://en.wikipedia.org/wiki/Spring_Framework |
| TheServerSide - Intro to Spring | https://www.theserverside.com/news/1364527/Introduction-to-the-Spring-Framework |

### 아키텍처 가이드

| 자료 | URL |
|------|-----|
| Baeldung - Hexagonal Architecture, DDD, and Spring | https://www.baeldung.com/hexagonal-architecture-ddd-spring |
| Reflectoring - Hexagonal Architecture with Java and Spring | https://reflectoring.io/spring-hexagonal/ |
| HappyCoders - Hexagonal Architecture | https://www.happycoders.eu/software-craftsmanship/hexagonal-architecture/ |
| Package by Feature vs Clean Architecture | https://jshingler.github.io/blog/2025/10/25/package-by-feature-vs-clean-architecture/ |
| foojay - Clean and Modular Java: Hexagonal | https://foojay.io/today/clean-and-modular-java-a-hexagonal-architecture-approach/ |
| Spring Modulith with DDD (GitHub) | https://github.com/xsreality/spring-modulith-with-ddd |

### CQRS & Event-Driven

| 자료 | URL |
|------|-----|
| Baeldung - CQRS and Event Sourcing in Java | https://www.baeldung.com/cqrs-event-sourcing-java |
| Baeldung - CQRS with Spring Modulith | https://www.baeldung.com/spring-modulith-cqrs |
| SpringFuse - CQRS in Spring Boot | https://www.springfuse.com/implementing-cqrs-spring-boot-applications/ |
| CQRS with Spring Modulith (블로그) | https://gaetanopiazzolla.github.io/java/design-patterns/springboot/2025/03/17/cqrs.html |
| Java Code Geeks - Axon Framework CQRS | https://www.javacodegeeks.com/2025/06/implementing-cqrs-and-event-sourcing-with-axon-framework-in-spring.html |

### Spring Modulith

| 자료 | URL |
|------|-----|
| Baeldung - Spring Modulith | https://www.baeldung.com/spring-modulith |
| JetBrains IDE 지원 | https://www.jetbrains.com/help/idea/spring-modulith.html |
| JetBrains - Modular Monoliths with Kotlin | https://blog.jetbrains.com/kotlin/2026/02/building-modular-monoliths-with-kotlin-and-spring/ |
| Medium - Modular Monolith with Spring Boot 4 | https://senoritadeveloper.medium.com/modular-monolith-with-spring-boot-spring-modulith-6687c234daab |
| Bell-sw - Spring Modulith 튜토리얼 | https://bell-sw.com/blog/how-to-build-a-modular-application-with-spring-modulith/ |

### Observability & 프로덕션

| 자료 | URL |
|------|-----|
| Dan Vega - OpenTelemetry with Spring Boot 4 | https://www.danvega.dev/blog/2025/12/23/opentelemetry-spring-boot |
| foojay - Spring Boot 4 OpenTelemetry | https://foojay.io/today/spring-boot-4-opentelemetry-explained/ |
| Baeldung - Observability with Spring Boot | https://www.baeldung.com/spring-boot-3-observability |
| Uptrace - Spring Boot Monitoring | https://uptrace.dev/blog/spring-boot-microservices-monitoring |
| OpenTelemetry Spring Boot Starter | https://opentelemetry.io/docs/zero-code/java/spring-boot-starter/ |

### Spring AI

| 자료 | URL |
|------|-----|
| Spring AI GitHub | https://github.com/spring-projects/spring-ai |
| Awesome Spring AI | https://github.com/spring-ai-community/awesome-spring-ai |
| InfoQ - Spring AI 1.0 GA | https://www.infoq.com/news/2025/05/spring-ai-1-0-streamlines-apps/ |
| Baeldung - Spring AI | https://www.baeldung.com/spring-ai |
| InfoWorld - Spring AI Tutorial | https://www.infoworld.com/article/4091447/spring-ai-tutorial-get-started-with-spring-ai.html |
| Spring I/O 2026 - AI Ecosystem | https://2026.springio.net/sessions/the-spring-ai-ecosystem-in-2026-from-foundations-to-agents/ |

### 프로젝트 구조 & 베스트 프랙티스

| 자료 | URL |
|------|-----|
| Baeldung - Package Structure | https://www.baeldung.com/spring-boot-package-structure |
| GitHub - springboot-best-practices | https://github.com/arsy786/springboot-best-practices |
| CodeWalnut - Best Practices for Scalable Apps | https://www.codewalnut.com/insights/spring-boot-best-practices-for-scalable-applications |

---

## 요약

Spring의 아키텍처 철학은 다음과 같이 요약할 수 있습니다:

| 원칙 | 설명 |
|------|------|
| **IoC & DI 중심** | 프레임워크의 핵심. 생성자 주입 권장 |
| **Convention over Configuration** | 합리적 기본값 + 필요 시 커스터마이징 |
| **모듈화** | Spring Boot 4의 autoconfigure 모듈화 |
| **Null Safety** | JSpecify 표준 채택 (Spring Framework 7) |
| **레이어 분리** | Presentation / Application / Domain / Infrastructure |
| **프레임워크 독립적 도메인** | 도메인 레이어에 Spring 의존성 없음 (Hexagonal) |
| **모듈러 모놀리스** | Spring Modulith로 모듈 경계 강제 |
| **이벤트 기반 통신** | 모듈 간 느슨한 결합 (ApplicationEvent + @ApplicationModuleListener) |
| **CQRS & Event Sourcing** | 복잡한 도메인에서 Axon / Spring Modulith 활용 |
| **Observability** | OpenTelemetry 네이티브 통합 (Spring Boot 4) |
| **AI 네이티브** | Spring AI로 LLM 통합의 새로운 지평 |
| **Cloud Native** | Virtual Threads, GraalVM 네이티브, Kubernetes |

> "Good architecture is what lets you keep shipping features without hating yourself in six months."
>
> — Spring Boot 커뮤니티 격언

---

*이 문서는 2026년 2월 18일 기준으로 작성되었습니다. Spring Boot 4.0.2, Spring Framework 7.0.x, Spring AI 2.0 M2 등 최신 생태계 변화와 커뮤니티 트렌드를 반영하여 작성되었습니다.*