# Flyway — 데이터베이스 스키마를 코드로 관리하기

> 이 프로젝트의 `db/migration/V*.sql` 파일들이 **언제 / 어떻게 / 어떤 순서로** 적용되는지, 그리고 적용된 마이그레이션의 상태가 어떻게 추적되는지 정리한 학습 노트.

---

## 0. Flyway란

> 데이터베이스 마이그레이션을 코드 파일로 관리하고, 그 파일들을 **순서대로 정확히 한 번씩** 적용하며, 어떤 파일이 적용됐는지를 DB 자체에 기록하는 도구.

쉽게 말하면 **스키마용 git**이다. 빈 DB든, 운영 중인 DB든, 같은 마이그레이션 파일 시퀀스를 적용하면 같은 결과 스키마에 도달한다.

### 0.1 왜 필요한가

DB 스키마 관리의 흔한 함정들:

| 안 좋은 방식 | 문제 |
|------------|------|
| 누군가 운영 DB에 직접 `ALTER TABLE` | "내 머신에선 되는데 운영에선 안 되는" 사태 |
| ddl-auto: update / create | 개발은 편하지만 운영에선 위험 (의도 안 한 변경 / 데이터 손실) |
| 수동 SQL 파일 + README | 누가 어디까지 적용했는지 추적 불가 |
| 매번 수동 백업/복원 | 느리고 실수 잦음 |

**Flyway는 위 문제를 한 번에 푼다**: 마이그레이션을 파일로 관리하고, 적용 이력을 DB에 자동 기록.

---

## 1. 핵심 개념 — 마이그레이션 파일 명명 규칙

### 1.1 기본 형식

```
V<버전>__<설명>.sql
R__<설명>.sql
U<버전>__<설명>.sql   (옵션, undo)
```

이 프로젝트의 예:

```
common/src/main/resources/db/migration/
  ├ V1__init.sql              ← orders, payments
  ├ V2__outbox.sql            ← outbox 테이블 + 부분 인덱스
  ├ V4__outbox_dlq.sql        ← Phase 3에서 추가될 DLQ 테이블
notification/src/main/resources/db/migration/
  └ V3__inbox.sql             ← inbox 테이블
```

### 1.2 V vs R 차이

| 접두사 | 의미 | 언제 |
|------|------|-----|
| **V** (Versioned) | 한 번만 실행되는 변경. 적용 후엔 수정 금지 | 테이블 생성, 컬럼 추가, 인덱스 추가 등 대부분 |
| **R** (Repeatable) | 파일 내용이 바뀔 때마다 재실행 | 뷰 정의, 함수 정의처럼 "최신 상태로 덮어쓰기"가 의미 있는 것 |
| **U** (Undo) | V의 역연산 (Flyway Teams 유료 기능) | 보통은 안 쓰고 "보상 V 마이그레이션"으로 처리 |

### 1.3 명명 규칙의 엄격함

```
V1__init.sql           ← OK
V1.1__patch.sql        ← OK (subversion)
V2.0.1__hotfix.sql     ← OK
V1_init.sql            ← ❌ 언더스코어 두 개 필요
v1__init.sql           ← ❌ 대문자 V
```

언더스코어 **두 개**가 버전과 설명의 구분자다. 한 개로는 인식 못 함.

### 1.4 버전 번호 비교

Flyway는 마이그레이션을 적용할 때 **버전 번호 순**으로 정렬한다.

```
V1   < V1.1 < V2 < V2.0.1 < V3 < V10
```

semantic version 같은 점 표기가 가능. 보통 정수로 충분(V1, V2, V3...)하지만 hotfix 시 V1.1 같은 식으로 끼워 넣을 수 있다.

> **주의**: 한번 적용된 V를 수정하면 안 된다. 마이그레이션 시 체크섬 검증으로 잡힌다 (다음 절).

---

## 2. 상태 관리 — `flyway_schema_history` 테이블

### 2.1 자동 생성되는 추적 테이블

Flyway는 첫 실행 시 **`flyway_schema_history`** 라는 테이블을 자동으로 만들고, 그 안에 적용 이력을 기록한다.

