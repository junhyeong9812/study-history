# ProcessPoolExecutor — CPU-bound 작업의 병렬화

> 수천만 건 데이터 마이그레이션을 다중 프로세스로 처리한 경험.
> "왜 ThreadPool 이 아니라 ProcessPool 인가" + 워커 프로세스 자원 캐시 트릭.

---

## 0. 분석 대상 코드

`run_migration_async` 핵심 부분:

```python
# example-app/src/migration/migration.py
import multiprocessing as mp
from concurrent.futures import ProcessPoolExecutor, as_completed

# 모듈 전역 — 워커 프로세스마다 독립적으로 살아있는 캐시
_worker_resource_cache = {}


def process_batch_worker(args: Tuple) -> Dict:
    """단일 배치를 처리하는 워커 함수."""
    batch_num, batch_applications, total_batches, log_queue = args

    # 각 프로세스에서 독립적으로 모듈 임포트
    from src.migration.biblio import BiblioProcessor
    from src.migration.admin import AdminProcessor
    # ... (10개 프로세서 import)
    from src.utils.search_utils import TrademarkSearchUtils, load_pronunciation_dicts

    process_logger = get_logger(f"migration.worker.{mp.current_process().pid}")

    global _worker_resource_cache

    # 첫 배치에서만 무거운 자원 로드 → 같은 워커에 들어오는 후속 배치들은 캐시 재사용
    if 'processors' not in _worker_resource_cache:
        _worker_resource_cache['processors'] = {
            'biblio': BiblioProcessor(),
            'admin': AdminProcessor(),
            # ...
        }

    if 'transformer' not in _worker_resource_cache:
        process_logger.info("First batch in this worker — loading resources")
        kor_dict, eng_dict = load_pronunciation_dicts()  # MongoDB 풀스캔
        search_utils = TrademarkSearchUtils(kor_dict=kor_dict, eng_dict=eng_dict)
        translation_loader = ProductTranslationLoader()
        translation_loader.load()
        transformer = TrademarkDataTransformer(
            search_utils=search_utils,
            translation_loader=translation_loader
        )
        _worker_resource_cache['transformer'] = transformer

    transformer = _worker_resource_cache['transformer']

    # 데이터 fetch + transform + ES bulk index
    # ...
    return {
        "batch_num": batch_num,
        "processed": ...,
        "success": ...,
        "failed": ...,
        "success_ids": [...],
        "failed_ids": [...],
    }


async def run_migration_async(self, ..., batch_size=1000, resume=False):
    # 배치 작업 준비
    batch_tasks = []
    for batch_index in range(start_batch_index, total_batches):
        batch_applications = self.checkpoint_manager.get_batch_from_file(batch_index, batch_size)
        if not batch_applications:
            break
        batch_tasks.append((batch_index + 1, batch_applications, total_batches, log_q))

    # 프로세스 풀로 병렬 처리
    with ProcessPoolExecutor(max_workers=current_max_workers) as executor:
        future_to_batch = {
            executor.submit(process_batch_worker, task): task[0]
            for task in batch_tasks
        }

        for future in as_completed(future_to_batch):
            batch_num = future_to_batch[future]
            try:
                result = future.result(timeout=300)  # 5분 타임아웃
                completed_batches += 1
                total_processed += result['processed']
                total_success += result['success']
                total_failed += result['failed']

                self.write_results_buffered(
                    result['success_ids'], result['failed_ids'],
                    success_file, failed_file
                )

                if completed_batches % 10 == 0:
                    self.checkpoint_manager.save_checkpoint(
                        completed_batches, total_success, total_failed
                    )
            except Exception as e:
                logger.error(f"Batch {batch_num} failed: {e}", exc_info=True)
                completed_batches += 1
```

main.py 의 spawn 강제:

```python
# main.py
import multiprocessing
if __name__ == "__main__":
    try:
        multiprocessing.set_start_method('spawn', force=True)
    except RuntimeError:
        pass
    uvicorn.run("main:app", workers=1, ...)
```

---

## 1. 왜 ProcessPool 인가 — GIL 의 정체

CPython 의 **GIL (Global Interpreter Lock)** 은 한 번에 하나의 스레드만 파이썬 바이트코드를 실행하게 함.
- I/O 대기 중에는 GIL 풀림 → 스레드풀이 효과적.
- CPU 작업 (파싱/변환/직렬화) 은 GIL 이 직렬화 → 스레드 N개 띄워도 1코어밖에 못 씀.

