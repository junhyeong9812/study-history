# FastAPI Lifespan — 백그라운드 컴포넌트 시작/종료 관리

> 콜렉터, Redis 풀, 스케줄러 같은 **장기 실행 컴포넌트**를 FastAPI 앱과 함께 시작/종료시키는 패턴.

---

## 0. 분석 대상 코드

통합 관제 앱(orchestration)의 lifespan. 콜렉터·Redis 풀·스케줄러 3개의 백그라운드 컴포넌트를 묶어서 관리한다.

```python
# orchestration/app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI

from app.monitor.collector import start_collector, stop_collector
from app.migration.stream_publisher import init_redis, close_redis
from app.migration.scheduler import start_scheduler, stop_scheduler
from app.api.migration_ws import send_agent_command


@asynccontextmanager
async def lifespan(app: FastAPI):
    await start_collector()
    await init_redis()
    await start_scheduler(send_command=send_agent_command)
    yield
    await stop_scheduler()
    await close_redis()
    await stop_collector()


app = FastAPI(
    title="Deploy Dashboard",
    version="1.0.0",
    lifespan=lifespan,
)
```

---

## 1. lifespan 의 본질

### 1.1 왜 `@asynccontextmanager`?

`contextlib.asynccontextmanager` 데코레이터는 **`yield` 가 있는 async 제너레이터를 async 컨텍스트 매니저로 변환**한다.

```python
# 위 코드는 사실상 아래와 동등하다:
class Lifespan:
    async def __aenter__(self):
        await start_collector()
        await init_redis()
        await start_scheduler(send_command=send_agent_command)
        return None  # yield 가 값 없이 호출됨
    async def __aexit__(self, exc_type, exc, tb):
        await stop_scheduler()
        await close_redis()
        await stop_collector()
```

데코레이터가 없으면 직접 클래스를 만들어야 함 → 데코레이터로 함수 형태가 가능해짐.

### 1.2 yield 의 역할

- `yield` **이전**: startup. 앱이 트래픽 받기 직전 실행됨.
- `yield` **이후**: shutdown. SIGTERM 등으로 앱이 종료될 때 실행됨.
- `yield` 가 반환하는 값(없으면 `None`) 은 `app.state` 처럼 의존성 주입에 쓸 수도 있음 (`yield {"redis": pool}`).

### 1.3 `@app.on_event("startup")` 과의 관계

구버전:
```python
@app.on_event("startup")
async def startup():
    await start_collector()

@app.on_event("shutdown")
async def shutdown():
    await stop_collector()
```

신버전(권장):
```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    await start_collector()
    yield
    await stop_collector()

app = FastAPI(lifespan=lifespan)
```

**왜 lifespan 이 더 나은가**:
- startup/shutdown 이 **하나의 함수에 묶임** → 시작·종료 짝 맞추기 쉬움.
- `try/finally` 로 부분 실패 처리 가능.
- ASGI 표준의 lifespan 프로토콜과 더 잘 맞음.

---

## 2. 시작/종료 짝 맞추기 — try/finally

위 코드는 **시작 중 한 단계가 실패하면 그 이후 단계는 실행되지 않고, finally도 없어서** 이미 시작된 컴포넌트가 정리되지 않는다는 약점이 있다. 안전 버전:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    started = []
    try:
        await start_collector(); started.append(stop_collector)
        await init_redis();      started.append(close_redis)
        await start_scheduler(send_command=send_agent_command)
        started.append(stop_scheduler)
        yield
    finally:
        for stop in reversed(started):
            try:
                await stop()
            except Exception as e:
                logger.error(f"shutdown error: {e}", exc_info=True)
```

**원칙**:
- 정리는 **역순**으로 (의존 관계 역방향).
- 정리 도중 예외가 나도 **다른 정리 단계는 계속**한다.
- 시작 중 실패했어도 `started` 에 등록된 만큼은 정리한다.

---

## 3. lifespan 안에서 띄우는 컴포넌트 3종 패턴

### 3.1 폴링 콜렉터 (asyncio.Task)

```python
# orchestration/app/monitor/collector.py (단순화)
import asyncio
_task: asyncio.Task | None = None
_stop_event = asyncio.Event()

async def _run():
    while not _stop_event.is_set():
        try:
            await collect_once()
        except Exception as e:
            logger.error(f"collect failed: {e}")
        try:
            await asyncio.wait_for(_stop_event.wait(), timeout=10.0)
        except asyncio.TimeoutError:
            pass

async def start_collector():
    global _task
    _stop_event.clear()
    _task = asyncio.create_task(_run())

async def stop_collector():
    _stop_event.set()
    if _task:
        await _task