```sql
SELECT * FROM flyway_schema_history;

 installed_rank | version |       description       |  type  |    script    | checksum   | installed_by |    installed_on    | execution_time | success
----------------+---------+-------------------------+--------+--------------+------------+--------------+--------------------+----------------+---------
              1 | 1       | init                    | SQL    | V1__init.sql | -845872319 | hybrid       | 2026-04-25 10:30   |             32 | t
              2 | 2       | outbox                  | SQL    | V2__outbox.sql | 1183920481 | hybrid       | 2026-04-25 10:30   |             18 | t
              3 | 3       | inbox                   | SQL    | V3__inbox.sql | 422118739  | hybrid       | 2026-04-25 10:31   |             12 | t
```

### 2.2 각 컬럼의 의미

| 컬럼 | 역할 |
|-----|------|
| `installed_rank` | 적용된 순서 |
| `version` | V 뒤의 버전 번호 (R 마이그레이션은 NULL) |
| `description` | `__` 뒤의 설명 |
| `type` | `SQL` 또는 `JDBC` (자바 코드 마이그레이션) |
| `script` | 실제 파일명 |
| **`checksum`** | **파일 내용의 해시 — 핵심** |
| `installed_by` | DB 사용자 |
| `installed_on` | 적용 시각 |
| `execution_time` | 적용 소요 시간 (ms) |
| `success` | 적용 성공 여부 |

### 2.3 체크섬 — "파일 변경을 잡는 안전장치"

Flyway는 매번 시작 시 다음을 한다.

```
[부팅 시 절차]
1. classpath에서 V*.sql, R*.sql 파일 스캔
2. flyway_schema_history와 비교
   ├─ 새 파일 (history에 없음) → 적용
   ├─ 이미 적용됨 + 체크섬 동일 → 건너뜀
   └─ 이미 적용됨 + 체크섬 다름 → 💥 예외 발생, 부팅 중단
3. R 파일은 체크섬이 바뀌면 재실행
```

**즉, 한번 적용된 V 파일을 수정하면 다음 부팅이 실패한다.**

```sql
-- V1__init.sql (이미 적용됨)
CREATE TABLE orders (id BIGSERIAL PRIMARY KEY, ...);

-- 누군가 이 파일에 컬럼을 한 줄 추가
CREATE TABLE orders (id BIGSERIAL PRIMARY KEY, ...);
ALTER TABLE orders ADD COLUMN comment TEXT;   -- ❌ 수정 금지

-- 다음 부팅
> Caused by: FlywayException: Validate failed: Migration checksum mismatch for migration V1__init.sql
```

이 안전장치가 **"테이블 변경은 새 V로 추가"** 라는 forward-only 규율을 강제한다.

### 2.4 forward-only 철학

- **이미 적용된 V는 절대 수정 안 한다.**
- 컬럼 추가가 필요하면 V5__add_comment_to_orders.sql 같은 새 파일.
- 컬럼 제거? 새 V로 `ALTER TABLE orders DROP COLUMN ...` 추가.
- 잘못 만든 마이그레이션? 그것을 되돌리는 새 V를 또 만든다 (보상 마이그레이션).

이 규율이 처음엔 답답하지만, 운영 DB에 일관된 시퀀스로 적용되도록 보장한다. **"히스토리는 항상 추가되고 절대 수정되지 않는다"** — git의 append-only 사상과 같다.

---

## 3. Spring Boot에서의 동작

### 3.1 자동 통합

`spring-boot-starter-data-jpa` + `flyway-core` + `flyway-database-postgresql`이 클래스패스에 있으면 Spring Boot가 자동으로:

1. 부팅 시 Flyway 빈을 만들고
2. `db.migration` classpath 위치 스캔
3. `migrate()` 호출 — DataSource 초기화 직후, JPA 엔티티 검증 직전

```
[Spring Boot 부팅 순서]
  1. DataSource 빈 생성 (HikariCP)
       ↓
  2. Flyway.migrate()  ← 여기서 V*.sql 적용
       ↓
  3. JPA EntityManagerFactory 생성
       ↓
  4. ddl-auto = validate → 엔티티와 실제 스키마 비교
       ↓
  5. Repository 빈 생성, 애플리케이션 ready
```

### 3.2 이 프로젝트의 의존성 위치

```groovy
// common/build.gradle
implementation 'org.flywaydb:flyway-core'
implementation 'org.flywaydb:flyway-database-postgresql'
```

