# 03 — 메모리 관리 (Reference Counting + Cyclic GC)

> "Python 은 GC 가 알아서 해준다" 는 절반만 진실. CPython 은 *Reference Counting* 으로 99% 의 객체를 즉시 해제하고, *Cyclic GC* 로 순환 참조만 별도 처리한다. 이 글은 두 메커니즘과 그 함정.

---

## §0 두 단계의 메모리 관리

```
객체 생성  ──▶  ob_refcnt = 1
   │
   ▼  여러 곳에서 참조
ob_refcnt 증감 (LOAD/STORE/DEL/스코프 종료마다)
   │
   ▼  ob_refcnt == 0
즉시 해제 (deallocation)

(별도 트랙)
순환 참조가 생기면 (a→b, b→a)  ──▶  refcnt 가 0 안 됨
   │
   ▼  세대별 GC 가 주기적으로 스캔
순환 끊고 해제
```

JVM / V8 의 *trace-and-sweep GC* 만 쓰는 모델과 다르다 — Python 은 **즉시 해제 + 보조 GC**.

---

## §1 Reference Counting

### 1.1 기본

```python
import sys

a = [1, 2, 3]
sys.getrefcount(a)       # 2 — getrefcount 호출 자체가 1 추가
```

- 모든 객체 헤더에 `ob_refcnt` 필드.
- 참조 늘면 +1, 줄면 -1.
- 0 이 되는 순간 *즉시* `__del__` + 메모리 해제.

### 1.2 언제 증가 / 감소

| 동작 | refcnt |
|---|---|
| `x = obj` | +1 |
| `y = x` | +1 (같은 객체) |
| `lst.append(obj)` | +1 |
| `del x` | -1 |
| 함수 끝 (로컬 변수 사라짐) | -1 |
| `lst.pop()` | -1 |
| 컨테이너 자체 해제 | 안 의 모든 참조 -1 |

### 1.3 즉시 해제의 이점

```python
def f():
    f = open("big.dat")
    return f.read(10)
# 함수 끝 → f 의 refcnt 0 → 파일 즉시 close
```

→ JVM 같으면 GC 가 돌 때까지 file descriptor 가 안 풀린다. Python 은 *결정론적*.

단, **명시적 close 가 더 안전** — refcount 의존은 다른 구현 (PyPy 등) 에서 깨짐. ShopTracker 도 `with` / `async with` 사용 (18 장).

---

## §2 Cyclic GC — 순환 참조

### 2.1 문제

```python
a = []
b = []
a.append(b)
b.append(a)

del a
del b
# a, b 의 refcnt 가 각자 1 (서로가 참조) → 0 안 됨 → 메모리 누수
```

Reference counting 은 *순환* 을 못 푼다.

### 2.2 해결 — 세대별 GC

CPython 의 `gc` 모듈이 컨테이너 객체 (list, dict, instance 등) 를 *3 세대* 로 추적.

- **gen 0** : 새 객체. 자주 스캔.
- **gen 1** : gen 0 에서 살아남은 것.
- **gen 2** : 오래 산 객체. 가끔 스캔.

알고리즘 :
1. 모든 컨테이너의 refcnt 를 임시 카피.
2. 컨테이너 안의 참조를 따라가며 카피된 카운트를 줄임.
3. 카운트가 0 이상인 것 = 외부에서 참조됨, 보존.
4. 0 인 것 = 순환만 이루는 것, 해제.

```python
import gc

gc.collect()                     # 수동 트리거
gc.get_count()                   # (gen0, gen1, gen2) 현재 객체 수
gc.get_threshold()               # 트리거 임계값 (700, 10, 10)
```

### 2.3 비활성화

```python
gc.disable()                     # 순환 GC 끔. refcnt 만 동작.
gc.enable()
```

대규모 데이터 로딩 (수백만 객체 생성) 시 GC pause 가 느려서 잠시 끄는 경우 — instagram 같은 대형 서비스 사례.

---

## §3 객체 메모리 구조

```c
typedef struct {
    Py_ssize_t ob_refcnt;       // 참조 카운트
    PyTypeObject *ob_type;      // 타입 포인터
    // ... (타입별 추가 필드)
} PyObject;
```

모든 객체가 이 헤더로 시작 — *전부 동일 인터페이스*.

```python
import sys

sys.getsizeof(0)         # 28 (CPython 3.12, int 객체)
sys.getsizeof("")        # 49
sys.getsizeof([])        # 56
sys.getsizeof({})        # 64
```

→ Python 객체는 *기본 비용* 이 큼. 작은 정수 백만 개 = 28MB+. 그래서 NumPy 같은 native array 가 메모리 효율적.

---

## §4 Object Caching / Interning

CPython 은 자주 쓰이는 객체를 캐시.

### 4.1 Small int

```python
a = 256
b = 256
a is b              # True - 캐시됨

a = 257
b = 257
a is b              # False - 새 객체
```

[-5, 256] 정수는 인터프리터 시작 시 미리 생성. 모든 같은 값이 같은 객체.

### 4.2 String interning

```python
a = "hello"
b = "hello"
a is b              # True - 식별자 같은 짧은 문자열은 자동 intern

a = "hello world!"
b = "hello world!"
a is b              # 구현 의존 (보통 True for short, False for long)

import sys
a = sys.intern("dynamic " + "string")
b = sys.intern("dynamic " + "string")
a is b              # True - 명시적 intern
```

