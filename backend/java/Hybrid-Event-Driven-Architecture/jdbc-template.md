# Spring `JdbcTemplate` — JDBC 보일러플레이트를 죽이는 도구

> 이 프로젝트의 `OutboxMigrationTest`, `OutboxCleanerTest`, `OutboxCleaner` 등이 사용하는 `JdbcTemplate`을 풀어 정리한 학습 노트.
> 언제 JPA 대신 JdbcTemplate을 쓰는가, 어떤 메서드를 어떤 경우에 쓰는가, 안전한 사용 패턴이 무엇인가.

---

## 0. JdbcTemplate이란

> JDBC API의 보일러플레이트(connection 열고/닫기, exception 변환, ResultSet 매핑)를 자동화한 Spring의 가장 오래된 데이터 액세스 도구.

순수 JDBC 코드와 비교하면 즉시 가치가 보인다.

### 0.1 순수 JDBC

```java
public List<String> findAllOrderIds() throws SQLException {
    Connection conn = null;
    PreparedStatement ps = null;
    ResultSet rs = null;
    try {
        conn = dataSource.getConnection();
        ps = conn.prepareStatement("SELECT id FROM orders WHERE status = ?");
        ps.setString(1, "CONFIRMED");
        rs = ps.executeQuery();
        List<String> ids = new ArrayList<>();
        while (rs.next()) {
            ids.add(rs.getString("id"));
        }
        return ids;
    } finally {
        if (rs != null) rs.close();
        if (ps != null) ps.close();
        if (conn != null) conn.close();
    }
}
```

### 0.2 JdbcTemplate

```java
public List<String> findAllOrderIds() {
    return jdbc.queryForList(
        "SELECT id FROM orders WHERE status = ?",
        String.class,
        "CONFIRMED");
}
```

같은 일, 두 줄.

---

## 1. 어디서 오는가, 어디로 들어가는가

### 1.1 자동 구성

```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
// 또는 'org.springframework.boot:spring-boot-starter-jdbc'
```

`spring-boot-starter-jdbc`는 `JdbcTemplate` 빈을 자동 등록한다. `spring-boot-starter-data-jpa`는 starter-jdbc를 transitive로 끌고 와서 같은 효과.

### 1.2 주입

```java
@Autowired JdbcTemplate jdbc;
```

또는 생성자 주입:

```java
@Component
public class OutboxCleaner {
    private final JdbcTemplate jdbc;
    public OutboxCleaner(JdbcTemplate jdbc) { this.jdbc = jdbc; }
}
```

### 1.3 `DataSource`와의 관계

`JdbcTemplate`은 내부에 `DataSource`를 들고 있고, 매번 거기서 connection을 빌려 쓴다.

```
DataSource (HikariCP 풀)
   ↓ getConnection()
Connection (풀에서 빌림)
   ↓
JdbcTemplate.queryForList(...)
   ↓
실행 후 자동 close (풀에 반납)
```

직접 connection을 만지지 않아도 되는 게 핵심.

---

## 2. 핵심 메서드

### 2.1 SELECT — 단일 값

```java
Integer count = jdbc.queryForObject(
    "SELECT COUNT(*) FROM outbox WHERE status = 'PENDING'",
    Integer.class);
```

`queryForObject`는 **정확히 1행 1열**을 기대. 0개면 `EmptyResultDataAccessException`, 2개 이상이면 `IncorrectResultSizeDataAccessException`.

### 2.2 SELECT — 단일 레코드

```java
OutboxEvent e = jdbc.queryForObject(
    "SELECT id, aggregate_type, event_type FROM outbox WHERE id = ?",
    (rs, rowNum) -> {
        OutboxEvent ev = new OutboxEvent();
        // rs.getLong("id") 등으로 매핑
        return ev;
    },
    42L);
```

두 번째 인자가 `RowMapper<T>`. 람다로 매핑.

### 2.3 SELECT — 여러 레코드

```java
List<String> messageIds = jdbc.queryForList(
    "SELECT message_id FROM inbox WHERE event_type = ?",
    String.class,
    "OrderConfirmed");
```

```java
List<OutboxEvent> events = jdbc.query(
    "SELECT * FROM outbox WHERE status = 'PENDING' ORDER BY created_at LIMIT 100",
    (rs, rowNum) -> mapToOutbox(rs));
```

### 2.4 SELECT — 컬럼별 Map

```java
List<Map<String,Object>> rows = jdbc.queryForList(
    "SELECT column_name FROM information_schema.columns WHERE table_name = 'outbox'");

rows.forEach(row -> System.out.println(row.get("column_name")));
```

이 프로젝트의 `OutboxMigrationTest`가 정확히 이 형태를 쓴다 — 스키마 메타데이터 조회.

### 2.5 INSERT / UPDATE / DELETE

