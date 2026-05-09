# 06 — Scope, LEGB, Closure, nonlocal/global

> 함수 안에서 `x` 를 쓸 때 Python 은 어디서 찾는가? 이 글은 *변수 lookup 의 4 단계* (LEGB), closure 의 작동, `nonlocal` / `global` 키워드를 정리.

---

## §0 LEGB 규칙

이름 lookup 순서 :

```
L — Local      (현재 함수의 로컬)
E — Enclosing  (감싸는 함수의 로컬)
G — Global     (모듈 레벨)
B — Built-in   (내장: print, len, ...)
```

```python
x = "global"

def outer():
    x = "enclosing"
    def inner():
        x = "local"
        print(x)         # local
    inner()

outer()
```

---

## §1 컴파일 시점에 결정

Python 은 *컴파일 시점* 에 함수 안의 변수가 local / enclosing / global 인지 결정.

```python
x = 1
def f():
    print(x)            # global x
    # x = 2             # ← 이 줄 추가만 해도 위 print(x) 가 UnboundLocalError 됨!
```

```python
x = 1
def f():
    print(x)            # UnboundLocalError - x 가 local 로 *분류* 됐지만 아직 할당 X
    x = 2
```

→ 함수 안에 `x = ...` 가 있으면 *함수 전체에서* x 는 local. compiler 가 그렇게 marking.

```python
import dis
dis.dis(f)
# LOAD_FAST 0 (x)    ← global 이면 LOAD_GLOBAL 였을 것
```

---

## §2 global / nonlocal

### 2.1 global

```python
x = 1

def f():
    global x
    x = 2

f()
print(x)             # 2
```

`global x` = "이 함수에서 x 는 모듈 레벨". 안 그러면 local 새 변수 생성.

### 2.2 nonlocal (3.0+)

```python
def outer():
    x = 1
    def inner():
        nonlocal x
        x = 2
    inner()
    print(x)         # 2

outer()
```

`nonlocal x` = "x 는 enclosing scope". 없으면 inner 의 local 새로 만듦.

### 2.3 global vs nonlocal

| | global | nonlocal |
|---|---|---|
| 대상 | 모듈 레벨 | 가장 가까운 enclosing 함수 |
| 새로 만들기 | OK (없으면 생성) | X (없으면 SyntaxError) |
| 사용처 | 드물게 — 보통 안 좋은 신호 | closure 안에서 mutation |

---

## §3 Closure

### 3.1 기본

```python
def make_counter():
    count = 0
    def counter():
        nonlocal count
        count += 1
        return count
    return counter

c = make_counter()
c(), c(), c()        # 1, 2, 3
```

`counter` 가 `count` 를 *기억* — 이게 closure. `make_counter` 가 끝나도 `count` 살아 있음 (refcnt).

### 3.2 cell 객체

```python
c.__closure__        # (<cell at 0x...: int object at 0x...>,)
c.__closure__[0].cell_contents   # 3 (현재 값)
```

closure 의 자유 변수는 *cell* 이라는 객체에 저장. cell 이 변수의 컨테이너 — 두 함수가 같은 cell 을 공유 가능.

### 3.3 late binding 함정

```python
fns = []
for i in range(3):
    fns.append(lambda: i)

[f() for f in fns]   # [2, 2, 2] - 모두 마지막 i!
```

→ closure 가 *변수* (i) 를 캡처 — 값이 아님. 루프 끝나면 i = 2 였으므로 모두 2.

해결 :
```python
fns = []
for i in range(3):
    fns.append(lambda i=i: i)   # default 인자로 *값* 캡처

# 또는
fns = [lambda: i for i in [0, 1, 2]]   # comprehension 의 새 scope
```

JS / 다른 언어에서도 같은 함정 (var vs let).

---

## §4 함수의 scope

```python
def f():
    x = 1
    if True:
        x = 2
        y = 3
    print(x, y)      # 2, 3 — Python 은 블록 scope X
```

Python 은 *블록 scope* 가 없다 (if / for / while 안에서 만든 변수가 밖에서 보임). C / Java / Rust 와 다름.

단, **comprehension 은 자기 scope** :

```python
[x for x in range(10)]
print(x)             # NameError (3.x) - comprehension 변수는 밖으로 안 새 나감
```

