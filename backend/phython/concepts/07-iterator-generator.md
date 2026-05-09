# 07 — Iterator / Generator / Lazy Evaluation

> `for x in xs:` 의 한 줄에 Python 의 *iterator protocol* 이 작동한다. generator 함수는 그 protocol 을 *yield 한 줄로* 구현해 준다. 이 글은 그 메커니즘과, lazy 시퀀스가 메모리 / 성능에 주는 영향.

---

## §0 iterator protocol

객체가 *iterable* 이려면 :
- `__iter__()` 가 *iterator* 를 반환.

객체가 *iterator* 이려면 :
- `__next__()` 가 다음 값 또는 `StopIteration` 발생.
- `__iter__()` 가 자기 자신 반환 (관용).

```python
class Counter:
    def __init__(self, n): self.n = n; self.i = 0
    def __iter__(self): return self
    def __next__(self):
        if self.i >= self.n: raise StopIteration
        self.i += 1
        return self.i

for x in Counter(3):
    print(x)        # 1, 2, 3
```

`for x in obj` 는 :
1. `it = iter(obj)`  # `__iter__` 호출
2. while True: `try: x = next(it)` ; `except StopIteration: break`

---

## §1 generator function

`yield` 가 있으면 함수가 *generator function* — 호출 시 generator object (= iterator) 반환.

```python
def counter(n):
    i = 0
    while i < n:
        i += 1
        yield i

c = counter(3)
next(c), next(c), next(c)    # 1, 2, 3
next(c)                       # StopIteration

for x in counter(3): print(x)
```

위 Counter 클래스 10 줄을 generator 4 줄로.

### 1.1 yield 의 작동

`yield` = "값 반환 + 함수 *일시정지*". 다음 `next()` 호출 시 *그 자리부터* 재개.

```python
def f():
    print("a"); yield 1
    print("b"); yield 2
    print("c")

g = f()
next(g)    # "a", 반환 1
next(g)    # "b", 반환 2
next(g)    # "c", StopIteration
```

내부적으로 frame 이 보존됨 (로컬 변수, 명령 포인터). 이게 바로 *coroutine* 의 본질 — async/await 도 이 메커니즘 위에.

### 1.2 generator expression

```python
g = (x*2 for x in range(10**7))     # 즉시 메모리 X
list(g)                              # 그제야 평가
```

list comprehension `[...]` 과 다름 — 후자는 즉시 모든 값 메모리.

---

## §2 lazy evaluation 의 가치

```python
# eager - 메모리 폭발
data = [process(x) for x in load_huge_file()]

# lazy - 한 번에 하나
data = (process(x) for x in load_huge_file())
for item in data:
    save(item)
```

generator 는 :
- 메모리 O(1).
- 첫 결과까지 빠름 (전체 평가 안 기다림).
- 무한 시퀀스 표현 가능.

---

## §3 표준 도구

### 3.1 내장

```python
range(10)              # iterable (3.x 에서 lazy)
zip([1,2], [3,4])      # iterator
map(str, [1,2,3])      # iterator (lazy)
filter(None, xs)       # iterator
enumerate(xs)          # iterator (인덱스, 값)
reversed(xs)           # iterator
```

3.x 에서 `range`, `map`, `filter`, `zip` 모두 lazy. 2.x 와 달라짐.

### 3.2 itertools

```python
import itertools as it

it.count(0)                          # 0, 1, 2, ... 무한
it.cycle([1, 2, 3])                  # 1, 2, 3, 1, 2, ... 무한
it.repeat("x", 3)                    # x, x, x

it.chain([1, 2], [3, 4])             # 1, 2, 3, 4
it.islice(it.count(), 0, 10, 2)      # 0, 2, 4, 6, 8

it.takewhile(lambda x: x < 5, it.count())   # 0, 1, 2, 3, 4
it.dropwhile(lambda x: x < 5, range(10))    # 5, 6, 7, 8, 9

it.groupby(sorted(xs, key=k), key=k)        # group by key

it.product([1,2], [3,4])             # (1,3), (1,4), (2,3), (2,4)
it.permutations([1,2,3])
it.combinations([1,2,3], 2)
```

함수형 시퀀스 처리. lazy.

### 3.3 functools.reduce

```python
from functools import reduce
reduce(lambda a, b: a + b, range(10))   # 45
```

lazy 가 아님 — 축약은 모든 값 평가.

