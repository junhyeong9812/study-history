# Redis Streams + Consumer Group — at-least-once 컨슈머

> 7개 스트림에서 단일 컨슈머가 메시지를 받아 PostgreSQL 에 저장하는 모니터링 파이프라인.
> XREADGROUP / XACK / pending 복구 / 무한 retry 위험까지.

---

## 0. 분석 대상 코드

```python
# monitoring/app/listener/stream_listener.py
import asyncio
import json
import logging
import socket
from app.core import redis_client
from app.listener.search_log_handler import handle_search_log
from app.listener.search_error_handler import handle_search_error
from app.listener.blacklist_handler import handle_blacklist
from app.listener.keyword_handler import handle_keyword_collect
from app.listener.migration_progress_handler import handle_migration_progress
from app.listener.migration_error_handler import handle_migration_error
from app.listener.migration_event_handler import handle_migration_event

logger = logging.getLogger(__name__)

STREAM_SEARCH_LOG = "stream:search:log"
STREAM_SEARCH_ERROR = "stream:search:error"
STREAM_BLACKLIST = "stream:search:blacklist"
STREAM_KEYWORD = "stream:search:keyword"
STREAM_MIGRATION_PROGRESS = "stream:migration:progress"
STREAM_MIGRATION_ERROR = "stream:migration:error"
STREAM_MIGRATION_EVENT = "stream:migration:event"
GROUP_NAME = "monitoring-group"
CONSUMER_NAME = f"worker-{socket.gethostname()}"

BATCH_SIZE = 10
BLOCK_MS = 5000
STREAM_MAXLEN = 100_000

INITIAL_RETRY_DELAY = 3
MAX_RETRY_DELAY = 30


async def _ensure_consumer_groups(redis):
    """컨슈머 그룹이 없으면 생성 (MKSTREAM으로 스트림도 자동 생성)"""
    for stream in [STREAM_SEARCH_LOG, STREAM_SEARCH_ERROR, STREAM_BLACKLIST, STREAM_KEYWORD,
                   STREAM_MIGRATION_PROGRESS, STREAM_MIGRATION_ERROR, STREAM_MIGRATION_EVENT]:
        try:
            await redis.xgroup_create(stream, GROUP_NAME, id="0", mkstream=True)
            logger.info(f"컨슈머 그룹 생성: {stream} / {GROUP_NAME}")
        except Exception as e:
            if "BUSYGROUP" in str(e):
                logger.debug(f"컨슈머 그룹 이미 존재: {stream} / {GROUP_NAME}")
            else:
                raise


async def _process_messages(messages, stream_name):
    """메시지 목록을 처리하고 ACK할 ID 목록 반환"""
    ack_ids = []
    for message_id, fields in messages:
        try:
            raw = fields.get("data")
            if not raw:
                logger.warning(f"빈 메시지: {stream_name}/{message_id}")
                ack_ids.append(message_id)
                continue

            data = json.loads(raw)

            if stream_name == STREAM_SEARCH_LOG:
                await handle_search_log(data)
            elif stream_name == STREAM_SEARCH_ERROR:
                await handle_search_error(data)
            elif stream_name == STREAM_BLACKLIST:
                await handle_blacklist(data)
            elif stream_name == STREAM_KEYWORD:
                await handle_keyword_collect(data)
            elif stream_name == STREAM_MIGRATION_PROGRESS:
                await handle_migration_progress(data)
            elif stream_name == STREAM_MIGRATION_ERROR:
                await handle_migration_error(data)
            elif stream_name == STREAM_MIGRATION_EVENT:
                await handle_migration_event(data)

            ack_ids.append(message_id)
        except json.JSONDecodeError as e:
            logger.error(f"메시지 파싱 실패: {stream_name}/{message_id}: {e}")
            ack_ids.append(message_id)  # 파싱 불가 메시지는 ACK하여 영구 재시도 방지
        except Exception as e:
            logger.error(f"메시지 처리 실패: {stream_name}/{message_id}: {e}", exc_info=True)
            # 처리 실패 시 ACK 하지 않음 → pending에 남아서 재처리
    return ack_ids


async def _recover_pending(redis):
    """미처리(pending) 메시지 복구 — 앱 시작 시"""
    for stream in [STREAM_SEARCH_LOG, STREAM_SEARCH_ERROR, STREAM_BLACKLIST, STREAM_KEYWORD,
                   STREAM_MIGRATION_PROGRESS, STREAM_MIGRATION_ERROR, STREAM_MIGRATION_EVENT]:
        while True:
            result = await redis.xreadgroup(
                GROUP_NAME, CONSUMER_NAME,
                streams={stream: "0"},  # 0 = pending 메시지만
                count=BATCH_SIZE,
            )
            if not result:
                break

            for stream_name, messages in result:
                if not messages:
                    break
                logger.info(f"pending 복구: {stream_name} × {len(messages)}건")
                ack_ids = await _process_messages(messages, stream_name)
                if ack_ids:
                    await redis.xack(stream_name, GROUP_NAME, *ack_ids)

            total = sum(len(msgs) for _, msgs in result)
            if total < BATCH_SIZE:
                break


async def start_stream_listener():
    """Redis Streams 리스너 시작"""
    retry_delay = INITIAL_RETRY_DELAY
    streams = {
        STREAM_SEARCH_LOG: ">", STREAM_SEARCH_ERROR: ">",
        STREAM_BLACKLIST: ">", STREAM_KEYWORD: ">",
        STREAM_MIGRATION_PROGRESS: ">", STREAM_MIGRATION_ERROR: ">", STREAM_MIGRATION_EVENT: ">",
    }

    while True:
        primary = redis_client.get_primary()
        if not primary:
            logger.warning(f"Redis Primary 없음 — {retry_delay}초 후 재시도")
            await asyncio.sleep(retry_delay)
            retry_delay = min(retry_delay * 2, MAX_RETRY_DELAY)
            continue

        try:
            await _ensure_consumer_groups(primary)
            await _recover_pending(primary)
            logger.info(f"Stream 리스너 시작: group={GROUP_NAME} consumer={CONSUMER_NAME}")
            retry_delay = INITIAL_RETRY_DELAY

            while True:
                result = await primary.xreadgroup(
                    GROUP_NAME, CONSUMER_NAME,
                    streams=streams,
                    count=BATCH_SIZE,
                    block=BLOCK_MS,
                )

                if not result:
                    continue

                for stream_name, messages in result:
                    if not messages:
                        continue
                    ack_ids = await _process_messages(messages, stream_name)
                    if ack_ids:
                        await primary.xack(stream_name, GROUP_NAME, *ack_ids)

        except asyncio.CancelledError:
            logger.info("Stream 리스너 종료")
            return
        except Exception as e:
            logger.error(f"Stream 연결 오류, {retry_delay}초 후 재연결: {e}")
            await asyncio.sleep(retry_delay)
            retry_delay = min(retry_delay * 2, MAX_RETRY_DELAY)
```

