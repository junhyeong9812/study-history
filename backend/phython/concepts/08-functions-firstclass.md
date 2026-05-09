# 08 — 함수의 정체 (first-class object)

> Python 에서 함수는 *값* 이다. 변수에 담고, 인자로 넘기고, 반환받고, 컨테이너에 저장. 이 일관성 위에 데코레이터 / 콜백 / DI / 함수형 패턴이 다 올라간다.

---

## §0 first-class function

```python
def add(a, b): return a + b

f = add               # 변수에 담기
f(1, 2)               # 3

def call(fn, x):
    return fn(x)
call(str, 1)          # "1"

ops = [add, lambda a, b: a*b]
[op(2, 3) for op in ops]   # [5, 6]
```

함수도 객체 (04 장) — `__name__`, `__doc__`, `__module__`, `__code__`, `__globals__`, `__defaults__` 같은 속성을 가짐.

---

## §1 함수의 구성요소

```python
def f(a, b=10, *args, c, **kwargs):
    """doc."""
    return a + b

f.__name__            # 'f'
f.__doc__             # 'doc.'
f.__module__          # '__main__'
f.__defaults__        # (10,)
f.__kwdefaults__      # None (c 의 default 없음)
f.__code__            # code object
f.__globals__         # 모듈의 globals dict
f.__closure__         # 자유 변수 (없으면 None)
f.__annotations__     # {} (타입 힌트)
```

이 모든 게 *동적 변경* 가능. 이게 데코레이터의 토대.

---

## §2 인자 패턴

### 2.1 positional / keyword

```python
def f(a, b, c): ...

f(1, 2, 3)            # positional
f(a=1, b=2, c=3)      # keyword
f(1, c=3, b=2)        # 혼합
```

### 2.2 default

```python
def f(a, b=10, c=20): ...
f(1)                   # b=10, c=20
f(1, 5)                # b=5, c=20
f(1, c=99)             # b=10, c=99
```

### 2.3 *args / **kwargs

```python
def f(*args, **kwargs):
    print(args)        # tuple
    print(kwargs)      # dict

f(1, 2, x=3, y=4)
# (1, 2)
# {'x': 3, 'y': 4}
```

### 2.4 keyword-only / positional-only

```python
def f(a, b, *, c, d):           # *, 다음은 keyword-only
    pass
f(1, 2, c=3, d=4)               # OK
f(1, 2, 3, 4)                   # TypeError

def g(a, b, /, c):              # /, 이전은 positional-only (3.8+)
    pass
g(1, 2, 3)                      # OK
g(a=1, b=2, c=3)                # TypeError
```

### 2.5 unpack

```python
def f(a, b, c): ...

args = (1, 2, 3)
f(*args)                        # 1, 2, 3

kwargs = {"a": 1, "b": 2, "c": 3}
f(**kwargs)
```

`*args, **kwargs` 패턴은 데코레이터의 흔한 시그니처 (17 장).

---

## §3 lambda

```python
add = lambda a, b: a + b
add(1, 2)              # 3

sorted(xs, key=lambda x: x.age)
```

- *식 (expression)* 만 가능 — 문 (statement) X.
- 한 줄.
- 이름 없음 (`__name__ == "<lambda>"`).
- 복잡하면 그냥 `def`.

---

## §4 functools

### 4.1 partial

```python
from functools import partial

def power(base, exp): return base ** exp
square = partial(power, exp=2)
square(5)              # 25
```

함수의 *일부 인자를 미리 채움* — 새 함수 생성. 콜백 / DI 에 유용.

### 4.2 reduce

```python
from functools import reduce
reduce(lambda a, b: a + b, [1, 2, 3, 4])    # 10
```

### 4.3 lru_cache

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fib(n):
    if n < 2: return n
    return fib(n-1) + fib(n-2)

fib(100)               # 즉시. 캐시 덕에.
```

### 4.4 cache (3.9+)

```python
from functools import cache

@cache
def heavy(x): ...
```

`lru_cache(maxsize=None)` 의 단축.

### 4.5 wraps

```python
def deco(f):
    @wraps(f)
    def wrapper(*a, **kw): return f(*a, **kw)
    return wrapper
```

원본 함수의 메타데이터 (`__name__` etc) 복사. 17 장 §1.3.

### 4.6 singledispatch

```python
from functools import singledispatch

@singledispatch
def render(obj): return str(obj)

@render.register(int)
def _(obj): return f"int: {obj}"

@render.register(list)
def _(obj): return f"list[{len(obj)}]"

render(1)          # "int: 1"
render([1,2])      # "list[2]"
render("hi")       # "hi"
```

타입 기반 dispatch. JS / Lisp 의 multimethod 비슷.

---

## §5 inspect — 함수 introspection

```python
import inspect

