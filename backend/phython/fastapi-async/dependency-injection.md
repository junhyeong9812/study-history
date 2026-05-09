# FastAPI Dependency Injection — `Depends` 와 의존성 yield 패턴

> "DB 세션을 라우터에 어떻게 넘기지?" 같은 질문에 대한 표준 답.
> Pydantic Query 검증과 결합한 실전 예제까지.

---

## 0. 분석 대상 코드

모니터링 대시보드의 실제 라우터.

```python
# monitoring/app/api/dashboard.py
from typing import Optional
from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.database import get_session
from app.services import statistics_service

router = APIRouter(prefix="/api/dashboard", tags=["Dashboard"])


@router.get("/overview")
async def overview(session: AsyncSession = Depends(get_session)):
    return await statistics_service.get_overview(session)


@router.get("/search-stats")
async def search_stats(
    country: Optional[str] = Query(None),
    period: str = Query("daily"),
    date_from: Optional[str] = Query(None, alias="from"),
    date_to: Optional[str] = Query(None, alias="to"),
    session: AsyncSession = Depends(get_session),
):
    return await statistics_service.get_search_stats(session, country, period, date_from, date_to)


@router.get("/trademark-ranking")
async def trademark_ranking(
    country: Optional[str] = Query(None),
    limit: int = Query(100, ge=1, le=1000),
    session: AsyncSession = Depends(get_session),
):
    return await statistics_service.get_trademark_ranking(session, country, limit)
```

세션 공급자:

```python
# monitoring/app/core/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = None
async_session = None


async def init_db():
    global engine, async_session
    engine = create_async_engine(
        settings.database_url,
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
```

---

## 1. `Depends` 의 본질

`Depends(get_session)` 의 의미: "이 핸들러가 호출되기 전에 `get_session()` 을 호출해서 그 결과를 `session` 인자에 넣어줘."

핵심 효과:
1. **자원 라이프사이클 관리**: `yield` 가 있는 의존성은 응답 후 정리.
2. **테스트에서 교체**: `app.dependency_overrides[get_session] = mock_session_factory` 로 통째로 갈아끼움.
3. **재사용**: 한 의존성을 여러 라우터가 공유. 권한 체크 같은 횡단 관심사에 효과적.
4. **트리 의존성**: 의존성이 또 다른 의존성을 가질 수 있음.

---

## 2. `yield` 의존성 — 자원 정리 보장

`get_session` 은 **제너레이터 의존성**이다.

```python
async def get_session() -> AsyncSession:
    async with async_session() as session:
        yield session
```

흐름:
1. 요청 도착.
2. FastAPI 가 `get_session()` 호출 → `async with` 진입 → `session` 획득.
3. `yield session` 으로 핸들러에 세션 주입.
4. 핸들러 실행.
5. **핸들러 종료 (정상/예외 모두)** → 제너레이터 재개 → `async with` 정상 종료 → 세션 close + 풀로 반환.

**`async with` 의 의미**:
- `async_sessionmaker` 가 만들어주는 세션은 컨텍스트 매니저.
- `__aexit__` 에서 트랜잭션 롤백 (commit 안 했으면) + 풀 반환.

**핵심 효과**:
- 핸들러가 예외를 던져도 세션은 반드시 정리됨 → 풀 누수 없음.
- 핸들러 안에서 `await session.commit()` 만 명시적으로 호출하면 됨.

---

## 3. `Query(...)` 와 Pydantic 의 결합

```python
limit: int = Query(100, ge=1, le=1000)
country: Optional[str] = Query(None)
date_from: Optional[str] = Query(None, alias="from")
```

`Query` 는 Pydantic `Field` 의 쿼리스트링 버전. 같은 검증 옵션 (`ge`, `le`, `regex`, `min_length` 등) 사용 가능.

### 3.1 alias 의 쓰임

```python
date_from: Optional[str] = Query(None, alias="from")
```

