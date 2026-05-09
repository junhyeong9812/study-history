# 09 — Class / MRO / super / Descriptor / Metaclass

> 단일 상속만 쓸 때는 신경 안 써도 되지만, 다중 상속 + super() 가 들어오면 *MRO (Method Resolution Order)* 가 보이기 시작한다. 그 너머에 *descriptor* (모든 메서드의 토대), *metaclass* (클래스의 클래스). 이 글은 그 계층의 메커니즘.

---

## §0 클래스의 본질

```python
class Foo:
    x = 1
    def hi(self): return "hi"

# 위는 사실
Foo = type("Foo", (object,), {"x": 1, "hi": lambda self: "hi"})
```

`class` 키워드 = `type()` 의 3-인자 호출 설탕.

---

## §1 단일 상속 + super

```python
class Animal:
    def __init__(self, name): self.name = name
    def speak(self): return "..."

class Dog(Animal):
    def __init__(self, name, breed):
        super().__init__(name)
        self.breed = breed
    def speak(self): return "Woof"
```

- `super().__init__(name)` = 부모의 __init__.
- `super()` 는 사실 `super(Dog, self)` 의 단축 (3.0+).

---

## §2 다중 상속 + MRO

```python
class A:
    def f(self): return "A"

class B(A):
    def f(self): return "B"

class C(A):
    def f(self): return "C"

class D(B, C): pass

d = D()
d.f()                # "B"
D.__mro__            # (D, B, C, A, object)
```

MRO = 메서드 lookup 순서. `D` → `B` → `C` → `A` → `object`.

### 2.1 C3 linearization

Python 의 MRO 알고리즘 (Python 2.3+). 규칙 :
1. 각 클래스의 부모 순서 보존.
2. 다이아몬드 상속에서도 일관된 순서.
3. depth-first 가 아닌 *모든 부모를 먼저 본 후 조부모* (BFS-ish).

```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass
# MRO: D, B, C, A, object  (A 가 B, C 모두 본 후에)
```

### 2.2 super() 의 마법

```python
class A:
    def f(self): print("A"); 

class B(A):
    def f(self): print("B"); super().f()

class C(A):
    def f(self): print("C"); super().f()

class D(B, C):
    def f(self): print("D"); super().f()

D().f()
# D
# B          ← super() 가 MRO 따라 다음 = C
# C
# A
```

**super() 는 부모가 아니라 *MRO 의 다음*** — 다중 상속에서 핵심.

---

## §3 메서드 종류

### 3.1 instance method

```python
class Foo:
    def f(self, x): ...        # self = 인스턴스
```

### 3.2 classmethod

```python
class Foo:
    @classmethod
    def from_dict(cls, d):     # cls = 클래스 자체
        return cls(...)

Foo.from_dict({})              # cls = Foo
```

ShopTracker 의 `Order.create`, `SubscriptionContext.guest` 가 classmethod factory.

### 3.3 staticmethod

```python
class Foo:
    @staticmethod
    def util(x): ...           # self/cls 없음
```

그냥 함수를 클래스 namespace 에. *문서화* / *그룹핑* 목적.

### 3.4 property

```python
class Money:
    def __init__(self, amount): self._amount = amount

    @property
    def amount(self): return self._amount

    @amount.setter
    def amount(self, v):
        if v < 0: raise ValueError()
        self._amount = v

m = Money(100)
m.amount        # 100 (메서드 호출처럼 안 보임)
m.amount = -1   # ValueError
```

attribute access 를 메서드로 가로채기.

---

## §4 Descriptor Protocol

`__get__`, `__set__`, `__delete__` 가 정의된 객체. property / classmethod / staticmethod 가 *모두 descriptor*.

```python
class Lower:
    """str 속성을 자동 lowercase."""
    def __set_name__(self, owner, name):
        self.name = "_" + name
    def __get__(self, obj, cls):
        if obj is None: return self
        return getattr(obj, self.name)
    def __set__(self, obj, v):
        setattr(obj, self.name, v.lower())

class User:
    name = Lower()

u = User()
u.name = "Alice"
u.name           # "alice"
```

descriptor 는 *모든 인스턴스에 공유* — 클래스 변수.

### 4.1 data vs non-data descriptor

- **data** : `__set__` (또는 `__delete__`) 정의. instance dict 보다 우선.
- **non-data** : `__get__` 만. instance dict 가 우선.

→ instance dict 의 `f.x = 99` 가 property 를 *못 덮는* 것은 property 가 data descriptor 라서.

### 4.2 SQLAlchemy 가 descriptor 사용

```python
class OrderModel(Base):
    id: Mapped[str] = mapped_column(String(36), primary_key=True)
```

`mapped_column(...)` 이 descriptor 반환 — 인스턴스의 속성 접근을 가로채 *DB 값* 으로 매핑.

---

