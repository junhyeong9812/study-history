# asyncio 기본기 — Task/Future/Event/Lock

> 운영 코드에서 자주 쓰는 asyncio 핵심 5가지를 한 자리에 정리.

---

## 1. asyncio 의 기본 모델

```
event loop
  ├── ready queue   (실행 대기 중인 코루틴)
  └── waiting       (await 중인 작업: I/O, Future, sleep)
```

- 코루틴(`async def`)은 호출만으로는 실행 안 됨 — `await` 또는 `asyncio.create_task` 로 루프에 등록되어야 함.
- 단일 스레드에서 협력적 멀티태스킹 → `await` 가 곧 양보 시점.
- `time.sleep`, 동기 DB 호출 같은 블로킹 코드는 루프 전체를 막음 → 절대 금지.

---

## 2. `asyncio.create_task` — 백그라운드 실행

```python
import asyncio

async def worker(name):
    await asyncio.sleep(1)
    print(f"{name} done")

async def main():
    task = asyncio.create_task(worker("A"))
    print("A scheduled")
    await task  # 결과 받고 싶으면
```

핵심:
- `create_task` 호출 즉시 워커가 루프에 등록되어 백그라운드에서 진행.
- task 객체에 강한 참조가 없으면 GC 가 도중 수거할 수 있음 → 셋에 보관 권장.
- task 안 예외는 await 또는 `add_done_callback` 으로 챙기지 않으면 로그 한 줄만 나오고 묻힘.

### 강한 참조 패턴
```python
_bg_tasks: set[asyncio.Task] = set()

def fire_and_forget(coro):
    t = asyncio.create_task(coro)
    _bg_tasks.add(t)
    t.add_done_callback(_bg_tasks.discard)
    return t
```

---

## 3. `asyncio.gather` vs `asyncio.wait` vs `as_completed`

```python
# gather: 모든 결과를 입력 순서대로
results = await asyncio.gather(coro1, coro2, coro3)

# gather + return_exceptions: 일부 실패해도 진행
results = await asyncio.gather(*coros, return_exceptions=True)
# results[i] 가 Exception 인스턴스일 수 있음

# wait: 조건부
done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)

# as_completed: 완료 순서대로
for coro in asyncio.as_completed(coros):
    result = await coro
```

| API | 결과 순서 | 조기 종료 | 권장 |
|-----|----------|---------|-----|
| `gather` | 입력 순서 | 첫 예외 시 다른 task 도 cancel (return_exceptions=False) | 결과 묶어서 처리 |
| `wait` | 별도 처리 | 첫 완료 시 즉시 반환 가능 | "첫 응답 채택" 패턴 |
| `as_completed` | 완료 순서 | - | 진행률 출력 |

### gather 의 함정
- `return_exceptions=False` (기본) + 한 task 실패 → 나머지 task **cancel** 됨. 그 cancel 도중 추가 예외 가능.
- DB 트랜잭션 같은 부수효과 있는 task 면 부분완료 상태 발생.

---

## 4. `asyncio.Event` — 신호 기반 종료

장기 실행 루프의 **그레이스풀 종료** 패턴:

```python
_stop_event = asyncio.Event()

async def loop_worker():
    while not _stop_event.is_set():
        try:
            await do_work()
        except Exception as e:
            logger.error(f"work failed: {e}")
        try:
            await asyncio.wait_for(_stop_event.wait(), timeout=10.0)
        except asyncio.TimeoutError:
            pass  # 10초 경과 → 다음 반복

async def stop():
    _stop_event.set()
```

### 왜 `asyncio.sleep(10)` 이 아니고 `wait_for(event.wait(), 10)`?
- `sleep(10)` 만 쓰면 stop 호출 후 최대 10초 기다려야 루프가 깸.
- `wait_for(event.wait(), 10)` 는 **이벤트 set 즉시 깨거나 10초 경과 시 깸**.

---

## 5. `asyncio.Lock`/`Semaphore`

