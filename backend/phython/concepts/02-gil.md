# 02 — GIL (Global Interpreter Lock)

> "Python 은 멀티스레드로 CPU 를 못 쓴다" 의 정체. CPython 의 *한 프로세스 안에서는 한 번에 하나의 스레드만 바이트코드를 실행할 수 있다* 는 락. 이 글은 *왜 GIL 이 존재하고, 무엇이 막히고 무엇은 안 막히고, 어떻게 우회하는가* 를 정리.

---

## §0 정의

> GIL = CPython 인터프리터의 mutex. *한 프로세스 안에서 한 번에 한 스레드만* Python 바이트코드를 실행 가능.

OS 스레드 자체는 N 개 만들 수 있지만, *Python 바이트코드 실행 권한* 은 한 번에 한 스레드.

---

## §1 왜 존재하나 — 단순함의 대가

### 1.1 CPython 의 객체 메모리 관리

모든 객체에 `ob_refcnt` (참조 카운트) 가 있다. `x = obj; y = obj` → refcnt += 2.

멀티스레드에서 두 스레드가 동시에 같은 객체의 refcnt 를 증감하면 :
- race condition → 잘못된 카운트 → 메모리 누수 또는 use-after-free.

**해결 옵션 A — 모든 refcnt 를 atomic** : 매 연산이 lock/CAS → 싱글스레드 성능 30~50% 손실.

**해결 옵션 B — 인터프리터 전체에 글로벌 락 (GIL)** : 싱글스레드는 그대로 빠름. 멀티스레드만 직렬화.

CPython 은 1992 부터 옵션 B 채택. 이후 Python 의 모든 C 확장이 "GIL 보유" 가정으로 작성됨 → 빼기 어려움.

### 1.2 PEP 703 — Free-Threaded Python (3.13+)

Python 3.13 부터 *no-GIL 빌드* 옵션 도입 (실험적). PEP 703 의 결과. 3.14, 3.15 에 정식화 예정.

핵심 변화 :
- refcnt 를 biased reference counting (스레드별 로컬 + global) 으로.
- `dict`, `list` 등에 fine-grained lock.
- 단일스레드 성능 ~10% 손실 (수용 범위).
- 모든 C 확장 호환 작업 진행 중 (NumPy 등).

→ ShopTracker 시점 (3.12) 은 여전히 GIL 있음. 단, *몇 년 안에* 사라질 가능성.

---

## §2 GIL 이 막는 것 / 안 막는 것

### 2.1 막힌다 (CPU bound)

```python
import threading

def cpu_heavy():
    n = 0
    for _ in range(10**7):
        n += 1

t1 = threading.Thread(target=cpu_heavy)
t2 = threading.Thread(target=cpu_heavy)
t1.start(); t2.start(); t1.join(); t2.join()
# 싱글스레드와 거의 동일한 시간 (or 더 느림 — context switch 오버헤드)
```

→ CPU 만 쓰는 작업은 멀티스레드 무력.

### 2.2 안 막힌다 (I/O bound)

```python
import threading, requests

def fetch(url):
    requests.get(url)            # 네트워크 대기 중에는 GIL 놓음

threads = [threading.Thread(target=fetch, args=(url,)) for url in urls]
for t in threads: t.start()
for t in threads: t.join()
# 병렬에 가까움
```

C 코드 안에서 `Py_BEGIN_ALLOW_THREADS` 매크로로 GIL 을 놓을 수 있다. socket / file I/O / sleep / NumPy 의 무거운 연산 등이 그렇게 되어 있음.

→ I/O bound + threading = 정상 작동.

### 2.3 안 막힌다 (NumPy 등 native)

NumPy 의 `np.dot(a, b)` 같은 연산은 BLAS 호출 — 그동안 GIL 놓음. 멀티스레드 활용 가능.

### 2.4 시간 분할

CPython 은 *일정 간격* 으로 GIL 을 놓고 다른 스레드에 양보 :
- 3.2 이전 : 100 바이트코드마다.
- 3.2+ : 5ms 마다 (`sys.setswitchinterval()`).

```python
import sys
sys.setswitchinterval(0.005)     # 기본 5ms
```

너무 짧으면 context switch 오버헤드, 너무 길면 응답성 ↓.

---

## §3 우회 — 멀티프로세스

```python
from multiprocessing import Pool

def cpu_heavy(n):
    return sum(range(n))

with Pool(4) as p:
    results = p.map(cpu_heavy, [10**7] * 4)
# 진짜 4 코어 활용
```

각 프로세스 = 자기 GIL → 진정한 병렬.

비용 :
- 프로세스 생성 비용 (스레드보다 훨씬 무거움).
- 메모리 공유 X — 데이터를 직렬화 (pickle) 로 주고받음.
- IPC 오버헤드.

권장 : *큰 작업 단위* 일 때만. 작은 작업 N 만 개를 process pool 로 보내면 IPC 비용 > 계산 이득.

---

## §4 우회 — C 확장 / Cython / Numba

