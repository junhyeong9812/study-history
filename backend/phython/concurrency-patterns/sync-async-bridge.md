# 동기 ↔ 비동기 경계 — `run_in_executor`, `to_thread`, 반대 방향

> "DB 라이브러리가 동기인데 FastAPI 라우터는 async" 같은 상황을 안전하게 풀기.

---

## 0. 분석 대상 코드

마이그레이션 코드의 동기 DB 호출을 async 컨텍스트로 옮기는 부분.

```python
# example-app/src/migration/migration.py
import asyncio

class TrademarkMigration:
    async def _get_total_count_async(self, updated_after=None) -> int:
        """전체 건수 조회"""
        try:
            loop = asyncio.get_event_loop()
            count = await loop.run_in_executor(
                None,                        # 기본 ThreadPoolExecutor
                biblio_repo.get_total_count, # 동기 함수
                updated_after,               # 인자
            )
            logger.debug(f"Total count from DB: {count}")
            return count
        except Exception as e:
            logger.error(f"Failed to get total count: {str(e)}", exc_info=True)
            raise

    async def _fetch_application_numbers_async(self, offset, limit, updated_after=None) -> list:
        """출원번호 배치 조회"""
        try:
            loop = asyncio.get_event_loop()
            numbers = await loop.run_in_executor(
                None,
                biblio_repo.get_all_application_numbers,
                offset,
                limit,
                updated_after,
            )
            return numbers
        except Exception as e:
            logger.error(f"Failed to fetch: {str(e)}", exc_info=True)
            raise
```

---

## 1. 왜 경계가 필요한가

- FastAPI 라우터가 `async def` → 핸들러는 이벤트 루프에서 실행.
- 그 안에서 동기 DB 라이브러리(예: 동기 SQLAlchemy, pymysql) 를 직접 호출하면 **루프 전체가 멈춤**.
- 동기 함수는 별도 스레드에서 실행해야 함 → `run_in_executor` 또는 `to_thread`.

---

## 2. `loop.run_in_executor(executor, fn, *args)`

### 2.1 시그니처 및 동작

```python
loop = asyncio.get_event_loop()
result = await loop.run_in_executor(executor, fn, arg1, arg2)
```

- `executor=None` → 기본 ThreadPoolExecutor (loop.set_default_executor 으로 변경 가능).
- `fn` 은 동기 함수.
- `*args` 는 위치 인자 (kwargs 직접 못 넘김 → `functools.partial` 사용).

```python
from functools import partial
result = await loop.run_in_executor(
    None,
    partial(my_sync_fn, x=1, y=2)
)
```

### 2.2 ProcessPoolExecutor 와 결합

```python
from concurrent.futures import ProcessPoolExecutor

executor = ProcessPoolExecutor(max_workers=4)
result = await loop.run_in_executor(executor, cpu_heavy_fn, data)
```

→ CPU bound 함수를 별도 프로세스로. (ProcessPool 은 인자/반환값 pickle 가능해야 함.)

---

## 3. `asyncio.to_thread(fn, *args, **kwargs)` — 신권장

Python 3.9+ 에서 `to_thread` 가 표준 권장:

```python
result = await asyncio.to_thread(my_sync_fn, x, y, key=value)
```

차이:
- kwargs 지원.
- 짧고 읽기 쉬움.
- 내부적으로 `loop.run_in_executor(None, ...)` 와 동일.

→ 새 코드는 `to_thread` 권장. 위 운영 코드는 3.9 이전부터 진화한 흔적.

---

## 4. 반대 방향 — 동기 → 비동기

동기 함수 안에서 비동기 코루틴을 실행해야 할 때.

### 4.1 새 이벤트 루프 시작
```python
def sync_caller():
    result = asyncio.run(async_fn())
    return result
```
- `asyncio.run` 은 루프를 새로 만들고 끝나면 닫음.
- **이미 루프가 도는 컨텍스트(스레드 안)** 에서 호출하면 RuntimeError.

