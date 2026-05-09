# 다중 DB 클라이언트 — MariaDB, PostgreSQL, MongoDB, Elasticsearch

> 한 시스템에서 4종 DB 를 다루는 실전 — 각자 적합한 라이브러리/패턴 정리.

---

## 1. 4종 DB 의 역할 분담

| DB | 라이브러리 | 용도 |
|----|-----------|------|
| **MariaDB** | SQLAlchemy + pymysql (동기) | 원본 트레이드마크 데이터 (5국가별) |
| **PostgreSQL** | SQLAlchemy 2.0 + asyncpg (비동기) | 모니터링 로그/통계 |
| **MongoDB** | motor (비동기) | 트레이드마크명 사전, 발음 사전 |
| **Elasticsearch** | elasticsearch-py (동기) | 검색 인덱스 |

각 DB 의 강점 활용:
- MariaDB: 정형화된 비즈니스 데이터, 트랜잭션.
- PostgreSQL: JSONB + Partial Index 같은 강력한 SQL 기능.
- MongoDB: 자유 스키마 사전 (단어 표제어가 동적).
- ES: 한글 형태소·발음 검색.

---

## 2. MariaDB (동기 SQLAlchemy + pymysql)

### 2.1 코드

```python
# example-app/src/database/maria.py
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker, Session
from sqlalchemy.pool import NullPool


class MariaDBConnection:
    def __init__(self):
        self.engine = None
        self.SessionLocal = None
        self._initialize_connection()

    def _build_connection_url(self) -> str:
        return (
            f"mysql+pymysql://{settings.MARIADB_USER}:{settings.MARIADB_PASSWORD}"
            f"@{settings.MARIADB_HOST}:{settings.MARIADB_PORT}/{settings.MARIADB_DATABASE}"
            f"?charset={settings.MARIADB_CHARSET}"
        )

    def _initialize_connection(self):
        database_url = self._build_connection_url()
        self.engine = create_engine(
            database_url,
            poolclass=NullPool,
            echo=False,
            connect_args={
                "connect_timeout": 10,
                "read_timeout": 300,
                "write_timeout": 60,
                "charset": settings.MARIADB_CHARSET
            }
        )
        self.SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=self.engine)
        self._test_connection()

    def _test_connection(self):
        with self.engine.connect() as conn:
            result = conn.execute(text("SELECT 1"))
            result.fetchone()
```

### 2.2 핵심 결정

#### `mysql+pymysql` vs `mysql+mysqlclient` vs `mysql+aiomysql`
| 드라이버 | 동기/비동기 | 비고 |
|---------|----------|------|
| pymysql | 동기, pure Python | 설치 쉬움, 성능 보통 |
| mysqlclient | 동기, C 확장 | 빠름, 설치 시 빌드 의존성 |
| aiomysql | 비동기 | async/await 환경 |
| asyncmy | 비동기, 더 빠름 | aiomysql 의 fork |

이 프로젝트는 ProcessPoolExecutor 워커 환경 → 동기 + pymysql 단순함 우선.

#### `NullPool` 의 의미
- 풀링 안 함. 매 연결마다 새 TCP.
- ProcessPoolExecutor 환경에서 풀이 fork 되면 동시 사용 위험 → NullPool 로 회피.
- 단일 프로세스 환경이면 일반 풀이 더 효율적.

#### `connect_args` 의 timeout
- `connect_timeout=10`: 연결 자체 10초 안에 안 되면 실패.
- `read_timeout=300`: 큰 쿼리 결과 5분 대기.
- `write_timeout=60`: write 쿼리 1분.

---

## 3. PostgreSQL (비동기 SQLAlchemy + asyncpg)

### 3.1 코드

```python
# monitoring/app/core/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession


async def init_db():
    global engine, async_session
    engine = create_async_engine(
        settings.database_url,   # postgresql+asyncpg://user:pwd@host/db
        pool_size=10,
        max_overflow=20,
        pool_pre_ping=True,
        pool_recycle=3600,
        echo=False,
    )
    async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


async def get_session() -> AsyncSession:
    if async_session is None:
        raise RuntimeError("Database not initialized")
    async with async_session() as session:
        yield session
```

### 3.2 asyncpg 의 강점

- **C 기반 파서** → 가장 빠른 PG 드라이버.
- **PG 네이티브 프로토콜** 직접 사용 (libpq 안 거침).
- 자동 prepared statement 캐시.
- numpy/pandas 와 잘 통합.

### 3.3 alembic 마이그레이션

PostgreSQL 측은 alembic 사용 — `alembic.ini` + `alembic/versions/`.

```bash
alembic revision --autogenerate -m "add blacklist table"
alembic upgrade head
```

자동 생성된 마이그레이션도 항상 사람 검토 — `op.alter_column` 등 위험한 변경은 다운타임 영향 고려.

---

## 4. MongoDB (motor — 비동기)

### 4.1 코드