---

## §4 무한 시퀀스

```python
def fibs():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

import itertools
list(itertools.islice(fibs(), 10))   # 0,1,1,2,3,5,8,13,21,34
```

generator 가 아니면 표현 불가. 메모리 무한 차지하면 안 되니까.

---

## §5 generator 의 send / throw / close

generator 는 단순 producer 이상 — *coroutine* 으로 활용 가능.

```python
def echo():
    while True:
        x = yield
        print(f"got {x}")

g = echo()
next(g)         # 처음 yield 까지 진행
g.send("hi")    # "got hi"
g.send("bye")   # "got bye"
g.close()       # 종료
g.throw(RuntimeError("oops"))   # 예외 던짐
```

이게 PEP 380 (yield from), PEP 492 (async/await) 의 토대. Python 의 async 는 generator 의 진화형.

---

## §6 yield from

```python
def gen1():
    yield 1; yield 2

def gen2():
    yield 0
    yield from gen1()   # subgenerator 위임
    yield 3

list(gen2())            # [0, 1, 2, 3]
```

가독성 + send/throw 위임 자동.

---

## §7 ShopTracker 의 generator 사용

### 7.1 Dishka session generator

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

*async generator* — `yield` 한 번. setup → yield → teardown 패턴 (18 장 context manager 와 결합).

### 7.2 pytest fixture

```python
@pytest.fixture
async def test_session():
    engine = create_async_engine(...)
    async with AsyncSession(engine) as s:
        yield s          # 여기까지 setup, 이후 teardown
    await engine.dispose()
```

같은 패턴.

---

## §8 함정

### 8.1 generator 한 번 소비

```python
g = (x for x in range(3))
list(g)        # [0, 1, 2]
list(g)        # [] - 이미 소진
```

→ 다시 쓰려면 `list` 로 변환 또는 generator 다시 생성.

### 8.2 generator 가 closure 의 변수 공유

```python
gens = [iter([i]*3) for i in range(3)]   # OK - 각자 독립
gens = [(i for _ in range(3)) for i in range(3)]
[list(g) for g in gens]
# [[2,2,2], [2,2,2], [2,2,2]] - late binding!
```

i 가 모든 generator 에서 같은 변수.

### 8.3 itertools.tee 의 메모리

```python
a, b = itertools.tee(some_iter)
# 두 iterator 가 *같은 iterable* 보지만 각자 진도
# 한쪽이 멀리 가면 그 사이 값들이 메모리에 buffer
```

짝이 멀어지면 메모리 폭발. 가까이 진행하는 경우만.

### 8.4 generator 안의 try/finally

```python
def gen():
    try:
        yield 1
    finally:
        print("cleanup")

g = gen()
next(g)
del g            # generator GC → finally 실행
```

명시적 close 가 더 안전.

### 8.5 list(generator) 가 메모리 의도와 정반대

```python
data = list(x*2 for x in range(10**7))    # 메모리 폭발
```

generator 의 가치는 *list 로 안 만들 때*.

---

## §9 다른 언어

| 언어 | iterator 모델 |
|---|---|
| **Python** | `__iter__/__next__`, generator 로 생성 |
| **JS** | `[Symbol.iterator]() / .next()`, generator function |
| **Java** | Iterator interface, Stream API (Java 8+) |
| **Rust** | Iterator trait, lazy. 매우 성능 최적. |
| **C#** | IEnumerable, yield return |
| **Go** | range / channel |

Python / JS / C# 의 generator 가 syntax 수준에서 가장 닮음.

---

## §10 학습 포인트 (한 줄 요약)

1. **iterator protocol** : `__iter__` + `__next__` + StopIteration.
2. **generator function = yield** — protocol 자동 구현.
3. **lazy** : 한 번에 하나, 메모리 O(1).
4. **`range/map/filter/zip` 모두 lazy** (3.x).
5. **`itertools`** = 함수형 lazy 도구상자.
6. **무한 시퀀스** = generator 만의 영역.
7. **send/throw/close** — generator 가 coroutine 도 됨.
8. **yield from** = subgenerator 위임.
9. **async generator** — Dishka session / pytest fixture 패턴.
10. **list(gen) 은 lazy 의 가치 무력화**.

---

## 참고

- PEP 255 (generator), PEP 380 (yield from)
- itertools docs : https://docs.python.org/3/library/itertools.html