```

**핵심**:
- `asyncio.create_task` 로 백그라운드 실행 → lifespan startup 이 빠르게 끝남.
- `asyncio.Event` 로 그레이스풀 종료 신호 전달.
- `asyncio.wait_for(event.wait(), timeout=N)` 패턴으로 "N초마다 깨거나 종료 신호로 즉시 종료".
- 단순 `asyncio.sleep(N)` 만 쓰면 stop 이 N초까지 대기해야 함.

### 3.2 외부 자원 풀 (Redis/DB)

```python
# stream_publisher.py
import redis.asyncio as redis
_pool: redis.Redis | None = None

async def init_redis():
    global _pool
    _pool = redis.from_url(settings.REDIS_URL, max_connections=10, decode_responses=True)
    await _pool.ping()

async def close_redis():
    global _pool
    if _pool:
        await _pool.aclose()
        _pool = None
```

**핵심**:
- 모듈 전역에 풀을 두고, lifespan 에서 init/close.
- `await _pool.ping()` 으로 시작 시점에 연결 검증 — 실패 시 startup 자체를 막을 수 있음.
- `aclose()` 가 신권장 (구 `close()` 는 deprecated).

### 3.3 스케줄러 (cron-like 백그라운드 루프)

```python
# scheduler.py
async def start_scheduler(send_command):
    global _task
    _task = asyncio.create_task(_scheduler_loop(send_command))

async def _scheduler_loop(send_command):
    while not _stop_event.is_set():
        now = datetime.now()
        for entry in _schedule_table:
            if entry.matches(now):
                asyncio.create_task(send_command(entry.action, entry.country))
        await asyncio.sleep(60 - now.second)  # 매 분 0초에 깨도록 정렬

async def stop_scheduler():
    _stop_event.set()
    if _task:
        await _task
```

**핵심**:
- `send_command` 를 인자로 주입 → 스케줄러가 WS 모듈에 직접 의존하지 않음 (DI 패턴).
- "매 분 0초" 같은 alignment 는 `60 - now.second` 로.

---

## 4. lifespan 에서 자주 하는 실수

### 4.1 startup 안에서 무한 루프 직접 실행
```python
# 잘못된 예
@asynccontextmanager
async def lifespan(app):
    while True:  # ← 여기서 멈춤. yield 도달 못 함.
        await collect_once()
        await asyncio.sleep(10)
    yield
```
→ 앱이 트래픽을 받지 못한다. 반드시 `asyncio.create_task` 로 분리해야 함.

### 4.2 동기 블로킹 호출
```python
@asynccontextmanager
async def lifespan(app):
    requests.get("...")  # ← 동기. 이벤트 루프 블로킹.
    yield
```
→ `httpx.AsyncClient` 사용하거나 `asyncio.to_thread(...)` 로 감싸야 함.

### 4.3 종료 시 task cancel 후 await 안 함
```python
async def stop_collector():
    _task.cancel()
    # ← await 안 하면 정리 코드가 실행될 시간이 없음
```
→ `try: await _task except asyncio.CancelledError: pass` 패턴.

### 4.4 lifespan 중 발생한 예외를 무시
- startup 실패가 silent 하면 앱은 떠 있지만 핵심 기능이 죽어 있는 상태가 됨.
- 의도적으로 무시해야 한다면 로깅이라도 명시적으로.

---

## 5. lifespan 과 의존성 주입 결합

`yield` 에 값을 반환하면 `request.state` 같은 곳을 통해 핸들러에서 접근 가능.

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    pool = await asyncpg.create_pool(...)
    app.state.db = pool
    yield
    await pool.close()

@app.get("/users/{uid}")
async def get_user(uid: int, request: Request):
    async with request.app.state.db.acquire() as conn:
        row = await conn.fetchrow("SELECT * FROM users WHERE id=$1", uid)
        return dict(row)
```

**왜 모듈 전역 대신 `app.state`?**
- 테스트에서 다른 풀(테스트 DB)을 주입하기 쉬움.
- 한 프로세스에 여러 FastAPI 인스턴스가 떠도 각자 격리됨.

---

## 6. 응용 포인트

- 외부 의존성(Redis/DB/메시지큐 풀)이 있는 모든 FastAPI 앱은 lifespan 사용.
- 컴포넌트 시작 함수는 `start_xxx()`, 종료는 `stop_xxx()` / `close_xxx()` 로 통일 → lifespan이 단순해짐.
- 실패 격리가 필요하면 `try/finally` + `started` 스택 + 역순 정리 패턴.
- 백그라운드 루프는 `asyncio.create_task` + `asyncio.Event` 조합. `time.sleep` 절대 금지.