- `flyway-core` — 마이그레이션 엔진 본체
- `flyway-database-postgresql` — PostgreSQL 전용 모듈 (Flyway 10+에서 분리됨, 필수)

> Flyway 9까지는 core만으로 모든 DB를 지원했지만, 10에서 DB별 모듈로 쪼개졌다. PostgreSQL 11+ 사용자는 `flyway-database-postgresql`을 명시적으로 추가해야 함.

### 3.3 `application.yml` 설정 (선택)

```yaml
spring:
  flyway:
    enabled: true               # 기본값 true
    locations: classpath:db/migration   # 기본값
    baseline-on-migrate: false  # 기존 DB에 처음 적용 시 true (주의 필요)
    validate-on-migrate: true   # 부팅 시 체크섬 검증
    out-of-order: false         # 누락 버전 허용 안 함
    schemas: public             # 적용 대상 스키마
```

이 프로젝트는 기본값으로 충분해서 별도 설정 없이 사용한다.

---

## 4. 멀티모듈에서의 동작 (이 프로젝트의 구조)

### 4.1 동시에 여러 모듈에 마이그레이션이 있다

```
common/src/main/resources/db/migration/
  ├ V1__init.sql
  ├ V2__outbox.sql
  └ V4__outbox_dlq.sql

notification/src/main/resources/db/migration/
  └ V3__inbox.sql
```

`app` 모듈을 부팅하면 모든 의존 모듈의 `db/migration` 폴더가 같은 classpath의 같은 위치(`db/migration/`)에 합쳐져 보인다.

```
[runtime classpath / db/migration]
  V1__init.sql       (from common)
  V2__outbox.sql     (from common)
  V3__inbox.sql      (from notification)
  V4__outbox_dlq.sql (from common)
```

Flyway는 출처를 신경 쓰지 않고 **버전 순으로** 적용한다.

### 4.2 버전 충돌 주의

여러 모듈이 같은 V 번호를 만들면 Flyway가 부팅 시 예외를 던진다.

```
common/db/migration/V3__foo.sql
notification/db/migration/V3__inbox.sql
→ FlywayException: Found more than one migration with version 3
```

**해결**: 모듈별 버전 범위를 정해 충돌 방지.

```
common         → V1, V2, V10~V19 (코어 테이블)
notification   → V3, V20~V29
order          → V30~V39
```

이 프로젝트는 단순해서 그냥 V1, V2, V3, V4 순으로 쓴다 — 새 모듈이 늘어나면 위 패턴으로 분할 권장.

### 4.3 모듈별 분리 vs 통합

| 전략 | 장점 | 단점 |
|------|------|------|
| 모든 마이그레이션을 한 모듈(common)에 | 적용 순서 통제 쉬움 | 도메인-스키마 결합도 ↑ |
| 모듈별로 분산 (이 프로젝트) | 모듈이 자기 테이블 소유 | 버전 범위 합의 필요 |
| 별도 `db-migration` 모듈 | 가장 깔끔한 분리 | 모듈 하나 추가 |

Phase 3에서 Notification을 분리할 때 **자기 마이그레이션을 자기 모듈에 들고 가게** 하려고 이 프로젝트는 분산을 택했다.

---

## 5. 자주 쓰는 명령

Spring Boot 자동 실행 외에 직접 다룰 일이 있다.

```bash
# Flyway Gradle 플러그인 사용 시
./gradlew flywayInfo       # 적용 상태 보기 (가장 자주 씀)
./gradlew flywayMigrate    # 마이그레이션 수동 실행
./gradlew flywayValidate   # 체크섬 검증만
./gradlew flywayRepair     # 깨진 history 복구 (실패한 마이그레이션 정리)
./gradlew flywayBaseline   # 기존 DB를 Flyway 추적 시작점으로 등록
./gradlew flywayClean      # ⚠️ 모든 객체 삭제 (운영 절대 금지)
```

### 5.1 가장 유용한 두 가지

**`flywayInfo`** — 어떤 V가 적용됐고, 어떤 게 펜딩인지 한눈에.