## §5 Abstract Base Class (ABC)

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self): ...

Shape()              # TypeError - abstract method 미구현
```

서브클래스가 `area` 구현 안 하면 인스턴스화 불가.

ShopTracker 는 ABC 안 쓰고 Protocol (02 장). 차이 :

| | ABC | Protocol |
|---|---|---|
| 종류 | nominal (명시적 상속) | structural (duck typing) |
| 강제 | 인스턴스화 시 | 정적 검사 (mypy) |
| 등록 | `class Foo(Shape):` | 자동 |
| Python | 항상 | 3.8+ |

---

## §6 Metaclass

> *클래스의 클래스*. `type` 이 모든 클래스의 metaclass.

```python
class Meta(type):
    def __new__(mcs, name, bases, namespace):
        print(f"creating class {name}")
        namespace["created_by"] = "Meta"
        return super().__new__(mcs, name, bases, namespace)

class Foo(metaclass=Meta):
    pass
# creating class Foo

Foo.created_by       # "Meta"
```

활용 :
- ORM (Django, SQLAlchemy 일부) — 클래스 정의 시 자동 등록.
- Plugin 시스템.
- Validation framework.

대부분의 코드는 metaclass 안 씀. **데코레이터로 같은 효과** 가능 (17 장). 단, *클래스 정의 시점에 hook* 이 필요하면 metaclass.

---

## §7 `__init_subclass__` (3.6+) — metaclass 의 가벼운 대안

```python
class Plugin:
    plugins = []
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        cls.plugins.append(cls)

class A(Plugin): pass
class B(Plugin): pass
Plugin.plugins       # [A, B]
```

metaclass 없이 *서브클래스 생성 시* hook. 가독성 좋음.

---

## §8 ShopTracker 코드와 연결

### 8.1 dataclass = decorator + descriptor

```python
@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str
```

`dataclass` 가 `__init__`, `__eq__`, `__hash__` 메서드를 클래스에 주입. metaclass 안 씀.

### 8.2 SQLAlchemy = metaclass + descriptor

```python
class OrderModel(Base):
    id: Mapped[str] = mapped_column(String(36), primary_key=True)
```

`Base` (DeclarativeBase) 가 metaclass 사용 — 클래스 정의 시 매핑 등록.

### 8.3 Protocol = ABC 의 사촌

```python
class OrderRepositoryProtocol(Protocol):
    async def save(self, order: Order) -> None: ...
```

`Protocol` 자체가 metaclass + 특수 처리. 02 장.

---

## §9 함정

### 9.1 super() 안 부르고 다중 상속

```python
class B(A):
    def __init__(self):
        # super().__init__() 안 함
        ...
```

→ `A` 의 init 안 돌아 → 상태 깨짐. 다중 상속에서 모든 클래스가 super() 호출해야 협력.

### 9.2 다이아몬드의 init 인자 차이

```python
class A:
    def __init__(self, x): self.x = x
class B(A):
    def __init__(self, x, y): super().__init__(x); self.y = y
class C(A):
    def __init__(self, x, z): super().__init__(x); self.z = z
class D(B, C): ...   # 인자 시그니처 충돌
```

→ 다중 상속은 *시그니처 합의* 필요. 어렵다. **mixin 으로 해결** (메서드 추가만, init 안 건드림).

### 9.3 metaclass 충돌

```python
class A(metaclass=M1): ...
class B(metaclass=M2): ...
class C(A, B): ...   # TypeError - 두 metaclass
```

→ M3(M1, M2) 새로 만들어야.

### 9.4 classmethod 가 staticmethod 와 헷갈림

```python
class Foo:
    @classmethod
    def f(cls): return cls.__name__
```

`cls` 는 *호출한 클래스* — 서브클래스에서 호출하면 서브클래스. polymorphism 활용 가능.

---

## §10 학습 포인트 (한 줄 요약)

1. **클래스도 객체** — `type()` 으로 동적 생성 가능.
2. **MRO** = C3 linearization. `Cls.__mro__` 로 확인.
3. **super() = MRO 의 다음** (부모 아님).
4. **classmethod / staticmethod / property** = descriptor 의 활용.
5. **descriptor protocol** = `__get__/__set__`. 모든 메서드의 토대.
6. **data vs non-data descriptor** — instance dict 와의 우선순위.
7. **ABC** = nominal, **Protocol** = structural — 02 장.
8. **metaclass** = 클래스의 클래스. ORM / framework 에 사용.
9. **`__init_subclass__`** = metaclass 의 가벼운 대안.
10. **다중 상속의 super 협력** — 시그니처 합의 또는 mixin.

---

## 참고

- "Python's super() considered super!" — Raymond Hettinger
- "Fluent Python" — descriptor / metaclass
- C3 linearization 알고리즘 : https://www.python.org/download/releases/2.3/mro/
