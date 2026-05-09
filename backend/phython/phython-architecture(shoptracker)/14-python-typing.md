# 14 — Python Typing (타입 힌트의 모든 것)

> Python 3.5 의 PEP 484 가 시작이고, 3.13 까지 매년 진화했다. ShopTracker 코드의 모든 함수 시그니처에 타입 힌트가 박혀 있다 — `list[OrderItem]`, `Decimal | None`, `Protocol`, `Mapped[str]`. 이 글은 *왜* 타입 힌트를 쓰고 *각 표기가 정확히 무엇을 의미하는지* 정리.

---

## §0 동기 — Python 의 타입 힌트는 *런타임 타입 검사가 아니다*

```python
def add(a: int, b: int) -> int:
    return a + b

add("1", "2")    # 그냥 작동함. "12" 반환.
```

타입 힌트는 :
- **런타임에는 무시됨** (대부분).
- mypy / pyright / pylance 같은 *정적 검사기* 가 컴파일 시 잡음.
- IDE 의 자동완성 / refactor 향상.
- 코드의 *의도 문서화*.

→ "타입 힌트는 *문서이자 도구* 이지 강제 검사 X".

---

## §1 기본 타입

### 1.1 Primitive

```python
x: int = 1
y: float = 1.0
z: str = "hello"
b: bool = True
n: bytes = b"\x00"
```

### 1.2 컨테이너 (3.9+)

```python
xs: list[int] = [1, 2, 3]
d: dict[str, int] = {"a": 1}
t: tuple[int, str] = (1, "a")           # 고정 길이
ts: tuple[int, ...] = (1, 2, 3)         # 가변 길이
s: set[int] = {1, 2}
fs: frozenset[int] = frozenset({1, 2})
```

3.9 이전엔 `from typing import List` 가 필요했음 → 3.9+ 에서 lowercase generic.

### 1.3 Optional / Union (3.10+)

```python
x: int | None = None        # 3.10+
y: int | str = 1            # 둘 중 하나

# 3.10 이전
from typing import Optional, Union
x: Optional[int] = None
y: Union[int, str] = 1
```

ShopTracker 가 `int | None` 을 쓰는 이유 : Python 3.10+ 의존, 명시적 + 짧음.

### 1.4 Any / object

```python
x: Any = ...     # 어떤 타입이든. 검사 비활성화.
y: object = ...  # 모든 객체의 root. 메서드 호출 시 검사 통과 X (object 메서드만).
```

`Any` 는 *탈출구* — mypy 가 통과시킴. 남용 금지. ShopTracker 의 EventBus 의 `event: object` 는 의도적 — *모든 dataclass 받기 위해*.

---

## §2 함수 시그니처

### 2.1 일반

```python
def f(x: int, y: int = 10) -> int:
    return x + y
```

### 2.2 가변 인자

```python
def f(*args: int, **kwargs: str) -> None: ...
```

### 2.3 Callable

```python
from collections.abc import Callable

handler: Callable[[int, str], bool] = lambda x, s: bool(x)
# Callable[[arg1, arg2, ...], return]

# 가변 인자
handler: Callable[..., bool] = ...    # 인자 신경 안 씀
```

ShopTracker 의 EventBus :

```python
def subscribe(self, event_type: type, handler: Callable) -> None: ...
```

`Callable` (인자 없음) — 어떤 callable 이든 OK. 더 엄격하게는 `Callable[[Any], Awaitable[None]]`.

### 2.4 async / Awaitable

```python
async def f(x: int) -> str:
    return str(x)

# 시그니처상 반환은 Coroutine[Any, Any, str], 보통 그냥 -> str 로 적음.
```

---

## §3 클래스 / 제네릭

### 3.1 dataclass

```python
@dataclass
class Order:
    id: UUID
    customer_name: str
    items: list[OrderItem]      # ← 필드 타입
```

### 3.2 Generic class