### 4.2 ProcessPool 워커 안에서 async 호출
ProcessPool 워커는 자체 인터프리터 → 자체 이벤트 루프 가능:
```python
def process_batch_worker(args):
    return asyncio.run(_async_inner(args))

async def _async_inner(args):
    async with httpx.AsyncClient() as c:
        return await c.get(...)
```

### 4.3 쓰레드에서 부모 루프로 결과 전달
```python
def in_thread(loop):
    fut = asyncio.run_coroutine_threadsafe(async_fn(), loop)
    return fut.result(timeout=30)
```

→ 다른 스레드에서 메인 루프에 코루틴 제출.

---

## 5. 안티패턴 모음

### 5.1 async 함수 안에서 동기 블로킹
```python
async def handler():
    requests.get(url)  # ← 루프 정지
```
→ `httpx.AsyncClient` 또는 `to_thread(requests.get, url)`.

### 5.2 동기 컨텍스트에서 await
```python
def sync_caller():
    result = await async_fn()  # SyntaxError
```
→ `asyncio.run` 또는 `run_coroutine_threadsafe`.

### 5.3 실행 중인 루프에서 `asyncio.run`
```python
async def outer():
    asyncio.run(inner())  # RuntimeError: cannot be called from a running event loop
```
→ `await inner()` 또는 `create_task(inner())`.

### 5.4 ThreadPool 의 무한 큐
- `run_in_executor(None, ...)` 는 디폴트 풀 사용 → 동시에 많은 동기 호출 시 큐가 폭증.
- 외부 API 호출은 별도 풀 + `asyncio.Semaphore` 로 제한.

---

## 6. 트랜잭션과 세션 — 자주 하는 실수

```python
# 잘못된 예
async def update_user(uid):
    user = await loop.run_in_executor(None, sync_db.get_user, uid)
    user.name = "X"
    await loop.run_in_executor(None, sync_db.save, user)  # ← 다른 스레드 = 다른 세션
```

각 `run_in_executor` 호출은 **다른 스레드**일 수 있음 → 세션이 스레드 로컬이면 다른 세션 → 트랜잭션 분리.

해결:
- 한 스레드 안에서 트랜잭션 단위를 한 번에 처리:
  ```python
  def _do_update(uid):
      with sync_db.session() as s:
          user = s.get(User, uid)
          user.name = "X"
          s.commit()
  await loop.run_in_executor(None, _do_update, uid)
  ```

---

## 7. 위 코드의 트레이드오프 분석

```python
async def _get_total_count_async(self, updated_after=None) -> int:
    loop = asyncio.get_event_loop()
    count = await loop.run_in_executor(None, biblio_repo.get_total_count, updated_after)
    return count
```

**좋은 점**:
- 동기 repo 인터페이스를 그대로 두고 async 라우터에서 사용 가능.
- 단순.

**제약**:
- 디폴트 ThreadPool 은 보통 코어수 × 5 (Python 3.8+ 는 max(32, cpu*5)).
- DB 호출이 많아지면 풀 큐 적체 가능.
- async DB 드라이버(asyncmy, aiomysql) 로 가는 게 더 효율적이지만 마이그레이션 비용 큼.

**이 프로젝트는** 마이그레이션이 일회성 + 동기 DB 풀이 잘 동작하는 상황이라 ThreadPool 우회가 합리적 선택.

---

## 8. 응용 포인트

- 동기 함수를 async 안에서 호출: `await asyncio.to_thread(fn, *args, **kwargs)` (3.9+) 또는 `loop.run_in_executor(None, fn, *args)`.
- ProcessPool 결합 시 `loop.run_in_executor(process_pool, fn, args)`.
- 동기 컨텍스트에서 코루틴 실행: `asyncio.run` (루프 없을 때) 또는 `run_coroutine_threadsafe` (다른 스레드).
- 트랜잭션은 한 스레드 안에서 한 번에 — 여러 `run_in_executor` 로 쪼개면 세션 분리.
- 외부 호출 많으면 Semaphore + 별도 ThreadPoolExecutor 로 큐 적체 방지.
