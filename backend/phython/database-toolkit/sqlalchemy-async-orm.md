# SQLAlchemy 2.0 async ORM — Mapped, AsyncSession, select 표현식

> 운영 모니터링 시스템에서 SQLAlchemy 2.0 async + asyncpg 로 PostgreSQL 을 다루는 패턴.

---

## 0. 분석 대상 코드

### 0.1 모델 정의 — 2.0 스타일 `Mapped[...]`

```python
# monitoring/app/models/blacklist.py
from datetime import datetime, timezone
from sqlalchemy import BigInteger, Integer, String, Boolean, DateTime, Index
from sqlalchemy.orm import Mapped, mapped_column
from app.core.database import Base


class Blacklist(Base):
    __tablename__ = "blacklist"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
    ip: Mapped[str] = mapped_column(String(45), nullable=False)
    user_pk: Mapped[int | None] = mapped_column(Integer)
    request_id: Mapped[str | None] = mapped_column(String(36))
    reason: Mapped[str] = mapped_column(String(100), nullable=False)
    level: Mapped[str] = mapped_column(String(10), nullable=False)
    country: Mapped[str | None] = mapped_column(String(5))
    endpoint: Mapped[str | None] = mapped_column(String(50))
    violation_count: Mapped[int] = mapped_column(Integer, default=1)
    expires_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=lambda: datetime.now(timezone.utc)
    )
    released_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))

    __table_args__ = (
        Index("idx_blacklist_ip", "ip"),
        Index("idx_blacklist_active", "is_active"),
        Index("idx_blacklist_ip_active", "ip", unique=True, postgresql_where="is_active = TRUE"),
    )
```

### 0.2 Mixin 패턴 — 5개국 검색 로그 공통 컬럼

```python
# monitoring/app/models/search_log_base.py
from datetime import datetime, timezone
from sqlalchemy import BigInteger, Integer, String, Text, Date, DateTime
from sqlalchemy.orm import Mapped, mapped_column


class SearchLogCommonMixin:
    """국가별 검색 로그 테이블의 공통 컬럼"""
    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
    search_type: Mapped[str] = mapped_column(String(30), nullable=False)
    trademark_name: Mapped[str | None] = mapped_column(Text)
    sort_field: Mapped[str | None] = mapped_column(String(30))
    sort_order: Mapped[str | None] = mapped_column(String(10))
    page: Mapped[int | None] = mapped_column(Integer)
    size: Mapped[int | None] = mapped_column(Integer)
    response_count: Mapped[int | None] = mapped_column(Integer)
    processing_time_ms: Mapped[int | None] = mapped_column(Integer)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=lambda: datetime.now(timezone.utc)
    )

    # BaseCondition (12)
    trademark_type: Mapped[str | None] = mapped_column(Text)
    vienna_code: Mapped[str | None] = mapped_column(Text)
    classification: Mapped[str | None] = mapped_column(Text)
    similar_code: Mapped[str | None] = mapped_column(Text)
    goods_services: Mapped[str | None] = mapped_column(Text)
    application_number: Mapped[str | None] = mapped_column(Text)
    registration_number: Mapped[str | None] = mapped_column(Text)
    international_registration_number: Mapped[str | None] = mapped_column(Text)
    priority_number: Mapped[str | None] = mapped_column(Text)
    applicant: Mapped[str | None] = mapped_column(Text)
    applicant_country: Mapped[str | None] = mapped_column(Text)
    representative: Mapped[str | None] = mapped_column(Text)

    # DateRange (8)
    application_date_start: Mapped[datetime | None] = mapped_column(Date)
    application_date_end: Mapped[datetime | None] = mapped_column(Date)
    # ... 나머지
```

### 0.3 엔진/세션 초기화

```python
# monitoring/app/core/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase
from app.config import settings


class Base(DeclarativeBase):
    pass


engine = None
async_session = None


async def init_db():
    global engine, async_session
    engine = create_async_engine(
        settings.database_url,                  # postgresql+asyncpg://user:pwd@host/db
        pool_size=10,
        max_overflow=20,
        pool_pre_ping=True,
        pool_recycle=3600,
        echo=False,
    )
    async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


async def get_session() -> AsyncSession:
    if async_session is None:
        raise RuntimeError("Database not initialized. Call init_db() first.")
    async with async_session() as session:
        yield session


async def close_db():
    global engine
    if engine:
        await engine.dispose()
        engine = None
```

### 0.4 쿼리 — 통계 서비스 일부