### Lock — 임계 영역
```python
_lock = asyncio.Lock()

async def critical():
    async with _lock:
        await db_write()
```

→ 동시에 한 코루틴만 진입. async 친화 (블로킹 안 함).

### Semaphore — 동시 실행 제한
```python
_sem = asyncio.Semaphore(10)

async def fetch(url):
    async with _sem:
        return await httpx_get(url)

# 1000개 동시 호출해도 항상 10개씩만 진행
results = await asyncio.gather(*[fetch(u) for u in urls])
```

→ 외부 API 의 rate limit 회피, 동시 연결 수 제한.

---

## 6. `asyncio.Queue`

```python
queue: asyncio.Queue = asyncio.Queue(maxsize=100)

async def producer():
    for item in source:
        await queue.put(item)  # 가득 차면 대기
    await queue.put(None)  # sentinel

async def consumer():
    while True:
        item = await queue.get()
        if item is None:
            break
        await process(item)
        queue.task_done()

await asyncio.gather(producer(), consumer())
```

핵심:
- `maxsize` 로 백프레셔 — 빠른 생산자가 메모리 폭주 방지.
- sentinel(None) 또는 `queue.join()` + `task_done()` 으로 종료 동기화.
- 멀티 컨슈머 패턴에 자연스러움 (한 큐에 여러 consumer task).

---

## 7. 흔한 안티패턴

### 7.1 동기 블로킹
```python
async def bad():
    time.sleep(5)             # 루프 전체 정지 5초
    requests.get("...")       # 동기 HTTP — 루프 정지
```

→ `await asyncio.sleep`, `httpx.AsyncClient`, `aiomysql` 등 async 친화 라이브러리 사용.
→ 어쩔 수 없이 동기 함수 호출해야 하면 `await asyncio.to_thread(fn, *args)`.

### 7.2 forgotten await
```python
async def fetch():
    return await client.get("...")

async def use():
    result = fetch()  # ← await 안 함. result 는 코루틴 객체.
```

→ Python 종료 시 `RuntimeWarning: coroutine was never awaited`.

### 7.3 동기 함수 안에서 코루틴 호출
```python
def sync_handler():
    result = await fetch()  # SyntaxError: await 는 async 함수 안에서만
```

→ 동기 컨텍스트에서 코루틴 실행하려면 `asyncio.run(fetch())` (다만 이미 루프 안이면 에러).

### 7.4 `asyncio.run` 중첩
```python
async def outer():
    asyncio.run(inner())  # ← 이미 루프 있는데 또 run → RuntimeError
```

→ 이미 루프 안이면 `await inner()` 또는 `asyncio.create_task(inner())`.

---

## 8. 운영 팁

### 디버깅 모드
```python
asyncio.run(main(), debug=True)
# 또는 환경변수
PYTHONASYNCIODEBUG=1 python app.py
```

→ slow callback 경고, 미await 코루틴 경고 등 증가.

### 이벤트 루프 정책
- Python 3.10+ 권장: `asyncio.run` 으로 시작.
- uvloop: 더 빠른 루프 구현. `asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())`. 단 멀티프로세싱 + spawn 조합에서 일부 이슈 가능 → 명시적으로 `asyncio` 사용하는 코드도 있음.

### Task 모니터링
```python
all_tasks = asyncio.all_tasks()
for t in all_tasks:
    print(t.get_name(), t.get_coro())
```

→ 운영 중 메모리 누수가 의심되면 task 수가 폭증하는지 확인.

---

## 9. 응용 포인트

- 백그라운드 폴링 → `asyncio.create_task` + `asyncio.Event` + `wait_for`.
- 외부 API 부하 제어 → `asyncio.Semaphore`.
- 생산자/소비자 → `asyncio.Queue` + maxsize 백프레셔.
- 여러 작업 병렬 + 결과 묶기 → `asyncio.gather` (실패 격리는 `return_exceptions=True`).
- 동기 라이브러리 호출 회피 → `asyncio.to_thread`.
