# Spring Data JPA + Hibernate

> 자바의 ORM 표준(JPA) + 가장 널리 쓰이는 구현체(Hibernate) + Spring Data의 Repository 추상화.

## 계층

```
┌────────────────────────────────────┐
│  Spring Data JPA                   │  Repository 인터페이스 자동 구현, 페이징, 쿼리 메서드 파생
├────────────────────────────────────┤
│  JPA (Jakarta Persistence)          │  표준 어노테이션/EntityManager API
├────────────────────────────────────┤
│  Hibernate                         │  실제 ORM 구현체, SQL 생성, 캐시, 더티체크
├────────────────────────────────────┤
│  JDBC                              │  DB 드라이버
└────────────────────────────────────┘
```

## 이 프로젝트의 패턴: 도메인-JPA 분리

순수 도메인 클래스는 JPA 어노테이션 없이 둔다. 별도 JpaEntity와 Mapper가 변환을 담당.

```
도메인 (orders/domain/model/Order.java)         ← Spring/JPA import 0건
   ↑↓ Mapper (orders/adapter/outbound/persistence/OrderMapper.java)
JPA Entity (OrderJpaEntity.java)               ← @Entity, @Column
   ↑↓
Spring Data Repository (SpringDataOrderRepository.java)
   ↑↓
PersistenceAdapter (OrderPersistenceAdapter.java) ← OutputPort 구현
```

이렇게 분리하는 이유:
- 도메인이 JPA의 lazy 로딩, 프록시, 부모 참조 같은 인프라에 오염되지 않음.
- ORM을 다른 기술로 바꿔도 도메인은 그대로.
- 단위 테스트가 DB 없이 가능.

## 핵심 어노테이션

```java
@Entity
@Table(name = "payments")
public class PaymentJpaEntity {
    @Id
    @Column(name = "id", nullable = false, updatable = false)
    private UUID id;

    @Column(name = "original_amount", nullable = false, precision = 15, scale = 2)
    private BigDecimal originalAmount;
    ...
}
```

| 어노테이션 | 역할 |
|-----------|------|
| `@Entity` | JPA가 관리할 클래스 |
| `@Table` | 매핑 테이블 이름 |
| `@Id` | 기본키 |
| `@Column` | 컬럼 매핑 + 제약 |
| `@OneToMany`, `@ManyToOne` | 관계 |
| `@Embedded` | 값 객체를 같은 테이블에 펴서 저장 |
| `@JdbcTypeCode(SqlTypes.JSON)` | JSON 컬럼 (Hibernate 6+) — `TrackingEventJpaEntity.detail` 에서 사용 |

## Spring Data Repository

인터페이스만 정의하면 구현체를 런타임에 자동 생성.

```java
public interface SpringDataPaymentRepository extends JpaRepository<PaymentJpaEntity, UUID> {
    Optional<PaymentJpaEntity> findByOrderId(UUID orderId);
}
```

- `JpaRepository`가 `save`, `findById`, `findAll`, `delete`, 페이징 등을 제공.
- **메서드 이름으로 쿼리 파생**: `findByOrderId` → `WHERE order_id = ?`.
- `@Query`로 JPQL/네이티브 SQL 직접 작성 가능.

## 페이징 / 정렬

```java
// orders/application/service/OrderQueryService.java
Pageable pageable = PageRequest.of(
    query.page(), query.size(),
    Sort.by(Sort.Direction.fromString(query.sortDir()), query.sortBy())
);
Page<Order> orders = orderRepository.findAll(pageable);
```
- `Page`는 데이터 + 메타정보(전체 개수, 페이지 수). 반환만 하면 컨트롤러에서 자동 JSON 직렬화.

## 트랜잭션

```java
@Service
@Transactional             // 메서드 단위로 트랜잭션
public class OrderCommandService { ... }

@Service
@Transactional(readOnly = true)   // 읽기 전용 (Hibernate 최적화)
public class OrderQueryService { ... }
```

- `@Transactional`은 프록시로 동작 → **public 메서드만**, **자기 호출은 미적용**.
- `readOnly = true`는 Hibernate flush를 건너뛰어 더 빠름.

## 더티 체크 (Dirty Checking)

JPA가 트랜잭션 안에서 영속 엔티티를 추적. 트랜잭션 종료 시점에 변경된 필드를 자동 감지하여 UPDATE.

```java
@Transactional
public void on(PaymentApprovedEvent event) {
    Order order = orderRepository.findById(...);
    order.transitionTo(OrderStatus.PAID);    // 도메인 객체만 변경
    // save() 호출 안 해도 커밋 시 UPDATE 실행
}
```
이 프로젝트는 도메인-JPA 분리 패턴이라 `save()`를 명시적으로 호출하는 형태(매 변경마다 새 JpaEntity로 변환). 그래서 더티 체크의 자동 UPDATE를 안 쓰지만, 일반 JPA 코드에서는 매우 강력한 기능.

## N+1 문제

`@OneToMany` 관계에서 부모 N개를 조회하면 자식 컬렉션 조회를 N번 하는 안티패턴.

해결:
- `@OneToMany(fetch = FetchType.EAGER)` — 위험 (모든 곳에 적용됨)
- `@EntityGraph` 또는 `JOIN FETCH` — 권장
- `OpenSessionInView` 끄기 (`open-in-view: false`) → 트랜잭션 밖 lazy access를 차단해 N+1을 빨리 발견

이 프로젝트는 `application.yml:16` 에 `open-in-view: false` 설정.

## HikariCP 커넥션 풀

기본 커넥션 풀. `application-prod.yml`에서:
```yaml
spring.datasource.hikari:
  maximum-pool-size: 20
  minimum-idle: 5
  connection-timeout: 30000
```
Virtual Threads를 켜도 DB 커넥션 수가 병목이 될 수 있어 적절한 사이즈 설정이 중요.

## FastAPI/SQLAlchemy 대응

| SQLAlchemy 2.0 (async) | Spring Data JPA |
|-----------------------|-----------------|
| `Mapped[T]` | `@Column` |
| `Session.scalars()` | `Repository.findXxx()` |
| `Repository` 직접 구현 | `JpaRepository` 자동 구현 |
| `Alembic` | `Flyway` |
| `expire_on_commit=False` | `open-in-view`/`readOnly` |
| `select(...).where(...)` | `@Query` 또는 메서드 이름 파생 |
| 도메인 ↔ ORM 매퍼 직접 작성 | 같음 (이 프로젝트 패턴) |