```java
int updated = jdbc.update(
    "UPDATE outbox SET status = 'PUBLISHED' WHERE id = ?",
    42L);
```

`update`는 영향받은 행 수를 반환. 이 프로젝트의 `OutboxCleaner`:

```java
@Scheduled(cron = "0 0 3 * * *")
@Transactional
public void run() {
    int deleted = jdbc.update("""
        DELETE FROM outbox
        WHERE status = 'PUBLISHED'
          AND published_at < NOW() - INTERVAL '7 days'
    """);
    log.info("outbox cleanup: deleted {} rows", deleted);
}
```

### 2.6 DDL / 임의 SQL

```java
jdbc.execute("CREATE INDEX IF NOT EXISTS idx_foo ON foo(bar)");
```

DDL은 결과가 없으니 `execute()` 사용.

### 2.7 메서드 요약

| 메서드 | 용도 | 반환 |
|-------|------|------|
| `queryForObject(sql, type, args...)` | 단일 값 | T |
| `queryForObject(sql, RowMapper, args...)` | 단일 레코드 | T |
| `queryForList(sql, type, args...)` | 단일 컬럼 여러 행 | List\<T\> |
| `queryForList(sql, args...)` | 여러 컬럼 여러 행 | List\<Map\<String,Object\>\> |
| `query(sql, RowMapper, args...)` | 객체 여러 행 | List\<T\> |
| `update(sql, args...)` | INSERT/UPDATE/DELETE | int (영향 행 수) |
| `execute(sql)` | DDL 또는 결과 없는 명령 | void |
| `batchUpdate(sql, batchArgs)` | 배치 쓰기 | int[] |

---

## 3. 안전한 파라미터 바인딩 — `?` placeholder

### 3.1 절대 문자열 연결 금지

```java
// ❌ SQL 인젝션 위험
jdbc.queryForList("SELECT * FROM orders WHERE customer_id = " + customerId);

// ✅ placeholder + 파라미터
jdbc.queryForList("SELECT * FROM orders WHERE customer_id = ?", Long.class, customerId);
```

JdbcTemplate이 내부에서 `PreparedStatement`로 바인딩 → SQL 인젝션 차단.

### 3.2 Named Parameter — `NamedParameterJdbcTemplate`

`?`은 위치 기반이라 인자가 많으면 헷갈린다. 이름 기반은 `NamedParameterJdbcTemplate`:

```java
NamedParameterJdbcTemplate named = new NamedParameterJdbcTemplate(dataSource);
named.queryForList(
    "SELECT * FROM outbox WHERE status = :status AND retry_count < :max",
    Map.of("status", "PENDING", "max", 10),
    OutboxEvent.class);
```

복잡한 쿼리는 named가 가독성 큼.

---

## 4. JdbcTemplate vs JPA — 언제 무엇을 쓰는가

이 프로젝트는 **둘을 혼용**한다. 어떤 기준인가?

### 4.1 JPA (Spring Data JPA, `JpaRepository`)

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    long countByStatus(OrderStatus status);
}

orderRepository.findById(42L).ifPresent(o -> o.confirm());
```

- 도메인 객체를 영속/조회.
- 엔티티 간 관계, dirty checking, 영속성 컨텍스트.
- **도메인 모델을 그대로 다룰 때**.

### 4.2 JdbcTemplate

```java
jdbc.update("DELETE FROM outbox WHERE published_at < NOW() - INTERVAL '7 days'");
jdbc.queryForObject("SELECT COUNT(*) FROM pg_indexes WHERE indexname = ?", Integer.class, "idx_outbox_pending");
```

- DB 메타데이터 조회 (information_schema, pg_indexes 등).
- 대량 UPDATE/DELETE (영속성 컨텍스트 거치면 느림).
- DB 고유 SQL 기능 (CTE, window function, JSONB 연산자, FOR UPDATE SKIP LOCKED 등).
- 마이그레이션·정리 잡 같이 도메인 객체가 의미 없는 작업.

### 4.3 비교 표

| 측면 | JPA | JdbcTemplate |
|------|-----|------------|
| 추상 수준 | 객체-관계 매핑 | SQL + 파라미터 |
| 학습 곡선 | 가파름 (영속성 컨텍스트, fetch, lazy 등) | 낮음 (SQL만 알면 됨) |
| DB 종속 기능 | 제한적 (네이티브 쿼리로 우회) | 자유 |
| 성능 (대량) | 느림 (영속성 컨텍스트 오버헤드) | 빠름 |
| 도메인 표현력 | 풍부 | 없음 |
| 권장 영역 | 비즈니스 로직, 도메인 객체 | 메타·정리·운영 작업 |

### 4.4 이 프로젝트의 분담

| 작업 | 도구 |
|------|------|
| Order/Payment/Notification 도메인 CRUD | JPA (`JpaRepository`) |
| Outbox/Inbox 엔티티 | JPA |
| Outbox PENDING 폴링 | JPA (`findTop100ByStatus...`) |
| **DB 스키마 메타 조회** (마이그레이션 검증 테스트) | **JdbcTemplate** |
| **7일 지난 outbox 일괄 DELETE** (Phase 3 OutboxCleaner) | **JdbcTemplate** |
| FOR UPDATE SKIP LOCKED 행 잠금 (Phase 3) | JPA의 `@Query(nativeQuery=true)` (또는 JdbcTemplate) |

---

## 5. 이 프로젝트에서의 실제 사용 사례

### 5.1 OutboxMigrationTest — 스키마 검증

```java
class OutboxMigrationTest extends KafkaIntegrationTestBase {

