# 11 — 동시성 모델 비교 (threading vs multiprocessing vs asyncio)

> "동시성" 한 단어 안에 *진짜 병렬 (parallelism)*, *I/O 다중화 (concurrency)*, *CPU 병렬* 이 다 들어 있다. Python 은 GIL 때문에 이 셋이 명확히 갈린다. 이 글은 셋의 모델 / 적합 영역 / 결정 가이드.

---

## §0 세 모델 한눈에

| | threading | multiprocessing | asyncio |
|---|---|---|---|
| 진짜 병렬 | X (GIL) | O | X |
| 메모리 | 공유 | 분리 (IPC) | 공유 (단일 스레드) |
| context switch | OS | OS + IPC | 이벤트 루프 |
| 적합 | I/O bound (수백) | CPU bound | I/O bound (수만) |
| 비용 | 스레드당 MB | 프로세스당 더 무거움 | task 당 KB |
| 동기화 | Lock / Queue / Event | Queue / Pipe / Manager | Lock / Queue (asyncio) |
| 라이브러리 호환 | 모든 sync | 모든 sync | async 라이브러리 필요 |

---

## §1 threading

```python
import threading

def worker(n):
    for _ in range(n):
        ...

threads = [threading.Thread(target=worker, args=(1000,)) for _ in range(10)]
for t in threads: t.start()
for t in threads: t.join()
```

### 1.1 GIL 영향

CPU bound 작업 = 직렬화 (02 장). I/O bound = 진짜 동시.

### 1.2 동기화 도구

```python
import threading

lock = threading.Lock()
with lock:
    critical_section()

cond = threading.Condition()
event = threading.Event()
sem = threading.Semaphore(5)
queue = queue.Queue()
```

### 1.3 ThreadPoolExecutor

```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=10) as ex:
    futures = [ex.submit(fetch, url) for url in urls]
    results = [f.result() for f in futures]
```

수동 thread 관리보다 추천.

---

## §2 multiprocessing

```python
from multiprocessing import Pool

def cpu_heavy(n):
    return sum(range(n))

with Pool(4) as p:
    results = p.map(cpu_heavy, [10**7] * 4)
```

### 2.1 데이터 공유

```python
from multiprocessing import Value, Array, Manager

# 단순 값
counter = Value('i', 0)
counter.value += 1

# 컨테이너 (느림)
manager = Manager()
shared_dict = manager.dict()

# shared memory (3.8+, 빠름)
from multiprocessing.shared_memory import SharedMemory
shm = SharedMemory(create=True, size=1024)
```

### 2.2 fork vs spawn

```python
import multiprocessing as mp
mp.set_start_method('spawn')   # 모든 OS 통일 (안전)
```

- **fork** (Linux 기본) : 빠름, 메모리 공유 (COW). 단 OS X 에서 deprecated.
- **spawn** : 새 인터프리터. 느리지만 안전.
- **forkserver** : 둘의 절충.

Python 3.14 부터 fork 는 제거 예정.

### 2.3 ProcessPoolExecutor

```python
from concurrent.futures import ProcessPoolExecutor

with ProcessPoolExecutor() as ex:
    results = list(ex.map(cpu_heavy, args))
```

### 2.4 비용

- 프로세스 생성 = OS fork/spawn (수백 ms).
- 데이터는 pickle 로 직렬화 → IPC.
- 대규모 데이터 = pickle 비용 ↑.
- **작은 작업 N 만 개 = process pool 비효율**.

---

## §3 asyncio

```python
import asyncio

async def fetch(url):
    async with aiohttp.ClientSession() as s:
        async with s.get(url) as r:
            return await r.text()

async def main():
    results = await asyncio.gather(*[fetch(u) for u in urls])

asyncio.run(main())
```

### 3.1 모델

- **단일 스레드 + 이벤트 루프**.
- task = coroutine 의 실행 단위. KB 단위 메모리.
- I/O 대기 = 다른 task 로 양보.
- *모든 라이브러리가 async 여야* 효과 있음.

(16 장 자세히)

### 3.2 동기화

```python
lock = asyncio.Lock()
async with lock: ...

queue = asyncio.Queue()
sem = asyncio.Semaphore(10)
event = asyncio.Event()
```

### 3.3 TaskGroup (3.11+)

```python
async with asyncio.TaskGroup() as tg:
    tg.create_task(f())
    tg.create_task(g())
# 둘 다 끝날 때까지 대기. 하나 실패 시 다른 것도 cancel.
```

`gather` 보다 안전 — 예외가 ExceptionGroup 으로 묶여 전파.

---

## §4 결정 가이드

### 4.1 I/O bound

| 동시성 정도 | 추천 |
|---|---|
| ~10 | sync + thread pool |
| ~100 | thread pool |
| ~1000 | asyncio |
| ~10000+ | asyncio |

ShopTracker = asyncio (FastAPI + DB 가 모두 async).

