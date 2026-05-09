# 17 — Python 데코레이터 (함수 / 클래스의 메타프로그래밍)

> ShopTracker 코드의 거의 모든 라우트가 `@router.post(...)`, 모든 도메인 객체가 `@dataclass`, DI Provider 가 `@provide(scope=...)`. 데코레이터는 *함수를 받아 함수를 반환하는 함수* 일 뿐이지만, 이 단순한 메커니즘이 Python 생태계의 절반을 떠받친다.

---

## §0 본질

```python
def deco(func):
    def wrapper(*args, **kwargs):
        print("before")
        result = func(*args, **kwargs)
        print("after")
        return result
    return wrapper

@deco
def hello():
    print("hello")

# 위는 아래와 같음
def hello():
    print("hello")
hello = deco(hello)
```

`@deco` 는 *문법 설탕* — `hello = deco(hello)`.

---

## §1 다양한 형태

### 1.1 인자 없는 데코레이터

```python
def log(func):
    def wrapper(*args, **kwargs):
        print(f"call {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@log
def f(): ...
```

### 1.2 인자 받는 데코레이터 (3 단계)

```python
def repeat(n: int):
    def deco(func):
        def wrapper(*args, **kwargs):
            for _ in range(n):
                func(*args, **kwargs)
        return wrapper
    return deco

@repeat(3)
def hello(): print("hi")
```

`@repeat(3)` → `repeat(3)` 호출 → `deco` 반환 → `hello = deco(hello)`.

### 1.3 functools.wraps

```python
from functools import wraps

def log(func):
    @wraps(func)         # ← 원본 함수의 __name__ / __doc__ 보존
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

`wraps` 안 쓰면 `f.__name__ == "wrapper"` 가 되어 디버깅 / introspection 깨짐. **항상 쓰기**.

### 1.4 클래스 기반

```python
class CountCalls:
    def __init__(self, func):
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        return self.func(*args, **kwargs)

@CountCalls
def f(): ...

f(); f(); f()
print(f.count)   # 3
```

### 1.5 클래스 데코레이터

함수가 아닌 *클래스* 를 변형.

```python
def add_repr(cls):
    def __repr__(self):
        return f"{cls.__name__}({self.__dict__})"
    cls.__repr__ = __repr__
    return cls

@add_repr
class Foo:
    def __init__(self, x): self.x = x
```

`@dataclass` 가 정확히 이 패턴.

---

## §2 ShopTracker 의 데코레이터 분석

### 2.1 @dataclass

```python
@dataclass(frozen=True)
class OrderCreatedEvent:
    order_id: UUID
    ...
```

- `dataclass` 가 클래스 데코레이터.
- `frozen=True` 인자 받음 — 3 단계 형태.
- 클래스에 `__init__`, `__eq__`, `__hash__` 메서드 주입.

### 2.2 FastAPI route 데코레이터

```python
@router.post("/", status_code=201)
async def create_order(...):
    ...
```

- `router.post(...)` → 데코레이터 반환 → `create_order` 등록.
- 함수 자체는 그대로 호출 가능. 단, FastAPI 가 라우팅 테이블에 등록.

### 2.3 Dishka @provide

```python
class OrdersProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def order_repo(self, session: AsyncSession) -> OrderRepositoryProtocol:
        return SQLAlchemyOrderRepository(session)
```

- `provide(scope=...)` → 데코레이터.
- 메서드를 *DI provider* 로 등록 (런타임에 introspection).

### 2.4 @classmethod / @staticmethod / @property

내장 데코레이터.

```python
class Order:
    @classmethod
    def create(cls, ...) -> "Order": ...

    @property
    def total(self) -> Money: ...
```

`classmethod` 는 첫 인자가 `cls`, `property` 는 메서드를 속성처럼.

### 2.5 @abstractmethod

```python
from abc import ABC, abstractmethod

class Repo(ABC):
    @abstractmethod
    async def save(self, ...): ...
```

서브클래스가 구현 안 하면 인스턴스화 시 TypeError.

ShopTracker 는 ABC 안 쓰고 Protocol 채택 (02 장).

### 2.6 pytest fixture

```python
@pytest.fixture
async def session():
    ...

@pytest.mark.asyncio
async def test_foo(session): ...
```

데코레이터로 fixture / mark 등록.

---

## §3 표준 라이브러리 유용 데코레이터

### 3.1 functools

```python
@functools.lru_cache(maxsize=128)
def fib(n):
    if n < 2: return n
    return fib(n-1) + fib(n-2)
```

자동 메모이제이션.

### 3.2 functools.cached_property (3.8+)

```python
class Foo:
    @cached_property
    def expensive(self):
        return heavy_compute()
```

첫 접근 시 계산, 이후 캐시.

### 3.3 contextlib.contextmanager

```python
from contextlib import contextmanager

@contextmanager
def temp_file():
    f = open("/tmp/x", "w")
    try:
        yield f
    finally:
        f.close()

