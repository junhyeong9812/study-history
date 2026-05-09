# redis-py async — 연결 관리와 기본 명령

> `redis.asyncio` (구 aioredis) 의 연결 풀, primary/replica 분리, 기본 명령 사용법.

---

## 0. 분석 대상 코드

```python
# monitoring/app/core/redis_client.py
import logging
import redis.asyncio as aioredis
from app.config import settings

logger = logging.getLogger(__name__)

_primary: aioredis.Redis | None = None
_replica: aioredis.Redis | None = None


async def init_redis():
    global _primary, _replica
    try:
        _primary = aioredis.Redis(
            host=settings.REDIS_PRIMARY_HOST,
            port=settings.REDIS_PRIMARY_PORT,
            decode_responses=True,
        )
        await _primary.ping()
        logger.info(f"Redis Primary 연결 성공: {settings.REDIS_PRIMARY_HOST}:{settings.REDIS_PRIMARY_PORT}")
    except Exception as e:
        logger.error(f"Redis Primary 연결 실패: {e}")
        _primary = None

    try:
        _replica = aioredis.Redis(
            host=settings.REDIS_REPLICA_HOST,
            port=settings.REDIS_REPLICA_PORT,
            decode_responses=True,
        )
        await _replica.ping()
    except Exception as e:
        logger.error(f"Redis Replica 연결 실패: {e}")
        _replica = None


async def close_redis():
    global _primary, _replica
    if _primary:
        await _primary.close()
        _primary = None
    if _replica:
        await _replica.close()
        _replica = None


def get_primary() -> aioredis.Redis | None:
    return _primary


def get_replica() -> aioredis.Redis | None:
    return _replica
```

---

## 1. `redis.asyncio` 의 정체

- 과거에는 `aioredis` 패키지가 별도로 있었다. Python 3.11+ 시점부터는 **redis-py 가 내장**.
- `import redis.asyncio as aioredis` 는 redis-py 의 async 클라이언트 모듈 alias.
- API 가 동기 redis-py 와 거의 동일하지만 모든 명령이 코루틴.

설치:
```
pip install redis>=4.2  # async 내장
```

---

## 2. `Redis(...)` 생성자 옵션

```python
aioredis.Redis(
    host=settings.REDIS_PRIMARY_HOST,
    port=settings.REDIS_PRIMARY_PORT,
    decode_responses=True,
)
```

| 옵션 | 의미 |
|------|------|
| `host`, `port` | 접속 정보 |
| `db` (default 0) | Redis DB 번호 (0~15) |
| `password` | AUTH 비밀번호 |
| `decode_responses` | True 면 응답을 자동 str 디코드. False 면 bytes |
| `max_connections` | 풀 최대 연결 수 |
| `socket_timeout` / `socket_connect_timeout` | I/O 타임아웃 (초) |
| `health_check_interval` | N초마다 ping 으로 연결 검증 |

### 2.1 `decode_responses` 의 함정

`True`:
```python
val = await r.get("k")  # → "value" (str)
```

`False` (기본):
```python
val = await r.get("k")  # → b"value" (bytes)
val.decode()             # 매번 디코드 필요
```

위 코드는 True 로 두어 핸들러에서 디코드 안 해도 되게 함. 단점: binary safe 가 필요한 데이터(이미지 등)는 손상.

---

## 3. 연결 풀의 암묵적 동작

`Redis(...)` 인스턴스는 **연결 풀** 을 내부에 갖는다 (직접 ConnectionPool 안 만들어도 자동 생성).

```python
r = aioredis.Redis(host="localhost")
await r.ping()       # 풀에서 한 연결 빌려서 사용
await r.set("a", 1)  # 다른 연결 또는 같은 연결
```

**기본 동작**:
- 명령 단위로 풀에서 연결을 빌리고 반환.
- 동시 호출 시 풀에서 여러 연결 동시 사용.
- 풀 크기 기본 무제한 → 대규모 동시성에서는 `max_connections` 설정 권장.

명시적 풀:
```python
pool = aioredis.ConnectionPool(host="localhost", max_connections=10)
r = aioredis.Redis(connection_pool=pool)
```

---

## 4. 헬스체크 — `await r.ping()`

```python
await _primary.ping()
```

- 응답: `True`. 실패 시 예외.
- 시작 시 연결 검증에 필수.
- 운영 중에는 `health_check_interval=30` 옵션으로 자동.