    @Autowired JdbcTemplate jdbc;

    @Test
    void outbox_테이블이_존재하고_필수_컬럼을_가진다() {
        List<Map<String,Object>> cols = jdbc.queryForList("""
            SELECT column_name FROM information_schema.columns
            WHERE table_name = 'outbox'
        """);
        assertThat(cols).extracting(c -> c.get("column_name"))
            .contains("id","aggregate_type","aggregate_id", ...);
    }

    @Test
    void 부분_인덱스가_PENDING에만_걸려있다() {
        Integer count = jdbc.queryForObject("""
            SELECT COUNT(*) FROM pg_indexes
            WHERE tablename = 'outbox' AND indexname = 'idx_outbox_pending'
        """, Integer.class);
        assertThat(count).isEqualTo(1);
    }
}
```

**왜 JPA가 아닌가**:
- `information_schema`, `pg_indexes`는 PostgreSQL 시스템 카탈로그. 엔티티로 매핑할 필요 없음.
- "스키마가 정확히 만들어졌는가"의 검증이라 도메인 객체 추상이 무의미.

### 5.2 OutboxCleaner — 대량 DELETE

```java
@Component
public class OutboxCleaner {

    private final JdbcTemplate jdbc;

    @Scheduled(cron = "0 0 3 * * *")
    @Transactional
    public void run() {
        int deleted = jdbc.update("""
            DELETE FROM outbox
            WHERE status = 'PUBLISHED'
              AND published_at < NOW() - INTERVAL '7 days'
        """);
        log.info("outbox cleanup: deleted {} rows", deleted);
    }
}
```

**왜 JPA가 아닌가**:
- 대상 행 수가 수만~수십만 가능. JPA로 `findAll().deleteAll()`하면 영속성 컨텍스트에 다 올라가서 OOM 위험.
- 단일 SQL 한 번이 가장 효율적.
- `INTERVAL '7 days'` 같은 PostgreSQL 표현을 그대로 쓸 수 있음.

### 5.3 OutboxCleanerTest — 시간 강제 조작

```java
@Test
void 발행된지_7일_이상된_outbox_레코드는_삭제된다() {
    OutboxEvent old = outboxRepo.save(OutboxEvent.of("Order","1","t","{}"));
    old.markPublished();
    // 시간 강제로 8일 전으로 변경
    jdbc.update("UPDATE outbox SET published_at = NOW() - INTERVAL '8 days' WHERE id = ?",
        old.getId());

    cleaner.run();

    assertThat(outboxRepo.findById(old.getId())).isEmpty();
}
```

**왜 JPA가 아닌가**:
- `published_at`은 도메인 의미상 `markPublished()`가 자동으로 채우는 필드. 도메인 메서드로는 **임의 시각** 설정 불가능.
- 테스트는 시간을 강제로 조작해야 하므로 SQL로 직접 UPDATE.

---

## 6. 트랜잭션 — JdbcTemplate도 `@Transactional` 안에서

JdbcTemplate은 자동으로 트랜잭션 매니저와 연동. `@Transactional` 메서드 안에서 호출하면 **같은 트랜잭션 경계**에서 동작.

```java
@Transactional
public void cleanupAndMetric() {
    int deleted = jdbc.update("DELETE FROM outbox WHERE ...");
    metricsRepo.save(new CleanupMetric(deleted));
    // 둘 다 같은 트랜잭션. 예외 발생 시 함께 롤백.
}
```

JPA Repository와 JdbcTemplate을 한 메서드 안에서 섞어 써도 같은 트랜잭션:

```java
@Transactional
public void recover() {
    int reset = jdbc.update("UPDATE outbox SET status = 'PENDING' WHERE status = 'DEAD_LETTER'");
    auditRepo.save(new RecoveryAudit(reset));   // JPA
}
```

Spring 트랜잭션 추상이 둘을 묶어준다.

---

## 7. 자주 만나는 함정

### 7.1 함정 1 — `queryForObject` 결과 0개

```java
Integer count = jdbc.queryForObject("SELECT COUNT(*) FROM ...", Integer.class);
```

`COUNT(*)`는 항상 한 행 반환이라 안전. 하지만:

```java
String name = jdbc.queryForObject(
    "SELECT name FROM users WHERE id = ?", String.class, 9999L);
