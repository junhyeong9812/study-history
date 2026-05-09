# 18 — Python Context Managers (`with` / `async with`)

> ShopTracker 의 DI session generator 가 `async with AsyncSession(engine) as s:` 로 시작하는 데에는 이유가 있다. 자원 (DB 연결 / 파일 / lock) 을 얻고 *반드시 정리* 해야 할 때 context manager 가 안전망. 이 글은 `with` 의 작동 원리와 직접 만드는 법.

---

## §0 동기 — 자원 누수 방지

```python
# anti-pattern
f = open("data.txt")
data = f.read()
f.close()                    # ← 중간에 예외 나면 close 안 됨!

# try/finally 로 안전화
f = open("data.txt")
try:
    data = f.read()
finally:
    f.close()

# context manager
with open("data.txt") as f:  # ← 같은 효과, 더 간결
    data = f.read()
# 자동으로 close
```

`with` 의 본질 = `try/finally` 의 캡슐화.

---

## §1 본질 — `__enter__` / `__exit__`

```python
class MyContext:
    def __enter__(self):
        print("enter")
        return "value"     # ← `as x` 의 x

    def __exit__(self, exc_type, exc, tb):
        print("exit")
        # return True 면 예외 swallow
        return False

with MyContext() as v:
    print(v)              # "value"
    # 예외든 정상이든 __exit__ 호출
```

- `__enter__` : 자원 획득.
- `__exit__(exc_type, exc, tb)` : 자원 해제. exc_type 이 None 이면 정상 종료.
- 반환값 True → 예외 무시. False → 예외 전파.

---

## §2 contextlib.contextmanager — 함수로 만들기

```python
from contextlib import contextmanager

@contextmanager
def my_ctx():
    print("enter")
    try:
        yield "value"
    finally:
        print("exit")

with my_ctx() as v:
    print(v)
```

- `yield` 가 분기점. 위 = `__enter__`, 아래 = `__exit__`.
- 단 한 번만 yield — generator 가 종료되면 context 도 종료.
- 더 짧고 직관적.

---

## §3 async with — 비동기 context manager

```python
class AsyncContext:
    async def __aenter__(self):
        await acquire()
        return self

    async def __aexit__(self, exc_type, exc, tb):
        await release()
```

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def async_ctx():
    await acquire()
    try:
        yield ...
    finally:
        await release()
```

ShopTracker 의 session generator :

```python
async with AsyncSession(engine) as s:
    yield s
    await s.commit()
```

- `AsyncSession` 가 async context manager — `__aenter__` / `__aexit__` 구현.
- 끝나면 자동으로 close.

---

## §4 ShopTracker 코드 분석

### 4.1 Session lifecycle

```python
@provide(scope=Scope.REQUEST, stream=True)
async def session(self, engine: AsyncEngine) -> AsyncIterator[AsyncSession]:
    async with AsyncSession(engine) as s:
        try:
            yield s
            await s.commit()
        except Exception:
            await s.rollback()
            raise
```

- `async with AsyncSession(engine) as s` — 세션 획득. exit 시 자동 close.
- `yield s` — Dishka 가 핸들러에 주입.
- `try/except` — 핸들러 끝난 후 commit 또는 rollback.

### 4.2 SQLAlchemy 트랜잭션

```python
async with engine.begin() as conn:
    await conn.run_sync(Base.metadata.create_all)
```

`engine.begin()` 자체가 async context manager — 끝나면 commit, 예외 시 rollback.

### 4.3 FastAPI lifespan

```python
@asynccontextmanager
async def lifespan(app):
    container = make_async_container(...)
    app.state.container = container
    yield
    await container.close()

app = FastAPI(lifespan=lifespan)
```

앱 시작 ~ 종료의 자원 관리.

### 4.4 pytest fixture (yield style)

```python
@pytest.fixture
async def session():
    engine = create_async_engine(...)
    async with AsyncSession(engine) as s:
        yield s
    await engine.dispose()
```

fixture 도 generator — `yield` 까지 setup, 이후 teardown.

---

## §5 표준 라이브러리

### 5.1 contextlib.suppress

```python
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove("maybe-not-there.txt")
```

특정 예외를 무시. try/except pass 의 단축.

### 5.2 contextlib.closing

```python
from contextlib import closing