def f(a, b: int = 10) -> str: ...

sig = inspect.signature(f)
print(sig)             # (a, b: int = 10) -> str
sig.parameters         # OrderedDict
sig.return_annotation  # <class 'str'>

inspect.iscoroutinefunction(f)
inspect.getsource(f)
inspect.getfile(f)
```

FastAPI / Pydantic / Dishka 가 모두 inspect 로 함수 시그니처 분석 → 인자 자동 주입.

---

## §6 callable

```python
def f(): pass
class Foo:
    def __call__(self): pass

callable(f)            # True
callable(Foo())        # True (because __call__)
callable(Foo)          # True (call → __init__)
callable(1)            # False
```

`__call__` 정의된 객체는 함수처럼. 17 장의 클래스 데코레이터가 이 패턴.

---

## §7 ShopTracker 의 활용

### 7.1 Dishka 가 function signature 분석

```python
@provide(scope=Scope.REQUEST)
def order_repo(self, session: AsyncSession) -> OrderRepositoryProtocol:
    return SQLAlchemyOrderRepository(session)
```

Dishka 가 :
- `session` 파라미터의 타입 힌트 (`AsyncSession`) 봄
- 컨테이너에서 그 타입을 찾아 주입
- 반환 타입 (`OrderRepositoryProtocol`) 으로 등록

→ inspect 로 signature 분석 → 자동 DI.

### 7.2 FastAPI 라우트

```python
@router.post("/")
async def create_order(
    body: CreateOrderRequest,
    handler: FromDishka[CreateOrderHandler],
) -> OrderResponse:
    ...
```

FastAPI 가 :
- `body` 의 타입 (Pydantic) → JSON body 파싱
- `handler` 의 타입 (FromDishka[T]) → DI 주입
- 반환 타입 (OrderResponse) → JSON 직렬화 + OpenAPI 스키마

함수 시그니처가 *선언* — Python 의 first-class 함수 + 타입 힌트의 강력함.

### 7.3 EventBus 핸들러 등록

```python
event_bus.subscribe(OrderCreatedEvent, payment_handler.on_order_created)
```

`on_order_created` = 메서드 = 함수. 변수처럼 넘김.

---

## §8 함정

### 8.1 mutable default

```python
def f(x=[]):           # 함수 객체 생성 시 한 번 평가
    x.append(1)
    return x
```

→ 04 장 §8.1, 05 장 §7.1, 15 장 §3.

### 8.2 lambda 의 late binding

```python
fns = [lambda: i for i in range(3)]
[f() for f in fns]     # [2, 2, 2]
```

→ 06 장 §3.3.

### 8.3 partial 의 keyword 우선순위

```python
def f(a, b): return a, b
g = partial(f, b=10)
g(1)               # (1, 10)
g(1, b=20)         # (1, 20)
g(1, 2)            # TypeError - b 가 두 번 (positional + 이미 partial 에)
```

### 8.4 호출과 reference 혼동

```python
threading.Thread(target=f())     # ← f() 호출 후 결과를 target 에! 의도와 다름
threading.Thread(target=f)       # 함수 자체를 넘김
```

콜백 등록 시 흔한 실수.

---

## §9 다른 언어

| 언어 | first-class? |
|---|---|
| **Python** | yes |
| **JS** | yes (function expression, arrow) |
| **Go** | yes |
| **Rust** | closures (FnOnce/FnMut/Fn trait) |
| **Java** | lambda (8+), 함수형 인터페이스로 wrap |
| **C** | function pointer (제한적) |

함수형 패턴 = first-class function 의 유무에 크게 의존.

---

## §10 학습 포인트 (한 줄 요약)

1. **함수도 객체** — 변수에 담고, 인자로, 반환.
2. **함수의 메타데이터** : `__name__`, `__code__`, `__defaults__`, `__closure__`.
3. **\*args / \*\*kwargs** = 가변 인자. 데코레이터의 표준.
4. **keyword-only (`*,`) / positional-only (`/`,)** — 인자 순서 강제.
5. **lambda** = 짧은 1 줄 익명 함수. 길면 def.
6. **functools** — partial / reduce / lru_cache / wraps / singledispatch.
7. **inspect.signature** — 런타임 시그니처 분석. DI / FastAPI 의 토대.
8. **`__call__`** = callable 객체. 클래스를 함수처럼.
9. **mutable default / lambda late binding** = 흔한 함정.
10. **함수 등록 시 `f` vs `f()` 혼동 주의**.

---

## 참고

- functools docs
- inspect docs
- "Fluent Python" Ramalho — 함수형 패턴 챕터