---

## 1. Streams 의 모델 — 한 페이지 요약

```
Stream (= log)         Consumer Group
  ┌─ msg_id_001 ─┐       ┌─ pending list (delivered but not acked) ─┐
  ├─ msg_id_002 ─┤       │  consumer A: [001, 003]                  │
  ├─ msg_id_003 ─┤  →    │  consumer B: [002]                       │
  ├─ msg_id_004 ─┤       └──────────────────────────────────────────┘
  └─ ...        ─┘       last-delivered-id: 004
```

- `XADD` 로 메시지 추가 (Producer).
- `XREADGROUP <group> <consumer> {stream: ">"} ...` 로 새 메시지 가져오기.
- Group 이 last-delivered-id 를 추적 → 한 메시지를 그룹 내에서 한 번만 deliver.
- 처리 후 `XACK` → pending 에서 제거.
- 처리 안 하고 죽으면 pending 에 남음 → 다른 consumer 가 `XCLAIM` 또는 같은 consumer 가 재시작 후 재처리.

---

## 2. `XGROUP CREATE` — 그룹 초기화

```python
await redis.xgroup_create(stream, GROUP_NAME, id="0", mkstream=True)
```

- `id="0"` — 스트림 처음부터 모든 메시지를 그룹이 인식. 이미 쌓인 메시지도 처리 대상.
- `id="$"` — 지금 이후 들어오는 메시지만.
- `mkstream=True` — 스트림이 없으면 자동 생성.

