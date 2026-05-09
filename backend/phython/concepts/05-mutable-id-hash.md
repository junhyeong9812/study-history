# 05 — Mutable / Immutable, id, hash, equality

> `[1, 2] == [1, 2]` 는 True 인데 `is` 는 False. 두 객체가 같은가? 어떤 객체를 dict 키로 쓸 수 있나? 이 글은 Python 의 *동등성 / 식별성 / 해시 가능성* 의 계약을 정리.

---

## §0 mutable vs immutable

| immutable (변경 불가) | mutable (변경 가능) |
|---|---|
| int, float, complex, bool | list |
| str, bytes | bytearray |
| tuple, frozenset | set, dict |
| None | 사용자 정의 클래스 (기본) |
| `frozen dataclass` | 일반 dataclass |

```python
x = "hello"
x[0] = "H"           # TypeError - str 불변

x = [1, 2]
x[0] = 99            # OK - list 가변
```

immutable 은 *값을 바꿀 수 없다*. *변수가 가리키는 객체를 바꾼다* 와 다름.

```python
s = "hello"
s = s + " world"     # 새 str 객체. 변수 s 가 새 객체를 가리킴.
```

---

## §1 id, is, ==

### 1.1 id

```python
x = [1, 2]
id(x)        # 140234567890 (메모리 주소)
```

객체의 *수명 동안 유일* 한 식별자. 객체가 해제되면 다른 객체가 같은 id 받을 수 있음.

### 1.2 is

```python
a = [1, 2]; b = [1, 2]
a is b       # False — id 다름
a is a       # True

# None / True / False 는 singleton
None is None     # True
```

`is` = `id(a) == id(b)`. 메모리 동일성.

### 1.3 ==

```python
a == b       # True — __eq__ 호출, 값 비교
```

`__eq__` 가 정의된 대로. 기본 `object.__eq__` 는 identity 비교 (== `is`).

### 1.4 언제 어느 것?

| 상황 | 사용 |
|---|---|
| None 체크 | `x is None` (관용구) |
| singleton 체크 | `is` |
| 값 비교 | `==` |
| 기타 | 거의 항상 `==` |

`x == None` 도 작동하지만 `x is None` 이 빠르고 관용적.

---

## §2 hash

### 2.1 기본

```python
hash(1)              # 1
hash("hello")        # -7048932231...
hash((1, 2))         # 작동
hash([1, 2])         # TypeError - list unhashable
```

hashable = `__hash__` 가 None 이 아닌 객체. 모든 *immutable* 은 기본적으로 hashable.

### 2.2 왜 mutable 은 unhashable?

```python
d = {}
key = [1, 2]
d[key] = "value"     # 만약 가능했다면...
key.append(3)        # 키가 바뀜!
d[key]               # 더 이상 못 찾음
```

해시는 *객체의 수명 동안 변하지 않아야* 함. mutable 은 그 보장 못 함.

### 2.3 hash 와 eq 의 계약

> *`a == b` 면 `hash(a) == hash(b)`*.

역은 성립 안 해도 됨 (해시 충돌 가능).

```python
class Foo:
    def __init__(self, x): self.x = x
    def __eq__(self, o): return self.x == o.x
    # __hash__ 없으면? Python 이 자동으로 None 으로 만듦 → unhashable
    def __hash__(self): return hash(self.x)
```

`__eq__` 정의하면 `__hash__` 도 정의해야 — 안 그러면 set / dict 키 못 씀. dataclass 의 frozen=True 가 자동으로 처리 (15 장).

---

## §3 작은 정수 / 짧은 문자열 캐시

```python
a = 256; b = 256
a is b               # True - small int 캐시

a = 257; b = 257
a is b               # False - 캐시 범위 밖

a = "hello"; b = "hello"
a is b               # True - 식별자 같은 str 자동 intern
```

→ `is` 의 결과가 *우연히* True 인 경우가 있음. 의도적 비교는 항상 `==`.

---

## §4 동등성과 상속

```python
class Animal:
    def __init__(self, name): self.name = name
    def __eq__(self, o): return self.name == o.name

class Dog(Animal): ...

a = Animal("rex"); d = Dog("rex")
a == d               # True ?
```

→ True. 그런데 의도가 그런가? "Animal name=rex" 와 "Dog name=rex" 가 정말 같은가?