```
+-----------+---------+---------------------+------+---------------------+--------+
| Category  | Version | Description         | Type | Installed On        | State  |
+-----------+---------+---------------------+------+---------------------+--------+
| Versioned | 1       | init                | SQL  | 2026-04-25 10:30    | Success|
| Versioned | 2       | outbox              | SQL  | 2026-04-25 10:30    | Success|
| Versioned | 3       | inbox               | SQL  | 2026-04-25 10:31    | Success|
| Versioned | 4       | outbox dlq          | SQL  |                     | Pending|
+-----------+---------+---------------------+------+---------------------+--------+
```

**`flywayRepair`** — 마이그레이션이 실패했는데 history에 success=false로 남았을 때 정리.

### 5.2 위험한 것 — `flywayClean`

```
flywayClean → 스키마의 모든 테이블·뷰·인덱스를 통째로 DROP
```

개발 시 "처음부터 다시"용이지만, **운영에서 실행되면 데이터 전체 손실**. Flyway 10+는 `cleanDisabled: true`가 기본값이라 옵션을 켜야 동작한다.

```yaml
spring.flyway.clean-disabled: false   # ⚠️ 절대 운영에 적용 금지
```

---

## 6. forward-only를 어기고 싶을 때 — 보상 마이그레이션

V1에서 잘못 만든 컬럼을 빼고 싶다고 해보자. **V1을 수정하면 안 된다.** 대신:

```sql
-- V5__remove_unused_column.sql
ALTER TABLE orders DROP COLUMN deprecated_field;
```

이게 정석이다. 추적 테이블에는:

```
V1 init                ← 적용됨, 옛 컬럼 포함
V2~V4 ...
V5 remove_unused_column ← 적용됨, 옛 컬럼 제거
```

운영 DB의 현재 상태는 V1+V5 = "옛 컬럼 없음". 새 환경에서도 V1~V5 순서대로 적용되어 같은 결과.

### 6.1 데이터가 있는 컬럼을 옮길 때

```sql
-- V6__add_new_column.sql
ALTER TABLE orders ADD COLUMN new_field TEXT;

-- V7__copy_data.sql
UPDATE orders SET new_field = old_field;

-- V8__drop_old_column.sql (배포 후 충분히 안정화된 다음에)
ALTER TABLE orders DROP COLUMN old_field;
```

세 단계로 쪼개는 이유:
- 애플리케이션 배포는 **점진적 롤아웃**일 수 있음.
- 옛 코드가 남아 있는 동안엔 옛 컬럼도 살아있어야 함.
- 새 컬럼 채우기와 옛 컬럼 제거 사이에 **소크 윈도우** 둠.

이게 **expand-and-contract 패턴**이다.

---

## 7. Flyway vs 다른 도구

### 7.1 Hibernate `ddl-auto`와 비교

| | Flyway | Hibernate ddl-auto |
|--|--------|------------------|
| 적용 시점 | 명시적 (V 파일이 있을 때만) | 부팅 시 자동 |
| 의도 표현 | `ALTER TABLE ...` 직접 작성 | 엔티티 변경에서 추론 |
| 운영 안전성 | 높음 (코드로 검토 가능) | 낮음 (`update`/`create-drop` 위험) |
| DB 종속 기능 | 자유롭게 사용 | 제한적 (인덱스 옵션, JSONB 등) |
| 권장 운영 모드 | `migrate` (자동) | `validate` (검증만) |

이 프로젝트의 조합:
```yaml
spring.flyway: 자동 활성화
spring.jpa.hibernate.ddl-auto: validate
```

→ **Flyway가 스키마를 만들고, Hibernate는 자기 엔티티가 그 스키마에 맞는지 검증만**. 둘이 안 맞으면 부팅 실패.

### 7.2 Liquibase와 비교

| | Flyway | Liquibase |
|--|--------|-----------|
| 마이그레이션 형식 | SQL (또는 자바) | XML / YAML / JSON / SQL |
| 학습 곡선 | 낮음 (그냥 SQL) | 중간 (DSL 익혀야) |
| DB 종속성 | 높음 (각 DB의 SQL 문법 직접) | 낮음 (DSL이 추상화) |
| 롤백 | Teams(유료) 또는 보상 | 기본 제공 (`<rollback>`) |
| 인기도 | 자바 생태계에서 압도적 | 엔터프라이즈에서 강세 |