```python
# Cython 예
cimport cython
from cython.parallel import prange

@cython.boundscheck(False)
def parallel_sum(double[:] arr):
    cdef double total = 0
    cdef int i
    for i in prange(arr.shape[0], nogil=True):    # ← GIL 놓음
        total += arr[i]
    return total
```

핵심 : *GIL 을 놓는 영역* 을 명시. 그 안에서 진짜 멀티스레드.

NumPy / pandas / Polars 같은 라이브러리는 내부적으로 이렇게 되어 있어서 *대규모 데이터 병렬* 이 자연스럽게 됨.

---

## §5 우회 — asyncio

```python
import asyncio, aiohttp

async def fetch(session, url):
    async with session.get(url) as r:
        return await r.text()

async def main():
    async with aiohttp.ClientSession() as session:
        results = await asyncio.gather(*[fetch(session, u) for u in urls])
```

GIL 무관 — 단일 스레드 + 이벤트 루프. I/O bound 에 최적.

ShopTracker 가 asyncio 채택한 이유 (16 장).

---

## §6 GIL 의 미묘한 동작

### 6.1 atomic 처럼 보이는 연산

```python
counter = 0

def inc():
    global counter
    for _ in range(10**6):
        counter += 1
```

`counter += 1` 은 *3 단계 바이트코드* : LOAD → ADD → STORE. GIL 이 사이에 양보할 수 있어 **race 발생**.

```python
threads = [threading.Thread(target=inc) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)
# 4_000_000 이 아닌 3_xxx_xxx — race 로 손실
```

→ 멀티스레드 + 공유 상태 = `threading.Lock` 필수.

### 6.2 dict / list 의 단일 연산은 GIL 보호

`d[k] = v`, `lst.append(x)` 같은 *단일 메서드 호출* 은 GIL 덕에 atomic. 단 *두 단계* 는 X :

```python
if k not in d:           # ← 여기와
    d[k] = v             # ← 여기 사이에 다른 스레드 끼어듦 가능
```

→ `defaultdict` 또는 lock.

### 6.3 GIL 은 sleep/I/O 에서 양보

```python
import time
def f():
    time.sleep(1)        # 그동안 다른 스레드 실행
```

`time.sleep` 의 C 구현은 `Py_BEGIN_ALLOW_THREADS` 안에서 sleep — GIL 놓음.

---

## §7 GIL 측정 / 디버깅

```python
import sys
sys.getswitchinterval()         # 기본 0.005

# GIL 보유 여부 (3.13+)
sys._is_gil_enabled()
```

도구 :
- `py-spy` — 프로파일러. GIL contention 시각화.
- `viztracer` — 스레드별 trace.

---

## §8 자주 하는 오해

| 오해 | 실제 |
|---|---|
| Python 은 멀티스레드를 못 쓴다 | I/O bound + native code 는 잘 씀 |
| GIL 때문에 asyncio 가 필요하다 | asyncio 와 GIL 은 다른 문제 (asyncio 는 단일 스레드 동시성) |
| 모든 Python 구현에 GIL 있다 | Jython / IronPython 은 없음. 3.13+ 의 free-threaded 빌드도 없음 |
| 멀티프로세스는 항상 빠르다 | IPC 비용이 계산 이득 넘으면 더 느림 |
| GIL 이 곧 사라진다 | PEP 703 진행 중이지만 수년 (3.14~3.15+) |

---

## §9 결정 가이드

| 상황 | 선택 |
|---|---|
| I/O bound (HTTP / DB / 파일) | asyncio (대규모) 또는 threading (수십 ~ 수백) |
| CPU bound (계산) | multiprocessing 또는 native (NumPy/Cython) |
| CPU bound + 큰 데이터 공유 | shared_memory + multiprocessing, 또는 native |
| 혼합 | asyncio + `run_in_executor` (CPU 작업을 process pool 로) |

ShopTracker 는 I/O bound 위주 (FastAPI + DB) → asyncio 일관 채택.

---

## §10 학습 포인트 (한 줄 요약)

1. **GIL = 한 프로세스에 한 스레드만 바이트코드 실행**.
2. **이유** = refcnt 보호의 단순한 해법. 단일스레드 성능 보존.
3. **막는 것** : CPU bound 멀티스레드.
4. **안 막는 것** : I/O bound + native (NumPy, BLAS).
5. **우회** : multiprocessing, native (Cython/Numba), asyncio.
6. **PEP 703 (3.13+)** : free-threaded 빌드 실험.
7. **`x += 1` 도 race** — 멀티스레드 공유 상태 시 Lock 필수.
8. **dict/list 단일 메서드는 atomic** , 두 단계는 X.
9. **`sys.setswitchinterval`** 으로 양보 간격 조절.
10. **결정** : I/O = async, CPU = process / native.

---

## 참고

- PEP 703 (Free-Threaded CPython) : https://peps.python.org/pep-0703/
- David Beazley, "Understanding the Python GIL" 강연
- py-spy : https://github.com/benfred/py-spy