해결 :
```python
def __eq__(self, o):
    if type(o) is not type(self): return NotImplemented
    return self.name == o.name
```

또는 `isinstance(o, Animal)` (느슨하게).

ShopTracker 의 Money (09 장) 가 `isinstance(other, Money)` 후 비교 + `NotImplemented` 반환.

---

## §5 NotImplemented

```python
def __eq__(self, o):
    if not isinstance(o, MyType):
        return NotImplemented
    return ...
```

- `False` 와 다름!
- `NotImplemented` 반환 = "내가 모름, 다른 쪽에 물어봐".
- Python 이 `o.__eq__(self)` 시도, 그래도 모르면 `False` 로 결정.
- `False` 반환하면 그 자리에서 끝 — 다른 쪽이 *아는데도* False 가 됨.

---

## §6 mutable 을 안전하게 쓰는 패턴

### 6.1 복사

```python
import copy

a = [1, [2, 3]]
b = a.copy()         # shallow — a[1] 과 b[1] 은 같은 list
c = copy.deepcopy(a) # deep — 모두 독립
```

### 6.2 frozen 변환

```python
mutable = [1, 2, 3]
frozen = tuple(mutable)
hashable_set = frozenset({1, 2, 3})
```

### 6.3 immutable 인터페이스만 노출

```python
class Order:
    def __init__(self):
        self._items = []
    @property
    def items(self):
        return tuple(self._items)   # 외부엔 immutable view
```

ShopTracker 의 도메인이 이 패턴을 *완벽히는* 안 따르지만 (08 장 §2.5) 의도는 같음.

---

## §7 함정

### 7.1 default 인자가 mutable

```python
def f(x=[]):              # 함수 객체에 한 번 만들어짐
    x.append(1)
    return x
f()    # [1]
f()    # [1, 1]
```

→ `def f(x=None): if x is None: x = []`.

### 7.2 클래스 변수가 mutable

```python
class Bag:
    items = []

a = Bag(); b = Bag()
a.items.append(1)
b.items   # [1] - 공유
```

### 7.3 dict 키 변경

```python
d = {}
key = [1]                 # 가능한 mutable 사용 시도 → TypeError
key = (1, 2)
d[key] = "v"
# 만약 key 가 mutable 컨테이너 안에 있으면...
nested = {(1, [2]): "v"}  # TypeError - tuple 안의 list 가 unhashable
```

### 7.4 hash 가 의미 없는 값

```python
def __hash__(self):
    return 0           # 모든 인스턴스 같은 hash
```

→ dict / set 이 사실상 list 처럼 동작 (해시 충돌이 모든 키). 성능 O(n).

### 7.5 frozen 도 안의 mutable 은 통과

```python
@dataclass(frozen=True)
class Foo:
    items: list

f = Foo([1, 2])
f.items = []         # FrozenInstanceError
f.items.append(3)    # 작동! frozen 은 속성 재할당만 막음
```

→ 진정 immutable 은 tuple.

---

## §8 다른 언어 비교

| 언어 | mutable / immutable |
|---|---|
| **Python** | str/tuple immutable, list/dict mutable |
| **Java** | String final, primitives immutable, 객체는 명시적 |
| **JS** | string/number immutable, object mutable. Object.freeze() 옵션. |
| **Rust** | 모든 변수 기본 immutable, `mut` 명시. |
| **Haskell** | 모든 것 immutable. mutation 은 IO monad. |

immutable 우선 = 함수형 / 동시성 친화. Python 은 절충.

---

## §10 학습 포인트 (한 줄 요약)

1. **immutable** : str, tuple, frozenset, int, float, ... — 변경 불가.
2. **mutable** : list, dict, set, 일반 클래스.
3. **id / is** = identity, **==** = value.
4. **None 비교는 `is`** 관용구.
5. **hashable = `__hash__` 가 None 아님** → mutable 은 기본 unhashable.
6. **`__eq__` 정의 시 `__hash__` 도** — 계약.
7. **NotImplemented** ≠ False — 다른 쪽에 양보.
8. **small int / 짧은 str interning** = `is` 가 우연히 True.
9. **mutable default / class 변수** = 흔한 버그.
10. **frozen 도 안의 list 는 mutable** — 진정 불변은 tuple.

---

## 참고

- Python docs — Numeric types, Built-in types
- "Fluent Python" Ramalho — 데이터 모델 챕터