```python
from typing import Generic, TypeVar

T = TypeVar("T")

class Box(Generic[T]):
    def __init__(self, value: T): self.value = value
    def get(self) -> T: return self.value

b: Box[int] = Box(1)
```

3.12+ 의 새 문법 :

```python
class Box[T]:                   # PEP 695, Python 3.12+
    def __init__(self, value: T): self.value = value
```

### 3.3 TypeVar 의 bound

```python
T = TypeVar("T", bound=Comparable)   # T 는 Comparable 의 하위 타입

def max_of(xs: list[T]) -> T: ...
```

### 3.4 Protocol

(02 장 참조)

```python
class Drawable(Protocol):
    def draw(self) -> None: ...

def render(d: Drawable): d.draw()    # 어떤 클래스든 draw() 있으면 OK
```

ShopTracker 의 모든 Repository / EventBus 가 Protocol 패턴.

---

## §4 Literal / Final / TypeGuard

### 4.1 Literal

```python
from typing import Literal

def set_mode(mode: Literal["dev", "prod", "test"]) -> None: ...

set_mode("dev")          # OK
set_mode("foo")          # mypy error
```

ShopTracker 에서 `currency: Literal["KRW", "USD"]` 같은 식으로 더 엄격하게 가능.

### 4.2 Final

```python
from typing import Final

MAX_RETRIES: Final = 3
MAX_RETRIES = 5          # mypy error
```

### 4.3 TypeGuard (3.10+)

```python
from typing import TypeGuard

def is_str_list(xs: list[object]) -> TypeGuard[list[str]]:
    return all(isinstance(x, str) for x in xs)

xs: list[object] = ["a", "b"]
if is_str_list(xs):
    xs[0].upper()        # mypy 가 xs 를 list[str] 로 좁힘
```

---

## §5 ShopTracker 코드의 타입 힌트 정독

### 5.1 Protocol + 비동기

```python
class OrderRepositoryProtocol(Protocol):
    async def save(self, order: Order) -> None: ...
    async def find_by_id(self, order_id: UUID) -> Order | None: ...
```

- `Protocol` — structural typing.
- `async def ... -> None` — coroutine 반환.
- `Order | None` — 못 찾으면 None.

### 5.2 dataclass + Generic 없이

```python
@dataclass(frozen=True)
class CreateOrderCommand:
    customer_name: str
    items: list[OrderItemDTO]
```

- 모든 필드 타입 명시. `mypy strict` 통과.

### 5.3 SQLAlchemy 의 Mapped[T]

```python
class OrderModel(Base):
    id: Mapped[str] = mapped_column(String(36), primary_key=True)
```

- `Mapped[T]` — SQLAlchemy 2.0 의 typed declarative. `id` 가 Python 코드에서는 `str` 로 보이고, ORM 메타데이터는 `mapped_column(...)` 가 갖고 있음.

### 5.4 Callable 의 일반형 vs 구체형

```python
# event_bus.py
self._handlers: dict[type, list[Callable]] = defaultdict(list)
```

`Callable` 만 — 어떤 callable 이든. 더 엄격하게 `Callable[[object], Awaitable[None]]` 가능.

### 5.5 Self 타입 (3.11+)

```python
from typing import Self

class Order:
    @classmethod
    def create(cls, ...) -> Self:    # 3.11+
        return cls(...)
```

3.11 이전엔 `-> "Order"` 또는 `TypeVar` 트릭.

---

## §6 mypy / pyright 사용

### 6.1 설치

```bash
pip install mypy
mypy src/
```

### 6.2 설정 (`pyproject.toml`)

```toml
[tool.mypy]
python_version = "3.12"
strict = true
disallow_untyped_defs = true
warn_unused_ignores = true
warn_redundant_casts = true

[[tool.mypy.overrides]]
module = ["dishka.*"]
ignore_missing_imports = true
```

`strict = true` 추천 — *모든 함수에 타입 힌트 강제*, 누락 시 error.

### 6.3 자주 쓰는 escape hatch