식별자처럼 보이는 문자열 (영문/숫자/_) 은 자동 intern. 그렇지 않으면 명시적 `sys.intern` 필요. dict 키로 자주 쓸 때 메모리 + 속도 ↑.

### 4.3 None / True / False / Ellipsis

singleton — 항상 같은 객체.

```python
None is None            # True (definition)
```

---

## §5 메모리 누수의 출처

### 5.1 순환 참조 + `__del__`

```python
class A:
    def __del__(self): print("del")

a = A(); b = A()
a.b = b; b.a = a
del a, b
# 이전 (3.4 미만) : __del__ 가 있으면 GC 가 못 풀어서 누수
# 3.4+ : PEP 442 — __del__ 도 처리. 단 __del__ 호출 순서 비결정.
```

→ `__del__` 신뢰 X. `with` 또는 명시적 cleanup.

### 5.2 closure / decorator 가 큰 객체 캡처

```python
def make_handler():
    big = load_huge()
    def handler():
        print(len(big))
    return handler

h = make_handler()        # big 이 h 의 closure 에 살아 있음
```

→ 핸들러를 어딘가에 저장하면 big 도 함께 살아 있음. weakref 로 해결 가능.

### 5.3 모듈 레벨 캐시

```python
_cache = {}
def f(x):
    _cache[x] = expensive(x)
```

모듈은 프로세스 종료까지 살아 있음 → 캐시 무한 증가. `lru_cache(maxsize=...)` 또는 명시적 만료.

### 5.4 이벤트 핸들러 / observer

```python
event_bus.subscribe(MyEvent, self.handle)   # self 가 bus 에 잡힘
```

bus 가 self 를 strong reference 로 잡으면 self 가 영원히 살아 있음. → `weakref.WeakMethod`.

### 5.5 thread / task 누수

```python
asyncio.create_task(f())     # task 자체가 살아 있는 동안 f 의 closure 도 산다
```

task 끝났는지 추적 (`done()` 콜백, TaskGroup).

---

## §6 메모리 디버깅 도구

```python
import tracemalloc

tracemalloc.start()
# ... 코드 실행 ...
snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics("lineno")[:10]:
    print(stat)
```

→ "어느 라인이 가장 많은 메모리 차지하는지" 확인.

다른 도구 :
- `objgraph` — 객체 참조 그래프 시각화.
- `pympler` — 객체별 메모리 분석.
- `memory_profiler` — 라인별 메모리.
- `gc.get_objects()` — 살아있는 모든 객체.

---

## §7 함정

### 7.1 sys.getsizeof 의 함정

```python
sys.getsizeof([1, 2, 3])    # 88 — 리스트 자체만, 안의 int 는 별도
```

*컨테이너의 메모리* 는 안의 객체 메모리를 *포함하지 않음*. 진짜 사용량 보려면 `pympler.asizeof`.

### 7.2 generator 의 메모리

```python
data = [x*2 for x in range(10**7)]   # 즉시 모든 값 메모리에
data = (x*2 for x in range(10**7))   # generator — 한 번에 하나씩
```

대규모 처리는 generator (07 장).

### 7.3 dict / set 의 비효율

```python
d = {i: i for i in range(100)}
sys.getsizeof(d)            # 4700+ bytes
```

dict 는 hash table — load factor 유지를 위해 실제 용량보다 크게 잡음.

### 7.4 GC 가 멀티스레드 안전?

GIL 덕에 yes. PEP 703 의 free-threaded 빌드는 GC 도 fine-grained lock.

---

## §8 다른 언어 비교

| 언어 | 모델 |
|---|---|
| **Python (CPython)** | RC + 보조 cyclic GC |
| **JVM** | 세대별 + concurrent GC (G1, ZGC) |
| **V8 (JS)** | 세대별 mark-and-sweep |
| **Go** | concurrent tri-color mark-and-sweep |
| **Rust** | RAII (compile-time) + 명시적 Rc/Arc |
| **C++** | manual / RAII |

Python 의 RC 는 *결정론적 해제* + *low pause* 특징. 단점은 순환 처리 비용.

---

## §10 학습 포인트 (한 줄 요약)

1. **Reference counting + Cyclic GC** 의 두 단계.
2. **refcnt == 0 → 즉시 해제** — 결정론적.
3. **순환 참조** = RC 가 못 풀음 → 보조 GC.
4. **세대별 (gen 0/1/2)** — 오래 산 객체는 가끔.
5. **객체 헤더에 ob_refcnt + ob_type** — 모든 객체 동일.
6. **small int / 짧은 str interning** — `is` 가 우연히 True.
7. **`__del__` 신뢰 X** — `with` 사용.
8. **closure / 모듈 캐시 / observer** = 흔한 누수 출처.
9. **tracemalloc / objgraph** — 디버깅 도구.
10. **PEP 703 free-threaded** — RC 와 GC 가 모두 변할 예정.

---

## 참고

- Python docs : `gc` 모듈
- "Inside CPython memory management" — 공식 dev guide
- Instagram 의 GC 비활성화 사례
