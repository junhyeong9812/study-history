# 04 — 객체 모델 ("Everything is Object")

> Python 에서 *모든 것* — 함수, 클래스, 모듈, 타입, 정수 — 이 객체다. `1.bit_length()`, `int.__add__`, `type(type) is type`. 이 일관성이 Python 의 메타프로그래밍 능력의 토대.

---

## §0 가장 기본적인 사실

```python
1                       # int 객체
"hello"                 # str 객체
[1, 2]                  # list 객체
print                   # function 객체
int                     # type 객체
type                    # type 객체 (자기 자신!)

print(type(1))          # <class 'int'>
print(type(int))        # <class 'type'>
print(type(type))       # <class 'type'>
```

모든 것이 객체 = 모든 것이 :
- `type()` 으로 타입 조회 가능
- 변수에 할당 가능
- 함수 인자로 전달 가능
- 컨테이너에 저장 가능
- 속성 (`.foo`) 접근 가능 (타입에 따라)

---

## §1 객체의 3 요소

모든 객체는 :

1. **identity** (`id()`) — 메모리 주소. 객체 일생 동안 불변.
2. **type** (`type()`) — 객체의 클래스.
3. **value** — 실제 데이터.

```python
x = [1, 2]
id(x)        # 140234567890 (예시)
type(x)      # <class 'list'>
x            # [1, 2]
```

`is` = identity 비교. `==` = value 비교.

```python
a = [1, 2]; b = [1, 2]
a is b       # False — 다른 객체
a == b       # True — 같은 값
```

---

## §2 type vs class

Python 3 부터 type ≡ class. 같은 것의 두 이름.

```python
class Foo: ...
type(Foo)            # <class 'type'>
Foo.__class__        # <class 'type'>

isinstance(Foo, type)    # True - 클래스도 type 의 인스턴스
```

→ 클래스 자체가 *type 클래스의 인스턴스*. 메타클래스의 출발점.

### 2.1 type() 의 두 용도

```python
type(1)              # 1 인자 — 객체의 타입 조회
type("Foo", (), {"x": 1})   # 3 인자 — 동적 클래스 생성
```

3 인자 형태 = 메타클래스 호출 (09 장).

---

## §3 dunder 메서드 (special methods)

`__init__`, `__add__`, `__len__` 같은 *언더스코어 두 개* 메서드. **연산자 / 내장 함수가 호출하는 hook**.

```python
1 + 2                # int.__add__(1, 2)
len([1, 2])          # list.__len__([1, 2])
str(1)               # int.__str__(1)
[1, 2][0]            # list.__getitem__([1, 2], 0)
```

→ 우리도 정의 가능 :

```python
class Money:
    def __init__(self, amount):
        self.amount = amount
    def __add__(self, other):
        return Money(self.amount + other.amount)
    def __repr__(self):
        return f"Money({self.amount})"

Money(10) + Money(20)    # Money(30)
```

전체 목록 : "Python data model" docs.

자주 쓰는 :

| 메서드 | 호출 트리거 |
|---|---|
| `__init__` | 인스턴스 생성 후 |
| `__new__` | 인스턴스 생성 (할당) |
| `__call__` | `obj()` |
| `__repr__` / `__str__` | 출력 |
| `__eq__` / `__hash__` | `==` / hash() |
| `__lt__` etc | 비교 |
| `__len__` / `__getitem__` / `__contains__` | 컨테이너 |
| `__iter__` / `__next__` | iteration |
| `__enter__` / `__exit__` | `with` |
| `__add__` / `__mul__` etc | 연산자 |
| `__getattr__` / `__setattr__` | 속성 접근 |

---

## §4 속성 접근의 메커니즘

`obj.attr` 의 lookup 순서 :

1. `type(obj).__mro__` 의 *data descriptor* (예 : property) — 04.5
2. `obj.__dict__` (인스턴스 속성)
3. `type(obj).__mro__` 의 *non-data descriptor* / 평범한 속성
4. `__getattr__` (없으면 AttributeError)

```python
class Foo:
    cls_attr = 1
    def method(self): ...

f = Foo()
f.x = 10              # f.__dict__['x'] = 10

f.cls_attr            # 1 — 클래스에서 찾음
f.cls_attr = 99       # 인스턴스에 새로 만듦, 클래스는 그대로
Foo.cls_attr          # 여전히 1
```

### 4.1 `__slots__`

```python
class Foo:
    __slots__ = ("x", "y")

f = Foo()
f.x = 1
f.z = 2          # AttributeError - slots 에 없음
```

- `__dict__` 안 만들음 → 메모리 ↓ (200B → 64B 정도).
- 동적 속성 추가 차단.
- ShopTracker 의 Money (09 장) 가 사용.

### 4.2 descriptor