이 마이그레이션 워크로드:
- DB fetch (I/O) → JSON 변환 (CPU) → 한글 형태소·자모 분리 (CPU) → ES bulk index (I/O)
- CPU 비중이 충분히 커서 멀티코어 활용이 필요 → **ProcessPool**.

ProcessPool 은 **각 워커가 독립 인터프리터** → GIL 영향 없음. 단점:
- 인자/반환값을 **pickle로 직렬화/역직렬화** → 큰 객체 주고받으면 비용 큼.
- 워커 시작 비용 큼 (특히 spawn).
- 워커 간 메모리 공유 안 됨.

---

## 2. 워커 자원 캐시 — `_worker_resource_cache`

핵심 트릭: 워커 프로세스 안에서 **모듈 전역 dict** 를 두고 첫 배치에서만 무거운 자원(MongoDB 사전, 변환기) 로드.

```python
_worker_resource_cache = {}

def process_batch_worker(args):
    global _worker_resource_cache
    if 'transformer' not in _worker_resource_cache:
        # 첫 배치: 사전 로드 (MongoDB 풀스캔)
        ...
        _worker_resource_cache['transformer'] = transformer
    transformer = _worker_resource_cache['transformer']
```

### 왜 동작하는가
ProcessPoolExecutor 는 N개의 **장기 실행** 워커 프로세스를 띄우고, 큐에서 작업을 받아 처리한다. 워커 프로세스는 풀 종료 시까지 살아있으므로 모듈 전역 변수도 유지됨.

### 효과
- MongoDB 사전 로드: 수십초 → 첫 배치 1번만 발생
- 100배치를 처리해도 사전 로드는 워커 수 만큼만 (예: 10 worker → 10번)
- 누적 시간 절약: 배치 수 × 사전 로드 시간 → 워커 수 × 사전 로드 시간

### 함정
- 캐시가 살아있는 동안 메모리 점유.
- 풀 종료 전에는 회수 안 됨.
- 자원이 mutable 하면 **concurrent 안전성** 까지는 모듈 전역이라 영향 없음 (워커 프로세스마다 독립).

---

## 3. spawn vs fork — 시작 방법의 함정

```python
multiprocessing.set_start_method('spawn', force=True)
```

| 방식 | 동작 | 장점 | 단점 |
|------|------|------|------|
| **fork** (Linux 기본) | 부모 메모리/락/FD 복제 | 빠름, 부모 상태 그대로 | asyncio 루프, DB 풀, 락 등 부모 자원이 자식으로 복제되어 **deadlock/이중사용 위험** |
| **spawn** | 새 인터프리터 시작 + 모듈 다시 import | 안전, 부모 상태와 독립 | 시작 느림, pickle 가능한 인자만 전달 가능 |
| **forkserver** | 미리 띄워둔 fork-server 가 자식 생성 | fork보다 안전 + 느림 빠름 | 설정 복잡 |

asyncio + uvicorn + ProcessPool 조합에서는 **spawn 강제** 가 정석. fork 면 자식이 부모의 이벤트 루프 핸들을 상속받아 다양한 에러 발생.

`force=True` 는 다른 모듈이 이미 set_start_method 했을 때 덮어쓰기.

---

## 4. as_completed vs map vs gather

```python
with ProcessPoolExecutor(max_workers=10) as executor:
    futures = {executor.submit(worker, task): task[0] for task in tasks}
    for fut in as_completed(futures):
        batch_num = futures[fut]
        result = fut.result(timeout=300)
        # 진행률/체크포인트 업데이트
```

### `as_completed` 의 의미
- 여러 future 를 **완료되는 순서대로** yield.
- 진행률 업데이트, 체크포인트 저장에 적합 (한 배치 끝날 때마다 반응).

### 대안과 차이

| API | 결과 순서 | 적합한 케이스 |
|-----|----------|--------------|
| `executor.map(fn, iter)` | **입력 순서** 유지 | 결과 모아서 일괄 처리, 순서 중요 |
| `as_completed(futures)` | **완료 순서** | 진행률 표시, 빠른 피드백 |
| `wait(futures)` | 일괄 대기 (FIRST_COMPLETED/ALL_COMPLETED 모드) | 타임아웃 일괄, 조건부 |

### `future.result(timeout=300)`
- 워커 hang 방지. 5분 안에 끝나야 함.
- timeout 발생 시 `concurrent.futures.TimeoutError`.
- 단점: timeout 후에도 워커는 계속 실행됨 (취소 안 됨). 풀 자체를 죽여야 정리.

---

## 5. 비동기 + ProcessPool 의 결합

`run_migration_async` 는 `async def` 인데 안에서 `with ProcessPoolExecutor(...)` 를 동기로 쓴다.