```python
# monitoring/app/services/statistics_service.py
from sqlalchemy import func, select, text
from sqlalchemy.ext.asyncio import AsyncSession
from app.models.user_search_history import UserSearchHistory


async def get_overview(session: AsyncSession):
    now = datetime.now(KST)
    today_start = now.replace(hour=0, minute=0, second=0, microsecond=0)
    hour_ago = now.replace(minute=0, second=0, microsecond=0)

    # 오늘 국가별 검색 수
    result = await session.execute(
        select(
            UserSearchHistory.country,
            func.count().label("count"),
        )
        .where(UserSearchHistory.created_at >= today_start)
        .group_by(UserSearchHistory.country)
    )
    country_counts = {row.country: row.count for row in result}

    # raw SQL
    bl_result = await session.execute(
        text("SELECT COUNT(*) FROM blacklist WHERE is_active = TRUE")
    )
    active_blacklist = bl_result.scalar() or 0
```

---

## 1. SQLAlchemy 2.0 의 새 타입 모델

### 1.1 `DeclarativeBase`
```python
class Base(DeclarativeBase):
    pass
```

- 1.x 의 `declarative_base()` 함수 대신 클래스 상속.
- 정적 타입 검사 (mypy) 와 더 잘 어울림.

### 1.2 `Mapped[T]` + `mapped_column(...)`
```python
id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
ip: Mapped[str | None] = mapped_column(String(45))
```

- `Mapped[T]` 가 컬럼의 파이썬 타입.
- `Mapped[int | None]` ↔ SQL `NOT NULL` 안 걸림.
- `mapped_column(...)` 이 SQL 측 메타.
- IDE 자동완성: `model.id` 가 `int` 로 보임.

### 1.3 1.x → 2.x 마이그레이션
```python
# 1.x
class Blacklist(Base):
    id = Column(Integer, primary_key=True)
    ip = Column(String(45), nullable=False)
```
```python
# 2.x
class Blacklist(Base):
    id: Mapped[int] = mapped_column(primary_key=True)
    ip: Mapped[str] = mapped_column(String(45))
```

→ `Column` 도 여전히 동작하지만 새 코드는 `mapped_column` 권장.

---

## 2. Mixin 으로 5개국 동일 스키마 — DRY

```python
class SearchLogCommonMixin:
    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
    ...

class KoSearchLog(Base, SearchLogCommonMixin):
    __tablename__ = "ko_search_logs"
    # 추가 한국 전용 컬럼
    pronunciation_search: Mapped[str | None] = mapped_column(Text)

class UsSearchLog(Base, SearchLogCommonMixin):
    __tablename__ = "us_search_logs"
    # 미국 전용
```

**핵심**:
- `SearchLogCommonMixin` 은 `Base` 를 상속하지 않음 → 자체 테이블 만들지 않음.
- 국가별 클래스가 `Base + Mixin` 으로 → 공통 컬럼 + 국가별 추가.
- 컬럼 추가/변경은 한 곳에서.

**한계**:
- 5개 테이블이 나뉘므로 "모든 국가 검색 횟수" 같은 쿼리는 union all 이 필요.
- 국가가 추가될 때마다 모델 클래스 + 마이그레이션 필요.

---

## 3. 인덱스 — `__table_args__`

```python
__table_args__ = (
    Index("idx_blacklist_ip", "ip"),
    Index("idx_blacklist_active", "is_active"),
    Index("idx_blacklist_ip_active", "ip", unique=True, postgresql_where="is_active = TRUE"),
)
```

### Partial Index (`postgresql_where`)
- "is_active=TRUE 인 행에서 ip 가 unique" — 같은 IP 의 비활성 이력은 여러 개여도 됨.
- 활성 블랙리스트 1개만 보장하는 게 비즈니스 규칙.
- 일반 unique constraint 로는 표현 불가능.
- PostgreSQL 전용 기능.

### 인덱스 명명 규칙
- `idx_<table>_<columns>` 가 흔한 컨벤션.
- 명시적 이름이 alembic auto-generate 후에도 안정적으로 유지.

---

## 4. AsyncSession 의 라이프사이클

### 4.1 sessionmaker 와 expire_on_commit

```python
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
```

**`expire_on_commit=False`** 의 중요성:
- 기본 True: commit 후 객체 속성 접근 시 다시 DB 조회 (lazy reload).
- async 컨텍스트에서는 위험: 응답 직렬화 시점에 lazy load 가 발동 → `MissingGreenlet` 같은 에러 빈발.
- False 로 두면 commit 후에도 in-memory 값 유지 → 응답 직렬화 안전.

### 4.2 세션 컨텍스트
```python
async def get_session() -> AsyncSession:
    async with async_session() as session:
        yield session
```

- `async with` 안에서 세션 사용.
- 핸들러 종료 시 자동 close + 트랜잭션 롤백 (commit 안 했으면).
- FastAPI Depends 로 핸들러에 주입.

### 4.3 트랜잭션 명시
```python
async with session.begin():
    s.add(entry)
    s.add(other_entry)
    # 자동 commit
```
또는 명시적:
```python
s.add(entry)
await s.commit()
```