**`BUSYGROUP` 예외**: 이미 그룹이 있으면 발생. 무시 (idempotent 시작).

```python
except Exception as e:
    if "BUSYGROUP" in str(e):
        logger.debug(...)
    else:
        raise
```

→ 운영에서 매번 시작하더라도 안전.

**`id="0"` 의 함정**:
- 첫 시작에서는 좋음 (과거 메시지 재처리).
- 하지만 운영 중 메시지가 백만 개 쌓인 상태에서 그룹을 잘못 재생성하면 처음부터 다시 처리 → 재해.
- 안전을 위해 운영 환경은 `id="$"` 로 신규 메시지만 받게 하고, 과거는 별도 마이그레이션 작업으로.

---

## 3. `XREADGROUP` 와 `>` vs ID

```python
await primary.xreadgroup(
    GROUP_NAME, CONSUMER_NAME,
    streams={"stream:x": ">"},
    count=BATCH_SIZE,
    block=BLOCK_MS,
)
```

| ID | 의미 |
|----|------|
| `">"` | last-delivered-id 이후의 **새 메시지** |
| `"0"` | 이 consumer 의 **pending 메시지**만 (재처리용) |
| 특정 ID | 그 ID 이후의 pending |

### `count` 와 `block` 의 동작
- `count=10`: 한 번에 최대 10개 반환.
- `block=5000`: 메시지 없으면 5초 대기. 5초 내 도착하면 즉시 반환. 시간 초과면 빈 결과.
- block 없음 → 즉시 빈 결과 반환 → busy loop 위험.

### 멀티 스트림 한 번에
```python
streams = {"s1": ">", "s2": ">", "s3": ">"}
result = await r.xreadgroup(group, consumer, streams=streams, count=10, block=5000)
# result = [("s1", [(id, fields), ...]), ("s2", [...])]
```

→ N 개 스트림을 한 호출에 폴링. 7개 스트림 처리에 효율적.

---

## 4. ACK / NO-ACK 의 의미

```python
try:
    await handler(data)
    ack_ids.append(message_id)
except json.JSONDecodeError as e:
    logger.error(...)
    ack_ids.append(message_id)  # ← 파싱 불가는 ACK해서 무한 재시도 방지
except Exception as e:
    logger.error(...)
    # ACK 안 함 → pending에 남음 → 재처리
finally:
    if ack_ids:
        await primary.xack(stream_name, GROUP_NAME, *ack_ids)
```

### 의도
- **정상 처리** → ACK → pending 에서 제거.
- **파싱 실패 (poison message)** → ACK → 무한 재시도 방지.
- **다른 예외 (DB 일시 장애 등)** → no-ACK → 다음에 재처리.

### 경계 결정의 핵심
"이 메시지를 다시 시도하면 성공할 가능성이 있는가?"
- 형식 자체가 깨졌으면 영원히 실패 → ACK + 로그.
- 일시적 외부 의존성 실패 → no-ACK.

---

## 5. Pending 복구 — 시작 시 한 번

```python
async def _recover_pending(redis):
    for stream in [...]:
        while True:
            result = await redis.xreadgroup(
                GROUP_NAME, CONSUMER_NAME,
                streams={stream: "0"},  # 0 = pending 만
                count=BATCH_SIZE,
            )
            if not result:
                break
            ...
            total = sum(len(msgs) for _, msgs in result)
            if total < BATCH_SIZE:
                break
```