```python
# type: ignore                     # 한 줄 무시
# type: ignore[arg-type]           # 특정 에러만
x: int = something()  # type: ignore

from typing import cast
y = cast(int, raw)                 # 강제 캐스트 (mypy 만 속임)
```

남용 X — *진짜로 mypy 가 못 잡는 경우* (런타임 introspection, dynamic import) 만.

---

## §7 함정

### 7.1 런타임에 안 잡힘

```python
def add(a: int, b: int) -> int:
    return a + b

add("1", "2")    # 작동 (str + str)
```

→ mypy / pyright 를 CI 에 포함 필수.

### 7.2 `list[int]` 가 `list[float]` 의 subtype 아님

```python
def f(xs: list[float]) -> None: ...
xs: list[int] = [1, 2]
f(xs)            # mypy error - list 는 invariant
```

→ Sequence 같은 covariant 컨테이너 사용.

```python
def f(xs: Sequence[float]) -> None: ...   # OK
```

### 7.3 forward reference

```python
class Order:
    def merge(self, other: "Order") -> "Order": ...  # 자기 자신 참조 시 string
```

3.11+ 의 `Self` 사용 권장. 또는 `from __future__ import annotations` (3.7+) 로 모든 annotation 을 string 으로 lazy.

### 7.4 dataclass + frozen + mutable default

```python
@dataclass(frozen=True)
class Foo:
    items: list[int] = []     # ← Error
    items: list[int] = field(default_factory=list)   # ← OK
```

### 7.5 타입 힌트와 런타임 introspection

```python
import inspect

def f(x: int) -> str: ...
sig = inspect.signature(f)
sig.parameters['x'].annotation    # <class 'int'>
```

런타임에서도 `__annotations__` 로 접근 가능 — Pydantic / FastAPI / Dishka 가 이걸 활용.

### 7.6 Generic Protocol

```python
class Repo(Protocol[T]):
    async def find_by_id(self, id: UUID) -> T | None: ...
```

3.11+ 는 더 자연스러움. 02 장 참조.

---

## §8 진화 로드맵 (PEP 별)

| PEP | Python | 주요 |
|---|---|---|
| 484 | 3.5 | 기본 타입 힌트 |
| 526 | 3.6 | 변수 어노테이션 (`x: int = 1`) |
| 544 | 3.8 | Protocol |
| 585 | 3.9 | `list[int]` (lowercase generic) |
| 604 | 3.10 | `int | str` |
| 612 | 3.10 | ParamSpec (Callable 의 인자 추론) |
| 647 | 3.10 | TypeGuard |
| 673 | 3.11 | Self |
| 695 | 3.12 | `class Box[T]` 새 문법 |
| 696 | 3.12 | `type Alias = ...` |
| 705 | 3.13 | TypedDict 의 ReadOnly |

---

## §10 학습 포인트 (한 줄 요약)

1. **타입 힌트는 런타임 검사 X** — 정적 검사기 + 도구의 입력.
2. **3.9+** : `list[int]`, **3.10+** : `int | None`. 모던하게 써라.
3. **Protocol** : structural typing — 인터페이스 분리의 도구.
4. **Generic + TypeVar** — 컬렉션 / Repository 추상.
5. **Literal / Final** — 더 엄격한 제약.
6. **Self (3.11+)** — `cls` 반환의 자연스러운 표현.
7. **mypy strict + CI** — 타입 힌트의 가치는 검사기 없이는 절반.
8. **`Any` 는 escape hatch** — 함부로 쓰지 말기.
9. **Mapped[T]** — SQLAlchemy 의 typed declarative.
10. **타입 힌트는 *문서이자 도구*** — 생산성의 핵심 인프라.

---

## 추가 참고

- mypy : https://mypy.readthedocs.io/
- pyright : https://github.com/microsoft/pyright
- typing-extensions : 옛 Python 에서 신 기능 백포트
- ShopTracker 다음 글 : `15-python-dataclass.md`