Flyway의 장점은 **"그냥 SQL"** 이라 학습이 빠르고, DB 고유 기능(PostgreSQL JSONB, 부분 인덱스 등)을 자유롭게 쓸 수 있다는 것.

---

## 8. 이 프로젝트의 마이그레이션 — 페이즈별 진화

### 8.1 Phase 1

```sql
-- V1__init.sql
CREATE TABLE orders (...);
CREATE TABLE payments (...);
```

도메인 엔티티(`Order`, `Payment`)와 1:1 매핑. **외래키 없음** — 도메인 분리 의지를 SQL 레벨에서도 유지.

### 8.2 Phase 2

```sql
-- V2__outbox.sql (common)
CREATE TABLE outbox (
    ..., status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    ...
);
CREATE INDEX idx_outbox_pending ON outbox(created_at) WHERE status = 'PENDING';   -- 부분 인덱스
```

**부분 인덱스**가 PostgreSQL의 강력한 기능. PENDING 상태일 때만 인덱싱 → 발행 후엔 인덱스가 작아짐.

```sql
-- V3__inbox.sql (notification)
CREATE TABLE inbox (
    ..., message_id VARCHAR(200) NOT NULL,
    ...
);
CREATE UNIQUE INDEX idx_inbox_message_id ON inbox(message_id);
```

UNIQUE 인덱스가 **멱등성의 최후 방어선**. 애플리케이션 레벨 체크가 경쟁 조건에 빠져도 DB가 막아줌.

### 8.3 Phase 3