### 4.2 CPU bound

| 작업 단위 | 추천 |
|---|---|
| 작고 많음 | NumPy / Cython (native) |
| 크고 적음 | multiprocessing |
| 수치 계산 | NumPy / Numba |
| 외부 라이브러리 | 그 라이브러리의 native 활용 |

### 4.3 혼합

```python
loop = asyncio.get_event_loop()
result = await loop.run_in_executor(
    ProcessPoolExecutor(),
    heavy_compute, data,
)
```

asyncio 를 메인으로, CPU 작업만 process pool 로 위임.

---

## §5 흔한 함정

### 5.1 sync 라이브러리를 async 함수에서

```python
async def f():
    requests.get(url)        # blocking - 이벤트 루프 멈춤!
```

→ `aiohttp`, `httpx`, `asyncpg`. 정 sync 써야 하면 :

```python
await asyncio.to_thread(requests.get, url)
```

### 5.2 Lock 을 await 안에서 너무 길게

```python
async with lock:
    await long_operation()    # 다른 task 가 lock 대기 길어짐
```

→ critical section 최소화.

### 5.3 multiprocessing 의 pickle 실패

```python
def make_local():
    def inner(): ...        # pickle 불가
    return inner

ProcessPoolExecutor().submit(make_local())   # PicklingError
```

→ top-level 함수만 멀티프로세스에 보낼 수 있음. lambda / 클로저 X.

### 5.4 thread + fork 위험

```python
import threading, os
t = threading.Thread(target=...)
t.start()
os.fork()                   # ← deadlock 가능 (libc 락이 frozen)
```

→ fork 전에 thread 만들지 말기. 또는 spawn.

### 5.5 asyncio.run 중첩

```python
async def f():
    asyncio.run(g())        # RuntimeError - already running loop
```

→ `await g()`.

### 5.6 GIL 가정 깨짐 (3.13+ free-threaded)

PEP 703 빌드에서는 dict/list 의 race condition 등이 다르게 작동. 미래 대비.

---

## §6 ShopTracker 의 선택

ShopTracker = **순수 asyncio**.

- FastAPI 가 async.
- SQLAlchemy 가 async.
- EventBus 가 async.
- 모든 핸들러가 async.

이유 :
- 웹 서버는 I/O bound.
- 단일 스레드라 race 없음 (단, await 사이는 끼어들 수 있음).
- 코드가 일관 (sync/async 혼용 X).

CPU 작업 (예 : 이미지 처리) 이 들어오면 → `asyncio.to_thread` 또는 별도 worker 프로세스.

---

## §7 다른 언어와 비교

| 언어 | 동시성 모델 |
|---|---|
| **Python** | GIL → asyncio (I/O), multiprocessing (CPU) |
| **JS / Node** | 단일 스레드 + 이벤트 루프. CPU 는 worker_threads. |
| **Go** | goroutine — runtime 이 M:N 스케줄. *진짜 병렬 + 사용자는 sync 처럼*. |
| **Rust** | tokio (async) + std::thread. 컴파일 타임 안전. |
| **Java** | thread (heavy) + virtual thread (Project Loom, 21+) — Go 와 유사. |
| **Erlang/Elixir** | actor (process) — 매우 가벼움, 진짜 병렬. |

Python 의 asyncio = JS 모델. Go / Java VT 가 가장 모던.

---

## §8 future 동향

- **PEP 703** (free-threaded) — GIL 제거. 3.13 실험, 3.14~3.15 정식.
- **Subinterpreters** (PEP 684) — 한 프로세스에 여러 인터프리터, 각자 GIL. 3.12+ 부터 점진 도입.
- **JIT (3.13+)** — Tier 2 인터프리터 → JIT 실험.

Python 의 동시성 모델이 *재구성 중*. 5 년 안에 답이 바뀔 수 있음.

---

## §10 학습 포인트 (한 줄 요약)

1. **threading** = I/O bound 수십~수백, GIL 때문에 CPU 무력.
2. **multiprocessing** = CPU bound, IPC 비용 / pickle 제약.
3. **asyncio** = I/O bound 수만, 라이브러리 호환 필수.
4. **ProcessPool / ThreadPoolExecutor** — 수동 관리 대신.
5. **TaskGroup (3.11+)** = 안전한 병렬 task.
6. **`asyncio.to_thread`** = sync 함수를 thread 로 우회.
7. **shared memory (3.8+)** — multiprocessing 의 빠른 데이터 공유.
8. **fork → spawn 전환** — 3.14 부터 강제.
9. **GIL 무관 native (NumPy/Cython)** — CPU 우회의 흔한 길.
10. **ShopTracker = 순수 asyncio** — 웹 서버 = I/O bound.

---

## 참고

- asyncio docs : https://docs.python.org/3/library/asyncio.html
- PEP 703, 684
- Go / Java Virtual Thread 비교 자료