`from` 은 파이썬 예약어 → 매개변수명으로 못 씀. alias 로 외부 이름과 내부 이름 분리.

호출: `?from=2026-01-01&to=2026-01-31`
파이썬 변수: `date_from`, `date_to`.

### 3.2 정규식 검증 (`pattern`)

```python
sort: str = Query("count_desc", regex="^(count_desc|count_asc|recent)$")
```

→ enum 처럼 동작. 잘못 들어오면 자동 422.

(주의: Pydantic v2 / FastAPI 0.100+ 에서는 `regex` 대신 `pattern` 권장.)

### 3.3 필수 쿼리 — `...`

```python
user_pk: int = Query(...)
```

`Ellipsis` 가 "필수". 빠지면 422.

---

## 4. 트리 의존성 — 의존성이 의존성을 가짐

```python
async def get_session() -> AsyncSession:
    async with async_session() as s:
        yield s

async def get_current_user(
    token: str = Header(...),
    session: AsyncSession = Depends(get_session),
) -> User:
    user = await session.scalar(select(User).where(User.token == token))
    if not user:
        raise HTTPException(401)
    return user

@router.get("/me")
async def me(user: User = Depends(get_current_user)):
    return user
```

→ 핸들러는 `user` 만 받지만, 그 뒤에 `session → token 검증 → user 조회` 트리가 자동 실행.

**같은 의존성은 한 요청에서 한 번만 실행** (캐시). `session` 을 두 군데서 `Depends` 해도 같은 인스턴스가 주입됨 → 한 트랜잭션 보장.

캐시 끄려면: `Depends(get_session, use_cache=False)`.

---

## 5. 테스트 시 의존성 교체

```python
# tests/conftest.py
from app.main import app
from app.core.database import get_session

async def override_get_session():
    async with TEST_SESSION() as s:
        yield s

app.dependency_overrides[get_session] = override_get_session
```

→ 운영 DB 세션 대신 테스트 DB 세션 주입. 라우터 코드는 그대로.

테스트 후 정리:
```python
@pytest.fixture(autouse=True)
def clear_overrides():
    yield
    app.dependency_overrides.clear()
```

---

## 6. 클래스 의존성 — `Depends(SomeClass)`

복잡한 의존성은 클래스로:

```python
class Pagination:
    def __init__(self, page: int = Query(1, ge=1), size: int = Query(50, ge=1, le=500)):
        self.page = page
        self.size = size
        self.offset = (page - 1) * size

@router.get("/items")
async def list_items(p: Pagination = Depends()):
    # Depends() 빈 호출 → 타입에서 추론
    return {"page": p.page, "offset": p.offset, ...}
```

→ Query 검증을 한 클래스로 묶고, 라우터들이 `Pagination` 만 의존.

---

## 7. 모듈 전역 vs 의존성 — 위 코드의 선택 분석

위 `get_session` 은 모듈 전역 `async_session` 팩토리에서 세션을 꺼낸다. 즉:
- 풀 자체는 모듈 전역 (`engine`, `async_session`).
- 세션은 요청마다 새로 만들어 주입.

대안: `app.state.engine` 에 풀을 두고 `Request.app.state.engine` 으로 접근. 테스트 격리에 유리하지만 코드는 약간 복잡.

이 프로젝트는 **단일 앱 인스턴스 가정** + 단순함을 우선 → 모듈 전역.

---

## 8. 응용 포인트

- DB 세션, 외부 클라이언트, 권한 체크 — 모두 `Depends` 로.
- 자원 라이프사이클이 있으면 `yield` 의존성. `try/except/finally` 로 예외 정리도 가능.
- Query 검증은 Pydantic Field 와 동일한 규칙 — 핸들러에서 if문 검증 금지.
- 테스트에서는 `dependency_overrides` 로 깔끔히 교체.
- 횡단 관심사 (인증/감사 로깅/요청 ID 부여) 도 의존성으로 풀면 라우터가 비즈니스 로직만 갖게 됨.