// id가 없으면 → EmptyResultDataAccessException
```

→ `query`로 받아서 `Optional`로 감싸거나, `try/catch`.

### 7.2 함정 2 — JPA와 JdbcTemplate의 1차 캐시 불일치

```java
@Transactional
void weird() {
    Order o = orderRepo.findById(42L).orElseThrow();   // 영속성 컨텍스트에 들어감
    o.setStatus(OrderStatus.CANCELLED);                // dirty marking, 아직 flush 안 됨

    jdbc.queryForObject("SELECT status FROM orders WHERE id = ?",
        String.class, 42L);
    // → "CREATED" (dirty 변경 전 값) 받음. 영속성 컨텍스트와 DB 상태 불일치
}
```

JPA의 dirty checking은 트랜잭션 commit 시점에 flush된다. JdbcTemplate은 그걸 모름.

→ JPA + JdbcTemplate 혼용 시 `entityManager.flush()` 명시 또는 한쪽으로 통일.

### 7.3 함정 3 — JSONB 같은 PostgreSQL 타입

```java
jdbc.update("UPDATE outbox SET payload = ? WHERE id = ?", payloadJson, 42L);
// PostgreSQL이 String을 JSONB로 자동 캐스팅 못 할 수 있음
```

→ `?::jsonb` 명시 캐스트:

```java
jdbc.update("UPDATE outbox SET payload = ?::jsonb WHERE id = ?", payloadJson, 42L);
```

### 7.4 함정 4 — Connection 누수

JdbcTemplate은 자동으로 close 처리하지만, 직접 `dataSource.getConnection()`을 호출하면 누수 발생.

```java
// ❌ 위험
Connection c = dataSource.getConnection();
// ... close 안 함

// ✅ 안전
jdbc.query(...);   // 알아서 close
```

### 7.5 함정 5 — 너무 많은 결과를 한 번에 메모리에

```java
List<Map<String,Object>> all = jdbc.queryForList("SELECT * FROM huge_table");
// 행 100만개면 OOM
```

→ `RowCallbackHandler` 또는 `ResultSetExtractor`로 스트리밍 처리:

```java
jdbc.query("SELECT * FROM huge_table", rs -> {
    while (rs.next()) {
        process(rs);   // 한 행씩 처리
    }
});
```

---

## 8. JdbcTemplate vs JdbcClient (Spring 6.1+)

Spring 6.1(2023)에서 더 모던한 fluent API `JdbcClient`가 추가됨.

```java
// JdbcTemplate
List<String> names = jdbc.queryForList(
    "SELECT name FROM users WHERE active = ?", String.class, true);

// JdbcClient
List<String> names = jdbcClient
    .sql("SELECT name FROM users WHERE active = :active")
    .param("active", true)
    .query(String.class)
    .list();
```

| 측면 | JdbcTemplate | JdbcClient |
|------|------------|----------|
| API 스타일 | 고전 (메서드 오버로딩 많음) | 빌더/체이닝 |
| named param | 별도 클래스(`NamedParameterJdbcTemplate`) | 기본 지원 |
| Spring 버전 | 1.x부터 | 6.1+ |

이 프로젝트는 학습 친화성을 위해 `JdbcTemplate`을 쓰지만, 새 코드라면 `JdbcClient`도 좋은 선택.

---

## 9. 한 줄 요약

> **`JdbcTemplate`은 SQL을 직접 다루면서 connection/exception/매핑 보일러플레이트만 제거해주는 가장 얇은 추상이다.** 도메인 객체를 다룰 땐 JPA가, 메타·정리·운영 작업을 할 땐 JdbcTemplate이 자연스럽다.
> 이 프로젝트에선 `OutboxMigrationTest`(스키마 검증), `OutboxCleaner`(7일 지난 행 일괄 삭제), `OutboxCleanerTest`(시각 강제 조작) 같은 곳에 쓴다 — JPA로 표현하기 어색하거나 비효율적인 작업들.

---

## 10. 함께 보면 좋은 자료

- 공식: [Spring JDBC Data Access](https://docs.spring.io/spring-framework/reference/data-access/jdbc.html)
- 공식: [JdbcTemplate Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/core/JdbcTemplate.html)
- `docs/study/flyway-migrations.md` — JdbcTemplate으로 검증하는 마이그레이션
- `docs/study/cross-cutting-infrastructure.md` — Spring Data JPA Repository와 JdbcTemplate의 역할 분담
- `docs/phase-2-tdd.md` Step 1.1 — `OutboxMigrationTest`가 `information_schema`를 조회하는 사례
- `docs/phase-3-tdd.md` Step 3 — `OutboxCleaner`의 일괄 DELETE