```python
# monitoring/app/core/mongodb_client.py
from motor.motor_asyncio import AsyncIOMotorClient, AsyncIOMotorDatabase, AsyncIOMotorCollection
from app.config import settings

_client: AsyncIOMotorClient | None = None
_db: AsyncIOMotorDatabase | None = None
_collection: AsyncIOMotorCollection | None = None


async def init_mongodb():
    global _client, _db, _collection
    if not settings.MONGO_URI:
        logger.warning("MONGO_URI 미설정 — MongoDB 비활성화")
        return

    try:
        _client = AsyncIOMotorClient(settings.MONGO_URI)
        await _client.admin.command("ping")
        _db = _client[settings.MONGO_DB]
        _collection = _db[settings.MONGO_COLLECTION]
        doc_count = await _collection.estimated_document_count()
        logger.info(f"MongoDB 연결 성공: {settings.MONGO_DB}.{settings.MONGO_COLLECTION} ({doc_count}건)")
    except Exception as e:
        logger.error(f"MongoDB 연결 실패: {e}")
        _client = None
        _db = None
        _collection = None


async def close_mongodb():
    global _client, _db, _collection
    if _client:
        _client.close()
        _client = None
        _db = None
        _collection = None


def get_db() -> AsyncIOMotorDatabase | None:
    return _db


def get_collection() -> AsyncIOMotorCollection | None:
    return _collection
```

### 4.2 motor 의 정체

- `pymongo` 의 비동기 wrapper.
- API 가 pymongo 와 거의 동일 + `await` 추가.
- 내부적으로 thread pool 위에 async 인터페이스.

### 4.3 motor vs pymongo

| | pymongo | motor |
|--|--------|------|
| 동기/비동기 | 동기 | 비동기 |
| 사용 환경 | 스크립트, 워커 | FastAPI/asyncio |
| 코드 차이 | `coll.find_one()` | `await coll.find_one()` |

### 4.4 자주 쓰는 명령

```python
# 단일 조회
doc = await coll.find_one({"name": "삼성"})

# 여러 개 — async cursor
async for doc in coll.find({"country": "KR"}).limit(100):
    process(doc)

# 모두 메모리에
docs = await coll.find({"country": "KR"}).to_list(length=1000)

# 삽입
await coll.insert_one({"name": "X", ...})
await coll.insert_many([{...}, {...}])

# 업데이트
await coll.update_one({"_id": id}, {"$set": {"field": "v"}}, upsert=True)

# 집계
pipeline = [{"$match": {"country": "KR"}}, {"$group": {"_id": "$applicant", "count": {"$sum": 1}}}]
async for row in coll.aggregate(pipeline):
    ...
```

### 4.5 estimated_document_count vs count_documents
- `estimated_document_count()`: 메타데이터 기반. 매우 빠름. 정확하지 않을 수 있음.
- `count_documents({})`: 실제 카운트. 느릴 수 있음 (큰 컬렉션은 인덱스 활용).

위 코드의 시작 시 카운트는 estimated 가 적합 — "대략 얼마 있는지" 만 로깅.

### 4.6 init 실패 허용
```python
except Exception as e:
    logger.error(f"MongoDB 연결 실패: {e}")
    _client = None
```

→ MongoDB 가 일시 장애여도 앱 시작은 진행. 호출 측에서 `if not _collection: ...` 체크.

---

## 5. Elasticsearch (동기 elasticsearch-py)

상세는 [elasticsearch-py-bulk.md](elasticsearch-py-bulk.md).

핵심:
- `Elasticsearch(...)` 동기 클라이언트.
- `helpers.parallel_bulk` 로 대량 색인.
- `client.search(index, body=query)` 로 DSL 쿼리.

비동기 환경이면 `AsyncElasticsearch`.

---

## 6. 한 앱에서 여러 DB 의 Lifespan 통합

```python
# monitoring/app/main.py (개념)
@asynccontextmanager
async def lifespan(app):
    await init_db()         # PostgreSQL
    await init_redis()      # Redis (primary + replica)
    await init_mongodb()    # MongoDB
    # ES 클라이언트는 사용 시점에 생성 (sync)
    yield
    await close_mongodb()
    await close_redis()
    await close_db()
```

각 init/close 가 짝.

### 부분 실패 허용 vs 강한 시작
- 모니터링/로깅 시스템 → MongoDB 단절 OK, 앱은 떠야 함.
- 핵심 기능 DB (PostgreSQL) 단절 → 앱 시작 자체를 막을지 결정 필요.

위 코드는 init 함수가 자체적으로 try/except → 단절돼도 앱은 떠짐. 호출 측에서 None 체크.

---

## 7. 트랜잭션의 차이

| DB | 트랜잭션 |
|----|---------|
| MariaDB | InnoDB ACID |
| PostgreSQL | ACID, 더 강한 isolation 옵션 |
| MongoDB | 4.0+ 다중 문서 트랜잭션 (단일 셀에서 권장) |
| Elasticsearch | **트랜잭션 없음**. 단일 문서 atomic |

→ "DB 간 atomic" 은 Saga / 보상 트랜잭션으로 해결. 한 트랜잭션에 묶을 수 없음.

이 프로젝트의 정합성:
- ES 색인 실패 시 PG 측 success 카운트 안 올림.
- 멱등 색인(`_id` 고정) 으로 재시도 안전.
- 부분 실패 허용 — 일부 데이터는 다음 마이그레이션에서 다시.

---

## 8. 응용 포인트

- 동기 환경 (워커 프로세스, 스크립트) → SQLAlchemy + pymysql/psycopg2, 동기 ES, pymongo.
- 비동기 환경 (FastAPI) → SQLAlchemy + asyncpg/asyncmy, AsyncElasticsearch, motor.
- 풀 동작이 fork 와 충돌 위험 시 NullPool.
- 비핵심 DB 의 init 실패는 silent fail + 호출 측 None 체크.
- DB 간 atomic 은 어렵다 — 멱등 + 보상 트랜잭션 설계.
