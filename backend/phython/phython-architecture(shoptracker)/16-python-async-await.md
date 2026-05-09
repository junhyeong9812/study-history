# 16 — Python async / await (이벤트 루프와 코루틴)

> ShopTracker 의 거의 모든 함수가 `async def`. SQLAlchemy 도 async, FastAPI 도 async, EventBus 도 async. 왜 sync 가 아니라 async 인가, 그리고 async 코드의 함정은 무엇인가.

---

## §0 동기 — I/O 기다리는 시간이 아깝다

```python
# sync
def fetch_3_users():
    a = http_get("/users/1")     # 100ms wait (CPU idle)
    b = http_get("/users/2")     # 100ms wait
    c = http_get("/users/3")     # 100ms wait
    # total 300ms
```

```python
# async
async def fetch_3_users():
    a, b, c = await asyncio.gather(
        http_get_async("/users/1"),   # 100ms 동안 다른 일
        http_get_async("/users/2"),
        http_get_async("/users/3"),
    )
    # total ~100ms
```

웹 서버는 I/O 가 90% 이상. async 로 동시성 ↑.

---

## §1 본질 — 이벤트 루프 + 코루틴

### 1.1 모델

```
┌──── Event Loop ────┐
│                    │
│  Task A (waiting)  │ ← I/O 대기 중. CPU 안 씀
│  Task B (running)  │ ← 지금 실행 중
│  Task C (ready)    │ ← I/O 끝. 곧 실행
│                    │
└────────────────────┘
```

- **싱글 스레드** — 동시에 *진짜* 두 일을 안 함.
- 한 Task 가 `await` 하면 → 이벤트 루프가 다른 Task 로 전환.
- I/O 대기 중에는 CPU 가 다른 Task 를 처리.

### 1.2 코루틴 (coroutine)

`async def` 가 만드는 것. 호출 시 *즉시 실행 안 함* — coroutine object 반환.

```python
async def f():
    print("start")
    await asyncio.sleep(1)
    print("end")

c = f()              # coroutine 객체. print 안 됨.
asyncio.run(c)       # 이벤트 루프 시작. 그제야 실행.
```

`await` 가 *suspension point* — 여기서 이벤트 루프가 다른 일 함.

---

## §2 ShopTracker 코드 정독

### 2.1 모든 핸들러가 async

```python
class CreateOrderHandler:
    async def handle(self, command: CreateOrderCommand) -> UUID:
        ...
        await self._repo.save(order)
        await self._event_bus.publish(...)
```

- DB 호출 (`save`) 이 async — `await` 동안 이벤트 루프가 다른 요청 처리.
- 이벤트 발행 (`publish`) 도 async.

### 2.2 SQLAlchemy Async

```python
async def save(self, order: Order) -> None:
    model = order_to_model(order)
    self._session.add(model)              # sync (큐잉만)
    await self._session.flush()           # async (실제 SQL)
```

- `session.add` 는 큐잉이라 sync.
- `flush` 가 진짜 DB 통신 → async.

### 2.3 FastAPI router

```python
@router.post("/")
async def create_order(body, handler: FromDishka[...]):
    ...
    await handler.handle(command)
```

FastAPI 가 async 함수면 그대로 이벤트 루프에서 실행. sync 함수면 별도 thread pool 로 보냄.

### 2.4 EventBus

```python
async def publish(self, event):
    for handler in self._handlers[type(event)]:
        try:
            await handler(event)
```

- 핸들러를 *직렬* 로 await. 첫 핸들러 끝나야 다음 시작.
- 병렬 원하면 `asyncio.gather`.

### 2.5 Dishka session generator

```python
@provide(scope=Scope.REQUEST, stream=True)
async def session(self, engine) -> AsyncIterator[AsyncSession]:
    async with AsyncSession(engine) as s:
        try:
            yield s
            await s.commit()
        except Exception:
            await s.rollback()
            raise
```

- `async with` — async context manager (§ 18 참조).
- `yield` — async generator. Dishka 가 `yield` 까지 실행 → 핸들러에 주입 → 끝나면 commit/rollback.

---

## §3 async 의 핵심 도구

### 3.1 asyncio.run

진입점. *한 번만* 호출.

```python
async def main():
    ...
asyncio.run(main())
```

### 3.2 await

코루틴을 실행하고 결과를 기다림.

```python
result = await some_async_func()
```

### 3.3 asyncio.gather — 병렬

```python
a, b, c = await asyncio.gather(f(), g(), h())
# 셋 다 동시 실행, 모두 끝나면 반환
```

옵션 :
- `return_exceptions=True` — 예외도 결과로 받음 (안 raise).

### 3.4 asyncio.create_task — fire-and-forget

```python
task = asyncio.create_task(f())
# f() 가 백그라운드에서 시작
...
result = await task    # 필요할 때 결과 받기
```

주의 : task 참조를 어딘가 보관 안 하면 GC 가 잡아갈 수 있음.

### 3.5 asyncio.wait / wait_for

```python
result = await asyncio.wait_for(f(), timeout=5)   # 5 초 안에 못 끝나면 TimeoutError
```

### 3.6 asyncio.Queue / Lock / Event

스레드의 queue.Queue / threading.Lock 의 async 버전.

### 3.7 async generator / async for

```python
async def stream():
    for i in range(10):
        await asyncio.sleep(0.1)
        yield i

async for x in stream():
    print(x)
```

### 3.8 async with

```python
async with AsyncSession(engine) as s:
    ...
```

`__aenter__` / `__aexit__` 가 async (§18).

---

## §4 함정

### 4.1 sync 함수 안에서 await

