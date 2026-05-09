# Flyway

> DB 스키마 마이그레이션 도구. 버전 관리되는 SQL을 순서대로 적용.

## 개념

코드처럼 DB 스키마도 git에 추적되어야 한다. Flyway는 `V1__*.sql`, `V2__*.sql` 같은 **순서가 있는 SQL 파일** 을 DB에 한 번씩만 적용하고, 적용 이력을 `flyway_schema_history` 테이블에 기록한다.

## 이 프로젝트의 마이그레이션

```
src/main/resources/db/migration/
├── V1__create_subscriptions.sql
├── V2__create_orders.sql
├── V3__create_payments.sql
├── V4__create_shipments.sql
└── V5__create_tracking.sql
```

각 페이즈가 진행되면서 새 모듈의 테이블이 추가됐다. 한 번 푸시되어 실행된 V1을 절대 수정하지 않고, 변경이 필요하면 V6를 추가하는 식.

## 파일명 규칙

```
V<버전>__<설명>.sql        # 일반 마이그레이션 (한 번만)
R__<설명>.sql              # 반복 (체크섬이 바뀌면 재실행, 뷰/함수에 적합)
U<버전>__<설명>.sql        # 언두 (Pro 기능)
```

언더스코어 두 개(`__`)가 구분자. 한 개는 인식 안 됨.

## 자동 실행

`spring-boot-starter-data-jpa` 와 `flyway-core` 가 클래스패스에 있으면 Spring Boot가 부팅 시 자동 실행.

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
```

`spring.jpa.hibernate.ddl-auto: validate` 와 함께 쓰면 안전:
- Flyway가 스키마를 만들고
- Hibernate는 검증만 (테이블/컬럼 누락 시 부팅 실패)
- `ddl-auto: create` 같은 자동 DDL은 위험 → 운영에서 금지

## 동작 원리

1. 부팅 시 `flyway_schema_history` 테이블 조회 (없으면 생성).
2. `db/migration/` 폴더의 파일 스캔.
3. 적용 안 된 버전을 순서대로 실행.
4. 각 파일 적용 후 history에 (version, description, checksum, success) INSERT.
5. 다음 부팅 시 이미 적용된 파일은 체크섬만 비교. **이미 실행된 파일을 수정하면 부팅 실패** (의도된 안전장치).

## 운영 주의사항

```yaml
# application-prod.yml
spring:
  flyway:
    clean-disabled: true    # 운영에서 clean (모든 테이블 drop) 절대 금지
```

- **이미 적용된 V1 수정 금지**: 체크섬 불일치 → 부팅 실패. 수정이 필요하면 V6 추가.
- **롤백은 신중히**: V1을 되돌리려면 직접 SQL 작성 + history 수동 정리.
- **데이터 마이그레이션은 분리**: 스키마 변경(V6)과 데이터 백필(V7)을 분리하면 디버깅이 쉬워짐.
- **큰 테이블 + DDL 동시 적용 주의**: PostgreSQL에서 `ALTER TABLE ... ADD COLUMN NOT NULL`은 큰 테이블에서 락. `ADD COLUMN ... DEFAULT ...`는 11+에서는 즉시 적용.

## 검증 명령

```bash
# 어떤 마이그레이션이 적용 대기인지 확인
./gradlew flywayInfo

# 다음 미적용 마이그레이션 적용
./gradlew flywayMigrate

# 강제로 체크섬 재계산 (체크섬 불일치 복구)
./gradlew flywayRepair
```

## FastAPI/Alembic 대응

| Alembic | Flyway |
|---------|--------|
| `alembic revision -m "..."` | 손으로 `V<n>__name.sql` 생성 |
| Python 스크립트 + `op.create_table()` | 순수 SQL |
| `alembic upgrade head` | 부팅 시 자동 |
| `alembic downgrade -1` | 직접 작성 (또는 Pro의 U-스크립트) |
| `alembic_version` 테이블 | `flyway_schema_history` 테이블 |
| 자동 diff 생성 | 없음 (수동 SQL) |

Alembic은 ORM 모델을 보고 자동 diff를 만들어주는 게 강점이지만, Flyway는 순수 SQL이라 DBA가 검토하기 쉽고 DB 특수 기능(파티셔닝, 인덱스 옵션)을 그대로 쓸 수 있다.