```python
async def run_migration_async(self, ...):
    ...
    with ProcessPoolExecutor(max_workers=N) as executor:
        for fut in as_completed(futures):
            result = fut.result(timeout=300)  # ← 동기 블로킹
```

**문제**: `fut.result()` 는 동기. 호출되면 이벤트 루프가 막힘 → 다른 async 작업 (헬스체크, 다른 요청) 처리 불가.

**개선 옵션**:

(a) `asyncio.get_event_loop().run_in_executor(executor, fn, args)` — async 친화 API:
```python
async def run_migration_async(self, ...):
    loop = asyncio.get_event_loop()
    with ProcessPoolExecutor(max_workers=N) as executor:
        coros = [loop.run_in_executor(executor, process_batch_worker, task) for task in tasks]
        for coro in asyncio.as_completed(coros):
            result = await coro
            # 처리
```

(b) `asyncio.to_thread(...)` 로 fut.result 를 별도 스레드로:
```python
result = await asyncio.to_thread(fut.result, 300)
```

위 코드는 마이그레이션이 시작되면 다른 API 호출이 거의 없는 운영 가정 + 단순함을 우선. 하지만 헬스체크가 막히는 부작용 가능성은 인지해야 함.

---

## 6. 입출력 직렬화 비용

`executor.submit(worker, task)` 의 task 와 worker 의 반환값은 **pickle** 로 직렬화되어 IPC.

```python
batch_tasks.append((batch_num, batch_applications, total_batches, log_q))
# batch_applications 가 1만 개의 짧은 문자열이면 → 수십 KB 직렬화. 빠름.
# 만약 거대 numpy array 였다면 → 수백 MB pickle → 병목.
```

위 코드는 입력으로 **출원번호 문자열 리스트** 만 보내고, 워커가 자체적으로 DB fetch 함 → 네트워크/IPC 비용 최소화.

반환값은 통계 + ID 목록 → 작음. ES 인덱싱은 워커 안에서 직접 하므로 큰 데이터를 부모로 안 보냄.

**원칙**: ProcessPool 에 큰 객체를 넘기지 마라. 핸들/식별자만 넘기고, 워커에서 직접 자원에 접근.

---

## 7. 타임아웃, 실패, 체크포인트의 결합

```python
for future in as_completed(future_to_batch):
    try:
        result = future.result(timeout=300)
        completed_batches += 1
        total_processed += result['processed']
        ...
        if completed_batches % 10 == 0:
            self.checkpoint_manager.save_checkpoint(
                completed_batches, total_success, total_failed
            )
    except Exception as e:
        logger.error(f"Batch {batch_num} failed: {e}", exc_info=True)
        completed_batches += 1
```

**관찰**:
- 한 배치 실패해도 다음 배치 계속 (`continue` 효과).
- 10배치마다 체크포인트 → 운영 중 SIGTERM 받으면 최대 10배치 손실.
- 체크포인트 저장 자체가 실패하면 다음 시도 — 멱등 (덮어쓰기).

**대안**:
- 매 배치 체크포인트: I/O 비용 ↑, 손실 ↓.
- 실패 배치를 별도 큐로: 재시도 로직 분리.

---

## 8. ProcessPool 의 진단

운영 중 만난 함정:
- 워커 프로세스가 **OOM 으로 silent kill**: 부모는 future.result() 가 BrokenProcessPool 발생.
- BrokenProcessPool 한 번 발생하면 풀 전체가 죽어서 **재생성 필요**.
- spawn 시작 비용 (~1초/워커) → 풀 재시작 비용도 비례.

방어:
- 워커 메모리 사용량 측정 (psutil) + 임계 도달 시 자발적 종료.
- 풀을 `with` 가 아닌 명시적 `executor` 로 두고 BrokenProcessPool 시 새 풀 생성.

---

## 9. 응용 포인트

- CPU-bound + 멀티코어 활용이 필요하면 ProcessPool. I/O-bound 만이면 ThreadPool 또는 asyncio.
- `multiprocessing.set_start_method('spawn', force=True)` 를 main 진입점에 명시.
- 무거운 워커 자원은 **모듈 전역 dict 캐시** 패턴으로 첫 배치만 로드.
- 큰 객체를 IPC 로 주고받지 않는다. 식별자만 보내고 워커가 직접 가져오게.
- async 컨텍스트라면 `loop.run_in_executor` 또는 `asyncio.to_thread` 로 감싸서 이벤트 루프 보호.
- 배치 간격으로 체크포인트 → 그레이스풀 재기동 가능.