```python
def f():
    await some_async()    # SyntaxError - async 함수 안에서만 await
```

### 4.2 async 함수를 sync 처럼 호출

```python
async def f(): return 1
x = f()       # ← coroutine 객체. 1 아님!
```

→ `await f()` 또는 `asyncio.run(f())`.

### 4.3 blocking I/O 가 async 함수 안에 섞임

```python
async def handler():
    data = requests.get(url)        # ← sync HTTP. 이벤트 루프 멈춤!
```

→ `httpx.AsyncClient`, `aiohttp` 같은 async 라이브러리. 정 sync 라이브러리 써야 하면 `asyncio.to_thread(...)`.

```python
async def handler():
    data = await asyncio.to_thread(requests.get, url)
```

### 4.4 await 잊기

```python
async def f():
    save()                 # ← async 인데 await 안 함. coroutine 그냥 버려짐.
                           # RuntimeWarning: coroutine 'save' was never awaited
```

→ mypy / pyright 가 잡아줌 (return 값 무시 경고).

### 4.5 이벤트 루프가 두 개

```python
asyncio.run(f())
asyncio.run(g())   # 새 루프. 같은 task / connection 공유 X
```

→ 한 루프에서 다 처리. `asyncio.run` 은 진입점에 한 번.

### 4.6 task 참조 누락

```python
async def main():
    asyncio.create_task(f())    # ← 변수에 안 받음
    await asyncio.sleep(10)
```

GC 가 task 를 거둘 수 있음 → "Task was destroyed but it is pending!" 경고.

→ 변수에 받거나 `asyncio.gather` 또는 `TaskGroup` (3.11+).

### 4.7 TaskGroup (3.11+)

```python
async with asyncio.TaskGroup() as tg:
    tg.create_task(f())
    tg.create_task(g())
# 둘 다 끝날 때까지 대기. 하나 실패면 다른 것도 cancel + raise.
```

`gather` 보다 안전. ShopTracker 가 채택할 만한 패턴.

### 4.8 thread-safety vs async-safety

async = 단일 스레드 → race condition 적음. 단 `await` 사이에 다른 task 가 끼어들 수 있음 → 임계영역 보호는 `asyncio.Lock`.

### 4.9 CPU-bound 작업

async 는 I/O 에 좋고 CPU 에는 무력. 무거운 계산은 process pool :

```python
loop = asyncio.get_event_loop()
result = await loop.run_in_executor(ProcessPoolExecutor(), heavy_compute, data)
```

---

## §5 async 와 동기의 비교

| | sync | async |
|---|---|---|
| 진정한 병렬 | 멀티스레드 / 멀티프로세스 | X (싱글 스레드) |
| 동시성 | thread per request | task per request |
| 메모리 / 요청 | thread (수 MB) | task (수 KB) |
| 스케일 | 수백 thread | 수만 task |
| CPU bound | thread/process | process pool |
| I/O bound | thread (느림) | task (빠름) |

웹 서버는 I/O 위주 → async 의 압승.

---

## §6 다른 언어

| 언어 | 모델 |
|---|---|
| **JS** | 단일 스레드 + 이벤트 루프 (Node.js). Python 의 모태. |
| **Go** | goroutine — 사용자 입장에선 sync 처럼. runtime 이 스케줄. |
| **Rust** | async/await + tokio. 컴파일 타임 안전성. |
| **Kotlin** | coroutine — Python 과 유사. structured concurrency. |
| **Java** | Project Loom (가상 스레드, 21+) — sync 처럼 짜도 동시성. |

Python 의 async/await 는 JS 와 가장 유사. Go 의 goroutine 은 다른 패러다임 (사용자가 await 안 적음).

---

## §7 실전 패턴

### 7.1 동시 호출

```python
async def get_dashboard(user_id):
    user, orders, notifications = await asyncio.gather(
        get_user(user_id),
        list_orders(user_id),
        list_notifications(user_id),
    )
    return {...}
```

### 7.2 timeout

```python
try:
    result = await asyncio.wait_for(slow_call(), timeout=5)
except asyncio.TimeoutError:
    result = None
```

### 7.3 retry

```python
for attempt in range(3):
    try:
        return await call()
    except Exception:
        await asyncio.sleep(2 ** attempt)
raise
```

### 7.4 producer / consumer

```python
queue = asyncio.Queue()

async def producer():
    for i in range(10):
        await queue.put(i)

async def consumer():
    while True:
        item = await queue.get()
        await process(item)
        queue.task_done()
```

---

## §10 학습 포인트 (한 줄 요약)

1. **async = 이벤트 루프 + 코루틴** — I/O 대기 중 다른 일.
2. **싱글 스레드** — race 적지만 await 사이는 끼어들 수 있음.
3. **`asyncio.run` 한 번** — 진입점에서.
4. **`await` 잊지 말 것** — coroutine 가 그냥 버려짐.
5. **blocking I/O 금지** — `asyncio.to_thread` 또는 async 라이브러리.
6. **`asyncio.gather`** = 병렬, **`TaskGroup` (3.11+)** = 더 안전.
7. **task 참조 보관** — GC 회피.
8. **CPU bound 는 process pool** — async 무력.
9. **`async with` / `async for`** — context manager / iterator 의 async 버전.
10. **웹 서버는 I/O 위주** → async 의 자연스러운 영역.

---

## 추가 참고

- asyncio 공식 docs : https://docs.python.org/3/library/asyncio.html
- "async/await in Python: a primer" — David Beazley 강연 추천
- ShopTracker 다음 글 : `17-python-decorators.md`