`__get__` / `__set__` 정의된 객체. `property` / `classmethod` / `staticmethod` 가 모두 descriptor.

```python
class Cached:
    def __init__(self, func): self.func = func; self.name = func.__name__
    def __get__(self, obj, cls):
        if obj is None: return self
        result = self.func(obj)
        setattr(obj, self.name, result)
        return result

class Foo:
    @Cached
    def heavy(self):
        return compute()

f = Foo()
f.heavy        # 첫 호출 — 계산
f.heavy        # 두 번째 — 인스턴스 dict 에 박혀서 즉시
```

descriptor protocol = Python 의 가장 강력한 메타프로그래밍.

---

## §5 인스턴스화의 흐름

```python
class Foo:
    def __new__(cls, x):
        print("__new__")
        return super().__new__(cls)
    def __init__(self, x):
        print("__init__")
        self.x = x

Foo(1)
# __new__
# __init__
```

- `__new__` : 객체 생성 (메모리 할당). cls 받음. 새 객체 반환.
- `__init__` : 초기화. 이미 만들어진 self 받음. None 반환.

대부분 `__new__` 안 씀 — singleton, immutable subclass 등 특수한 경우만.

```python
class Singleton:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

---

## §6 isinstance / issubclass

```python
isinstance(1, int)              # True
isinstance(True, int)           # True - bool 은 int 의 서브클래스!
isinstance([], (list, tuple))   # True - 튜플로 여러 타입
issubclass(bool, int)           # True
```

내부적으로 `__instancecheck__` / `__subclasscheck__` (메타클래스의 메서드) 호출. 그래서 ABC / Protocol 이 동작.

---

## §7 일반 타입 계층

```
object
 ├── int ── bool
 ├── float
 ├── complex
 ├── str
 ├── bytes
 ├── tuple
 ├── list
 ├── dict
 ├── set / frozenset
 ├── function
 ├── type
 └── ... (수백 가지)
```

`object` 가 모든 것의 root. `type` 도 `object` 의 인스턴스, `object` 도 `type` 의 인스턴스 — 닭과 달걀. 인터프리터 부트스트랩 시 동시에 생성.

```python
isinstance(object, type)         # True
isinstance(type, object)         # True
type(object)                     # <class 'type'>
type(type)                       # <class 'type'>
```

---

## §8 함정

### 8.1 mutable default

```python
def f(x=[]):           # ← 함수 객체에 한 번 만들어진 list
    x.append(1)
    return x
f()    # [1]
f()    # [1, 1] — 같은 객체
```

→ 01 장 §8.3, 15 장 §3.

### 8.2 클래스 변수 vs 인스턴스 변수

```python
class Foo:
    items = []          # 클래스 변수

a = Foo(); b = Foo()
a.items.append(1)
b.items                 # [1] — 같은 list 공유!
```

→ 가변 클래스 변수 X. 인스턴스 별로 원하면 `__init__` 에서.

### 8.3 monkey patching

```python
def new_method(self): ...
Foo.method = new_method   # 모든 Foo 인스턴스가 영향받음
```

런타임에 클래스 변경 가능. 강력하지만 디버깅 지옥.

### 8.4 dynamic attr

```python
class Bag:
    def __getattr__(self, name):
        return f"got {name}"

b = Bag()
b.foo                # "got foo" - 어떤 속성이든 응답
```

`__getattr__` 은 *없는 속성* 에만 호출. `__getattribute__` 는 *모든 속성*.

---

## §9 Duck Typing

```python
def quack(thing):
    return thing.quack()

class Duck: def quack(self): return "quack"
class Person: def quack(self): return "I'm a duck"

quack(Duck()), quack(Person())   # 둘 다 작동
```

> "If it walks like a duck and quacks like a duck..."

타입 검사 없이 *동작* 으로 호환. Protocol (02 장) 이 이를 *타입 시스템* 화.

---

## §10 학습 포인트 (한 줄 요약)

1. **Everything is object** — 함수, 클래스, 모듈, 타입까지.
2. **객체 = id + type + value**.
3. **`is` vs `==`** — identity vs value.
4. **type ≡ class** — 메타클래스의 출발.
5. **dunder methods** = 연산자 / 내장 함수의 hook.
6. **속성 lookup** : data descriptor → instance dict → class mro → __getattr__.
7. **`__slots__`** : 메모리 + 속성 차단.
8. **descriptor** = property / classmethod / staticmethod 의 기반.
9. **mutable default / 클래스 가변 속성** = 흔한 함정.
10. **Duck typing → Protocol** : Python 의 자연스러운 타입 사고.

---

## 참고

- Python data model : https://docs.python.org/3/reference/datamodel.html
- Luciano Ramalho, *Fluent Python* — 객체 / descriptor 챕터