with closing(some_resource()) as r:
    r.use()
# r.close() 자동
```

`__enter__/__exit__` 없는 객체에 close 만 보장.

### 5.3 contextlib.ExitStack

여러 context 를 동적으로 stacking.

```python
from contextlib import ExitStack

with ExitStack() as stack:
    files = [stack.enter_context(open(f)) for f in filenames]
    # 모든 파일이 자동 close
```

### 5.4 contextlib.AsyncExitStack

async 버전.

---

## §6 패턴 예

### 6.1 Lock

```python
import asyncio

lock = asyncio.Lock()

async def f():
    async with lock:
        await critical_section()
```

### 6.2 Timing

```python
@contextmanager
def timer(name):
    start = time.monotonic()
    try:
        yield
    finally:
        print(f"{name}: {time.monotonic() - start:.3f}s")

with timer("import"):
    import heavy_module
```

### 6.3 Temp working directory

```python
@contextmanager
def cd(path):
    old = os.getcwd()
    os.chdir(path)
    try:
        yield
    finally:
        os.chdir(old)

with cd("/tmp"):
    do_stuff()
```

### 6.4 Mocking

```python
from unittest.mock import patch

with patch("module.heavy") as m:
    m.return_value = "fake"
    test()
```

---

## §7 함정

### 7.1 yield 두 번

```python
@contextmanager
def bad():
    yield 1
    yield 2     # RuntimeError

with bad(): ...
```

→ 정확히 한 번.

### 7.2 yield 안 함

```python
@contextmanager
def bad():
    print("forgot yield")
```

→ RuntimeError on enter.

### 7.3 generator 내부의 finally 가 cleanup 보장 X (옛 Python)

3.6+ 는 OK. 단, *unstarted generator* 가 GC 되면 finally 안 돔. 명시적 close.

### 7.4 async vs sync 혼용

```python
with async_ctx_manager():        # ← TypeError
    ...
async with sync_ctx_manager():   # ← TypeError
    ...
```

`__enter__` 와 `__aenter__` 는 다른 protocol. 매칭되는 with 사용.

### 7.5 multiple values

```python
with open("a") as a, open("b") as b:
    ...

# 또는
with open("a") as a:
    with open("b") as b:
        ...
```

Python 3.10+ 는 multiline with 도 지원 :

```python
with (
    open("a") as a,
    open("b") as b,
    open("c") as c,
):
    ...
```

### 7.6 예외 swallow 의도치 않게

```python
class Bad:
    def __exit__(self, *a):
        return True       # ← 모든 예외 무시. 위험.
```

`__exit__` 의 반환값에 주의. 보통 None / False (전파).

---

## §8 다른 언어와 비교

| 언어 | 자원 관리 |
|---|---|
| **Java** | try-with-resources (`AutoCloseable`), 7+ |
| **C#** | `using` 블록 / `IDisposable` |
| **Rust** | `Drop` trait — RAII, 자동 |
| **Go** | `defer` — 함수 끝에 실행 보장 |
| **JS** | 명시적 try/finally, `using` (TC39 stage 3) |

Python 의 `with` 는 Java 의 try-with-resources, C# 의 using 과 사상 동일.

---

## §10 학습 포인트 (한 줄 요약)

1. **`with` = `try/finally` 의 캡슐화** — 자원 정리 보장.
2. **`__enter__` / `__exit__`** — 클래스 기반 protocol.
3. **`@contextmanager`** — 함수 + yield 로 간결하게.
4. **async** : `async with`, `__aenter__/__aexit__`, `@asynccontextmanager`.
5. **ShopTracker session generator** — async + try/commit/rollback.
6. **FastAPI lifespan** = `@asynccontextmanager` — 앱 시작/종료.
7. **pytest fixture (yield)** = context manager 같은 setup/teardown.
8. **ExitStack** — 여러 컨텍스트 동적 조립.
9. **`__exit__` 반환 True 면 예외 swallow** — 위험, 신중히.
10. **자원이 있는 곳엔 항상 with** — 누수의 가장 흔한 원인 차단.

---

## 추가 참고

- contextlib docs : https://docs.python.org/3/library/contextlib.html
- PEP 343 (with statement) : https://peps.python.org/pep-0343/
- ShopTracker 다음 글 : `19-python-imports.md`