with temp_file() as f:
    f.write("hi")
```

(§ 18 에서 자세히)

---

## §4 함정

### 4.1 @wraps 잊기

```python
def log(func):
    def wrapper(*args, **kwargs): ...
    return wrapper      # ← @wraps 없음

@log
def f(): pass

f.__name__              # "wrapper" — 디버깅 어려움
```

### 4.2 데코레이터가 함수 시그니처 변경

```python
def add_arg(func):
    def wrapper(extra, *args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@add_arg
def f(x): return x

f("extra", 1)    # 호출자 시그니처가 바뀜 — 혼란
```

→ ParamSpec (3.10+) 으로 시그니처 보존 가능.

### 4.3 클래스 메서드에 데코레이터

```python
class Foo:
    @log              # OK
    def method(self): ...
```

`self` 는 wrapper 가 자동 처리 (`*args` 에 포함).

### 4.4 데코레이터 적용 순서

```python
@a
@b
@c
def f(): ...

# 위는 f = a(b(c(f)))
# 즉 c 가 가장 먼저 적용.
```

순서 중요 — `@property` 는 항상 가장 위, `@classmethod` 는 두 번째 등 관용.

### 4.5 mutable closure

```python
def deco(func):
    calls = []
    def wrapper(*args, **kwargs):
        calls.append(args)
        return func(*args, **kwargs)
    wrapper.calls = calls   # 외부 노출
    return wrapper
```

closure 의 mutable 객체 공유 — 의도적이면 OK, 실수면 버그.

### 4.6 인자 받는 vs 안 받는 헷갈림

```python
@deco       # deco(f)
def f(): ...

@deco()     # deco()(f)
def f(): ...
```

`@deco` 와 `@deco()` 는 다름. 라이브러리 설계 시 둘 다 지원하려면 분기 처리.

### 4.7 stacking 시 시그니처 보존

여러 데코레이터 쌓으면 IDE 의 자동완성 / mypy 가 혼란. ParamSpec + Protocol 활용.

---

## §5 메타프로그래밍의 큰 그림

데코레이터는 *함수/클래스 변형* 의 한 도구. 다른 도구들 :

| 도구 | 용도 |
|---|---|
| 데코레이터 | 함수/클래스 *후처리* |
| `__init_subclass__` | 서브클래스 생성 시 hook |
| `__set_name__` | descriptor 가 자기 이름 알기 |
| `metaclass` | 클래스의 클래스 (가장 강력, 가장 복잡) |
| descriptor (`__get__`, `__set__`) | 속성 접근 가로채기 |

데코레이터는 진입 비용이 가장 낮음. ShopTracker 는 데코레이터 + 표준 dataclass 만 — 메타클래스 안 씀.

---

## §6 직접 만든 유용 데코레이터 예

### 6.1 retry

```python
def retry(times=3, delay=1):
    def deco(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            for attempt in range(times):
                try:
                    return await func(*args, **kwargs)
                except Exception:
                    if attempt == times - 1: raise
                    await asyncio.sleep(delay * 2 ** attempt)
        return wrapper
    return deco

@retry(times=3, delay=1)
async def call_pg(...): ...
```

### 6.2 timing

```python
def timing(func):
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start = time.monotonic()
        try:
            return await func(*args, **kwargs)
        finally:
            elapsed = time.monotonic() - start
            logger.info("elapsed", func=func.__name__, ms=elapsed*1000)
    return wrapper
```

### 6.3 require_role

```python
def require_role(role):
    def deco(func):
        @wraps(func)
        async def wrapper(*args, current_user, **kwargs):
            if role not in current_user.roles:
                raise PermissionError()
            return await func(*args, current_user=current_user, **kwargs)
        return wrapper
    return deco

@require_role("admin")
async def delete_order(order_id, current_user): ...
```

→ FastAPI 에서는 보통 Depends 로 처리. 데코레이터는 더 일반적.

---

## §10 학습 포인트 (한 줄 요약)

1. **데코레이터 = 함수를 받아 함수를 반환하는 함수**.
2. `@deco` 는 `f = deco(f)` 의 설탕.
3. **인자 받는 데코레이터** = 3 단계 (deco factory).
4. **@functools.wraps** 항상 — 메타데이터 보존.
5. **클래스 데코레이터** = `@dataclass` 가 그 예.
6. **순서** : 아래에서 위로 적용.
7. ShopTracker : `@dataclass`, `@router.post`, `@provide`, `@classmethod`, `@property`.
8. **closure 활용** — counter / cache / lock 같은 상태.
9. **ParamSpec (3.10+)** — 시그니처 보존하는 정밀 데코레이터.
10. **메타클래스보다 데코레이터** — 같은 효과를 더 단순하게.

---

## 추가 참고

- functools docs : https://docs.python.org/3/library/functools.html
- "Python Cookbook" 9 장 (Metaprogramming)
- ShopTracker 다음 글 : `18-python-context-managers.md`