`session.begin()` 컨텍스트는 자동 commit / 예외 시 롤백.

---

## 5. select / where / group_by — 2.0 표현식

```python
result = await session.execute(
    select(
        UserSearchHistory.country,
        func.count().label("count"),
    )
    .where(UserSearchHistory.created_at >= today_start)
    .group_by(UserSearchHistory.country)
)
country_counts = {row.country: row.count for row in result}
```

### `select(...)` 의 동작
- 2.0 통합 select API. 1.x 의 `query()` 대신.
- 특정 컬럼만 선택: `select(Model.col1, Model.col2)`.
- 전체 모델: `select(Model)` → 결과는 모델 인스턴스.

### `result` 순회
- `for row in result`: Row 객체 yield.
- `row.country`, `row.count` 처럼 컬럼명/라벨로 접근.
- `result.scalar_one_or_none()`: 단일 스칼라.
- `result.scalars()`: 첫 컬럼만 추출.
- `result.all()`: 모두 리스트로.

### `func.count()` + `label`
- SQL `COUNT(*) AS count` 에 해당.
- 라벨이 row 의 attribute 이름이 됨.

---

## 6. Raw SQL — `text(...)`

```python
bl_result = await session.execute(text("SELECT COUNT(*) FROM blacklist WHERE is_active = TRUE"))
active_blacklist = bl_result.scalar() or 0
```

언제 raw SQL?
- ORM 으로 표현하기 복잡한 쿼리 (복잡한 윈도우 함수, recursive CTE 등).
- 성능이 중요해서 EXPLAIN 으로 직접 튜닝한 SQL.
- 빠르게 적은 수의 컬럼만 조회.

**주의**:
- 파라미터는 반드시 bind:
  ```python
  await session.execute(
      text("... WHERE created_at >= :hour_ago"),
      {"hour_ago": hour_ago},
  )
  ```
- 직접 f-string 으로 만들면 **SQL injection**.

---

## 7. 풀 옵션 — `pool_pre_ping`, `pool_recycle`

```python
engine = create_async_engine(
    settings.database_url,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,
    pool_recycle=3600,
)
```

| 옵션 | 의미 |
|------|------|
| `pool_size=10` | 풀 기본 크기 |
| `max_overflow=20` | 일시적 초과 허용 (10 + 20 = 30) |
| `pool_pre_ping=True` | 연결 빌릴 때 ping 으로 살아있는지 확인. 죽은 연결은 폐기 후 재생성 |
| `pool_recycle=3600` | 1시간 이상 된 연결 폐기 (DB 측 timeout 회피) |

### `pool_pre_ping` 의 가치
- DB 가 일시 단절 후 복구 → 풀 안의 stale 연결이 첫 쿼리에서 에러 → 핸들러 실패.
- pre_ping 으로 사용 직전 검증. 실패 시 자동 재생성.
- 비용: 매 빌림마다 1회 ping (간단한 SELECT 1).

### `pool_recycle` 의 가치
- 클라우드 LB / DB 측 idle timeout (예: 8시간) 으로 끊어진 stale 연결 회피.
- recycle 시간을 DB timeout 보다 짧게 둠.

---

## 8. UPSERT 패턴

```python
existing = await s.execute(select(Blacklist).filter_by(ip=ip, is_active=True))
row = existing.scalar_one_or_none()
if row:
    row.level = level
    row.expires_at = expires_at
else:
    s.add(Blacklist(ip=ip, ..., is_active=True))
await s.commit()
```

**SQLAlchemy 표준 ORM 으로의 UPSERT** 는 위처럼 select → 분기.

PostgreSQL 의 `ON CONFLICT` 를 직접 쓰려면:
```python
from sqlalchemy.dialects.postgresql import insert as pg_insert
stmt = pg_insert(Blacklist).values(ip=ip, level=level, ...)
stmt = stmt.on_conflict_do_update(
    index_elements=["ip"],
    set_=dict(level=stmt.excluded.level, expires_at=stmt.excluded.expires_at)
)
await s.execute(stmt)
```

→ 동시성 안전 (단일 SQL).

---

## 9. 응용 포인트

- 새 코드는 SQLAlchemy 2.0 + `Mapped[T]` + `mapped_column` + `select()` 표준화.
- 비슷한 테이블 여러 개는 Mixin 으로 DRY.
- async 환경에서는 `expire_on_commit=False` 거의 필수.
- 풀 옵션은 `pool_pre_ping=True`, `pool_recycle=3600` 기본값으로.
- Partial Index 같은 강력한 PostgreSQL 기능은 `__table_args__` 로 모델 정의 안에.
- 복잡한 쿼리는 raw `text(...)` + bind 파라미터.
- UPSERT 는 PostgreSQL `ON CONFLICT` 권장 (동시성).