(2.x 에서는 새 나감 — 3.x 에서 수정.)

---

## §5 모듈 레벨 = global

```python
# foo.py
counter = 0

def inc():
    global counter
    counter += 1
```

"global" 은 *모듈 레벨* 이지 *프로세스 전체* 가 아님. 다른 모듈의 global 과 별개.

---

## §6 Built-in scope

```python
print, len, list, range, ...
```

`builtins` 모듈에 다 있음.

```python
import builtins
print(dir(builtins))
```

shadow 가능 :

```python
list = "no"        # 새 global 'list' 생성
list([1, 2])       # TypeError - 이제 list 는 str
del list           # builtin 다시 보임
```

이런 shadowing 은 버그의 원인. `id`, `type`, `list`, `dict` 같은 이름은 변수명으로 쓰지 말기.

---

## §7 Class scope 의 묘함

```python
class Foo:
    x = 1
    def f(self):
        return x        # NameError - 클래스 scope 는 함수에서 안 보임!

    y = x + 1            # OK - 클래스 본문 안에서는 보임
```

→ **클래스 본문은 *enclosing scope 가 아님***. 메서드 안에서 클래스 변수는 `self.x` 또는 `Foo.x`.

---

## §8 ShopTracker 와 LEGB

```python
# orders/application/command_handlers.py
from app.shared.events import OrderCreatedEvent     # global (모듈 레벨)

class CreateOrderHandler:
    def __init__(self, repo, event_bus):
        self._repo = repo                            # 인스턴스 속성 (scope 아님)

    async def handle(self, command):
        order = Order.create(...)                    # global Order
        await self._repo.save(order)                 # local order
```

- `Order`, `OrderCreatedEvent` = global (import).
- `command`, `order` = local.
- closure 안 씀 (대부분의 OOP 코드처럼).

closure 가 자주 쓰이는 곳 : **데코레이터** (17 장).

```python
def log(func):
    def wrapper(*args, **kwargs):
        print(f"call {func.__name__}")    # func 이 closure
        return func(*args, **kwargs)
    return wrapper
```

`wrapper` 가 `func` 을 closure 로 잡음.

---

## §9 함정

### 9.1 함수 안의 augmented assignment

```python
x = 1
def f():
    x += 1           # UnboundLocalError - x 가 local 로 분류
```

`x += 1` 은 `x = x + 1` 인데 LHS 가 있어 local 로 marking. RHS 의 x 는 아직 없음.

→ `global x` 또는 인자로.

### 9.2 closure 가 무거운 객체 잡음

```python
def make(big_data):
    def f():
        print(len(big_data))
    return f

handler = make(load_huge())   # big_data 가 handler 가 살아있는 동안 메모리에
```

03 장 §5.2 와 같은 누수.

### 9.3 nonlocal 대신 mutable 컨테이너

```python
# nonlocal 없이 closure 에서 mutation
def make():
    state = [0]                # list 자체는 enclosing, 안의 0 을 변경
    def inc():
        state[0] += 1
        return state[0]
    return inc
```

opaque 하지만 작동. 옛날 패턴.

### 9.4 global 남발

전역 상태는 테스트 / 동시성 에서 문제. 의존성 주입 (03 장) 으로 대체.

---

## §10 학습 포인트 (한 줄 요약)

1. **LEGB** : Local → Enclosing → Global → Built-in.
2. **컴파일 시점에 결정** — 함수 안 어디든 `x = ...` 면 x 는 local.
3. **`global` / `nonlocal`** 로 명시적으로 다른 scope 변경.
4. **closure** = 자유 변수를 cell 로 캡처.
5. **late binding 함정** — 루프 + lambda 는 default 인자 트릭.
6. **블록 scope 없음** — if/for 변수가 밖으로.
7. **comprehension 은 자기 scope** (3.x).
8. **클래스 본문은 enclosing 아님** — 메서드 안에서 self/Foo 명시.
9. **builtin shadow 주의** — list / dict / id 변수명 X.
10. **데코레이터 = closure 의 흔한 활용**.

---

## 참고

- Python docs — Execution model / Naming and binding
- "Fluent Python" Ramalho — Function 챕터