**시나리오**:
- T0 메시지 100개 도착, 50개 처리 중 SIGTERM.
- T1 재시작 → 50개가 pending 에 남음.
- `xreadgroup(stream, "0")` 로 그 50개를 다시 받아 처리.

**한계**:
- 다른 consumer 의 pending 은 가져오지 않음 (자기 것만).
- 다른 consumer 가 죽고 안 돌아오면 그 메시지는 영영 pending.
- → `XCLAIM`/`XAUTOCLAIM` 으로 idle 시간이 긴 메시지를 다른 consumer 가 가져갈 수 있음 (이 코드에는 없음).

### 단일 컨슈머 환경의 단순화
이 프로젝트는 컨슈머 1대 가정 → 자기 pending 만 복구하면 충분.
다중 컨슈머라면 XCLAIM 도입 필요.

---

## 6. 컨슈머 이름 — 호스트 기반

```python
CONSUMER_NAME = f"worker-{socket.gethostname()}"
```

- 컨테이너/호스트마다 고유 이름.
- 같은 호스트에서 한 인스턴스만 띄우면 충돌 없음.
- 같은 호스트에 여러 인스턴스면 `pid` 추가 권장: `f"worker-{hostname}-{os.getpid()}"`.

---

## 7. 무한 재시도의 함정 — 독 메시지

위 코드의 "ACK 안 하면 다음에 재처리" 가 양날의 검.

**시나리오**:
- 메시지가 **항상 같은 예외**를 일으킴 (예: DB 컬럼 unique 위반, 코드 버그).
- 무한히 pending 으로 살아남음.
- pending 누적 → memory 압박.
- 매 사이클마다 같은 메시지로 처리 시도 → CPU 소모.

**대응**: Dead Letter Queue (DLQ).
```python
# 시도 횟수 추적
delivery_count = await r.execute_command("XPENDING", stream, group, ...)
# delivery_count > N 이면
await r.xadd("stream:dlq", {"original_id": msg_id, "data": raw})
ack_ids.append(msg_id)
```

또는 메시지 자체에 retry counter 를 넣어서 처리.

위 코드에는 없음 — 향후 과제.

---

## 8. 백오프 재연결

```python
INITIAL_RETRY_DELAY = 3
MAX_RETRY_DELAY = 30

while True:
    primary = redis_client.get_primary()
    if not primary:
        await asyncio.sleep(retry_delay)
        retry_delay = min(retry_delay * 2, MAX_RETRY_DELAY)
        continue
    try:
        ...
    except Exception:
        await asyncio.sleep(retry_delay)
        retry_delay = min(retry_delay * 2, MAX_RETRY_DELAY)
```

- Redis 단절/장애 시 지수 백오프.
- 정상 진입(`while True` 안쪽)하면 retry_delay 리셋.
- WS 재연결과 동일 패턴.

---

## 9. CancelledError — 그레이스풀 종료

```python
except asyncio.CancelledError:
    logger.info("Stream 리스너 종료")
    return
```

- lifespan shutdown 에서 task 가 cancel 되면 CancelledError 발생.
- 이걸 잡지 않으면 traceback 으로 보임. 잡아서 깔끔히 종료.
- `return` 으로 외부 while True 도 빠져나옴.

---

## 10. 응용 포인트

- 모니터링/로깅용 이벤트 파이프라인은 거의 항상 Streams + Consumer Group.
- 처음 시작은 `id="0"` 으로 풀 처리, 운영 진입 후 `id="$"` 로 전환 고려.
- ACK 정책: poison message 는 ACK + 로그, 일시 실패는 no-ACK.
- 재시작 시 pending 복구 루틴 필수.
- 무한 재시도 차단을 위해 retry counter / DLQ 도입 권장.
- 컨슈머 이름은 hostname + pid 조합으로 충돌 방지.
- 백오프 재연결 + CancelledError 처리로 안전한 lifespan 통합.