위 코드의 패턴:
- ping 실패 시 `_primary = None` 설정.
- 호출 측에서 `if not primary:` 체크 + 재시도 가능.
- 즉, **Redis 가 죽어도 앱은 뜨고**, 백오프 재연결.

---

## 5. Primary/Replica 분리

```python
def get_primary() -> aioredis.Redis | None: return _primary
def get_replica() -> aioredis.Redis | None: return _replica
```

**왜 분리?**
- Primary: 쓰기 + 중요 읽기 (Streams 의 XREADGROUP/XACK).
- Replica: 읽기 전용 부하 분산 (단순 캐시 조회 등).

**주의**:
- Replica 는 비동기 복제 → 방금 쓴 값을 즉시 읽을 수 없음 (eventually consistent).
- 트랜잭션이나 strict consistency 필요한 곳은 Primary.

이 프로젝트는 사용 분기가 단순:
- Streams = Primary 만.
- 단순 SET/GET 캐시 = Replica.

---

## 6. close 와 자원 정리

```python
async def close_redis():
    if _primary:
        await _primary.close()
```

**`close()` vs `aclose()`**:
- 구버전 `close()` 가 deprecated → 신권장은 `aclose()`.
- 동작은 동일: 풀의 모든 연결 종료.

lifespan 과 결합:
```python
@asynccontextmanager
async def lifespan(app):
    await init_redis()
    yield
    await close_redis()
```

---

## 7. 자주 쓰는 명령 정리

### 7.1 String
```python
await r.set("key", "value", ex=60)        # TTL 60초
await r.setnx("key", "v")                 # 없을 때만
await r.get("key")
await r.incr("counter")
await r.expire("key", 30)
await r.ttl("key")
await r.delete("key1", "key2")
```

### 7.2 Hash
```python
await r.hset("user:1", mapping={"name": "alice", "age": "30"})
await r.hget("user:1", "name")
await r.hgetall("user:1")
await r.hincrby("user:1", "visits", 1)
```

### 7.3 List
```python
await r.lpush("queue", "task1")
await r.rpop("queue")
await r.brpop("queue", timeout=5)   # 블로킹 pop
```

### 7.4 Set
```python
await r.sadd("blacklist", "1.2.3.4")
await r.sismember("blacklist", "1.2.3.4")
await r.smembers("blacklist")
```

### 7.5 Sorted Set (랭킹)
```python
await r.zadd("score", {"alice": 90, "bob": 80})
await r.zrange("score", 0, -1, withscores=True, desc=True)
```

### 7.6 Stream
```python
await r.xadd("mystream", {"data": json.dumps(event)}, maxlen=100_000, approximate=True)
await r.xreadgroup(group, consumer, {"mystream": ">"}, count=10, block=5000)
await r.xack("mystream", group, msg_id)
```

상세는 [streams-consumer-group.md](streams-consumer-group.md).

---

## 8. 파이프라인 (배치 명령)

```python
async with r.pipeline() as pipe:
    pipe.set("a", 1)
    pipe.set("b", 2)
    pipe.incr("counter")
    results = await pipe.execute()
```

→ N개 명령을 한 번의 round-trip 으로. throughput 크게 향상.

**transaction 모드** (`pipeline(transaction=True)`):
- MULTI/EXEC 로 atomic 실행.
- 다만 redis 의 transaction 은 강한 isolation 이 아님 (관찰자 측에서 부분 결과 보일 수 있음 — WATCH/MULTI 스킴 참고).

---

## 9. Pub/Sub vs Streams 의 redis-py 차이

```python
# Pub/Sub
async with r.pubsub() as ps:
    await ps.subscribe("chan")
    async for msg in ps.listen():
        ...

# Streams
result = await r.xreadgroup(group, consumer, {"s": ">"}, count=10, block=5000)
```

- Pub/Sub: 실시간, 휘발성, 컨슈머 그룹 없음.
- Streams: 영속, 재처리 가능, 컨슈머 그룹 지원.

이벤트 모니터링/로깅에는 Streams 가 거의 항상 정답.

---

## 10. 응용 포인트

- 단일 Redis 인스턴스라도 풀 인스턴스 1개를 모듈 전역으로 두고 lifespan 으로 init/close.
- `decode_responses=True` 권장 (단 binary safe 필요 시 False).
- 주요 동작 시작 전 `ping()` 으로 연결 검증.
- Primary/Replica 분리는 읽기 부하가 큰 시스템에 유효.
- 빈번한 명령은 `pipeline` 으로 round-trip 절약.
- 운영 모니터링은 Streams 가 거의 항상 정답.