```sql
-- V4__outbox_dlq.sql (common)
CREATE TABLE outbox_dlq (
    ..., original_outbox_id BIGINT NOT NULL,
    failure_reason TEXT,
    moved_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

DLQ는 새 테이블로 분리 — Phase 2의 outbox_status에서 진화.

### 8.4 가상의 미래 — Order에 컬럼 추가가 필요할 때

```sql
-- V10__add_orders_priority.sql
ALTER TABLE orders ADD COLUMN priority INT NOT NULL DEFAULT 0;
CREATE INDEX idx_orders_priority ON orders(priority) WHERE priority > 0;
```

V1을 절대 수정하지 않고 **새 V로 변경 분만** 추가. 이게 forward-only.

---

## 9. 자주 만나는 함정

### 9.1 함정 1 — 적용된 마이그레이션을 수정

```
> Caused by: FlywayException: Migration checksum mismatch for migration V2__outbox.sql
```

**해결**:
- 의도한 변경이라면 새 V 파일로.
- 정말 수정해야 한다면 (개발 환경) `flywayRepair`로 history 갱신.

### 9.2 함정 2 — 운영 DB에 처음 Flyway 도입 시

기존 운영 DB에 테이블이 이미 있는데 Flyway를 처음 적용하면:

```
> FlywayException: Found non-empty schema(s) "public" but no schema history table
```

**해결**: `flywayBaseline`으로 현 상태를 기준점(보통 V0)으로 등록.

```yaml
spring.flyway.baseline-on-migrate: true
spring.flyway.baseline-version: 0
```

이러면 V1부터 적용. 기존 객체와 충돌하지 않게 V1을 작성해야 함.

### 9.3 함정 3 — 마이그레이션 도중 실패

V3이 절반쯤 실행되다 SQL 에러로 실패하면:

- Flyway는 history에 `success=false`로 기록.
- 다음 부팅 시 펜딩으로 인식되지 않음 (이미 시도됨).
- **한 번 더 부팅**해도 같은 V3을 재실행하지 않음.

**해결**: `flywayRepair`로 실패 기록 정리 → V3 파일 수정 → 다시 부팅.

### 9.4 함정 4 — `flywayClean` 실수

운영에서 절대 실행 금지. CI 환경에서도 신중. 안전장치:

```yaml
spring.flyway.clean-disabled: true   # 기본값 (Flyway 10+)
```

운영 환경 `application-prod.yml`에 명시적으로 못 박아 두는 게 안전.

### 9.5 함정 5 — 테스트마다 다른 Flyway 동작

Testcontainers는 매번 새 컨테이너를 띄우니 Flyway가 항상 V1부터 적용한다 — 일관성 보장됨. 하지만 **컨테이너 재사용**(`.withReuse(true)`) 켜면 한번 적용된 마이그레이션이 다음 테스트에 남아 있어 버그처럼 보일 수 있음.

```bash
# 의심되면 컨테이너 강제 재시작
docker rm -f $(docker ps -aq --filter "label=org.testcontainers")
```

---

## 10. 디버깅 도구

### 10.1 적용 상태 확인

```sql
-- 직접 DB에 접속해서
SELECT installed_rank, version, description, success, installed_on
FROM flyway_schema_history
ORDER BY installed_rank;
```

또는 Spring Boot Actuator의 `/actuator/flyway` 엔드포인트.

### 10.2 다음 부팅에서 무엇이 적용될지 미리 보기

```bash
./gradlew flywayInfo
```

`Pending` 상태인 V들이 다음 `migrate` 시 적용된다.

### 10.3 SQL 로그 보기

```yaml
logging.level.org.flywaydb: DEBUG
```

각 마이그레이션의 실행 SQL과 소요 시간이 콘솔에 찍힌다.

---

## 11. 이 프로젝트에서 Flyway가 보장하는 것

1. **로컬 개발자 머신, CI, 운영의 스키마가 항상 동일하다**
   같은 V 파일 시퀀스가 적용되므로.

2. **Testcontainers가 띄운 깨끗한 PostgreSQL이 매번 같은 스키마로 시작된다**
   통합 테스트의 결정성 확보.

3. **JPA 엔티티 검증(`ddl-auto: validate`)이 의미를 가진다**
   Flyway가 만든 테이블과 엔티티가 일치하지 않으면 부팅이 막힘 → 스키마 드리프트 방지.

4. **Phase 진화가 추적 가능하다**
   V1(Phase 1) → V2/V3(Phase 2) → V4(Phase 3) — 페이즈별 변경이 그대로 마이그레이션 시퀀스에 매핑.

---

## 12. 한 줄 요약

> **Flyway는 DB 스키마용 git이다 — 변경을 V 파일로 작성하고, `flyway_schema_history` 테이블에 적용 이력을 추적하며, forward-only 규율로 어떤 환경이든 같은 시퀀스로 같은 결과 스키마에 도달하게 한다.**
> 이 프로젝트는 `application.yml` 한 줄 설정 없이 Spring Boot 자동 통합 + classpath의 V 파일만으로 모든 마이그레이션이 자동 적용되며, `ddl-auto: validate`와 짝을 이뤄 **"코드와 스키마가 영원히 일치한다"** 는 보장을 만든다.

---

## 13. 함께 보면 좋은 자료

- 공식: [Flyway Documentation](https://documentation.red-gate.com/fd)
- 공식: [Spring Boot — Database Initialization with Flyway](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto.data-initialization.migration-tool.flyway)
- `docs/study/spring-boot-4-migration.md` — Spring Boot 4.0의 모듈 분리와 함께 Flyway 의존성 패턴
- `docs/study/testcontainers.md` — 테스트마다 깨끗한 DB로 Flyway가 매번 V1부터 적용하는 시나리오
- `docs/phase-1-tdd.md` Step 0.3 — V1__init.sql 작성
- `docs/phase-2-tdd.md` Step 1, 4 — V2(outbox), V3(inbox) 작성
- `docs/phase-3-tdd.md` Step 2 — V4(outbox_dlq) 작성

---

## 14. 결정 가이드

```
스키마 변경이 필요하다
   │
   ├ 컬럼/테이블 추가, 인덱스 추가
   │   → 새 V 파일을 만든다 (forward-only)
   │
   ├ 컬럼 이름 변경
   │   → V1: ADD new_column / V2: UPDATE / V3: DROP old_column
   │     (expand-and-contract)
   │
   ├ 뷰/함수 변경
   │   → R 파일로 (재실행 가능)
   │
   ├ 운영 데이터 마이그레이션이 큼 (대용량 백필)
   │   → V로 작성하되 batched UPDATE 또는 별도 백필 잡 검토
   │
   └ 한 번 적용된 V를 수정하고 싶음
       → 99% 잘못된 충동. 새 V로 차이를 표현하자.
```

이 규율을 지키면 운영 DB의 스키마는 코드와 함께 진화하면서도 절대 신비로운 상태가 되지 않는다.
