# Redis Streams XADD 발행자 패턴

> 마이그레이션 진행 상황을 모니터링 서버로 보내는 publisher.
> XADD + maxlen + 분류 헬퍼 + silent fail 정책.

---

## 0. 분석 대상 코드

```python
# orchestration/app/migration/stream_publisher.py
import json
import logging
from datetime import datetime
from typing import Optional
import redis.asyncio as aioredis
from app.config import MONITORING_REDIS_HOST, MONITORING_REDIS_PORT

logger = logging.getLogger("migration.stream")

STREAM_MIGRATION_PROGRESS = "stream:migration:progress"
STREAM_MIGRATION_ERROR = "stream:migration:error"
STREAM_MIGRATION_EVENT = "stream:migration:event"
STREAM_MAXLEN = 100_000

_redis: Optional[aioredis.Redis] = None


async def init_redis():
    global _redis
    _redis = aioredis.Redis(
        host=MONITORING_REDIS_HOST,
        port=MONITORING_REDIS_PORT,
        decode_responses=True,
    )
    try:
        await _redis.ping()
        logger.info(f"Redis connected: {MONITORING_REDIS_HOST}:{MONITORING_REDIS_PORT}")
    except Exception as e:
        logger.warning(f"Redis connection failed (will retry on publish): {e}")


async def close_redis():
    global _redis
    if _redis:
        await _redis.close()
        _redis = None


async def publish_progress(country: str, migration_data: dict):
    """진행률 발행. Agent push마다 호출."""
    if not _redis:
        return
    try:
        await _redis.xadd(STREAM_MIGRATION_PROGRESS, {
            "data": json.dumps({
                "country": country,
                "total_count": migration_data.get("processed", 0) + migration_data.get("failed", 0),
                "processed": migration_data.get("processed", 0),
                "success": migration_data.get("success", 0),
                "failed": migration_data.get("failed", 0),
                "progress_percentage": migration_data.get("progress_percentage", 0),
                "processing_rate": migration_data.get("processing_rate", 0),
                "timestamp": datetime.now().isoformat(),
            })
        }, maxlen=STREAM_MAXLEN)
    except Exception as e:
        logger.warning(f"Failed to publish progress for {country}: {e}")


async def publish_error(country: str, error_data: dict):
    if not _redis:
        return
    try:
        await _redis.xadd(STREAM_MIGRATION_ERROR, {
            "data": json.dumps({
                "country": country,
                "application_no": error_data.get("app_no", ""),
                "batch_num": error_data.get("batch", 0),
                "error_category": _classify_error(error_data.get("error", "")),
                "error_message": error_data.get("error", ""),
                "timestamp": error_data.get("timestamp", datetime.now().isoformat()),
            })
        }, maxlen=STREAM_MAXLEN)
    except Exception as e:
        logger.warning(f"Failed to publish error for {country}: {e}")


async def publish_event(country: str, event_type: str, event_data: dict = None):
    if not _redis:
        return
    try:
        await _redis.xadd(STREAM_MIGRATION_EVENT, {
            "data": json.dumps({
                "country": country,
                "event_type": event_type,
                "event_data": event_data or {},
                "timestamp": datetime.now().isoformat(),
            })
        }, maxlen=STREAM_MAXLEN)
    except Exception as e:
        logger.warning(f"Failed to publish event for {country}: {e}")


def _classify_error(error_msg: str) -> str:
    msg = error_msg.lower()
    if "connection" in msg or "connect" in msg:
        return "db_connection" if any(k in msg for k in ("mysql", "maria", "pymysql")) else "es_connection"
    if "mapper_parsing" in msg or "mapping" in msg:
        return "es_mapping"
    if "bulk" in msg or "index" in msg:
        return "es_indexing"
    if "transform" in msg or "convert" in msg or "key" in msg:
        return "data_transform"
    if "timeout" in msg:
        return "timeout"
    return "unknown"
```

---

## 1. `XADD` 의 형태와 `maxlen`

```python
await redis.xadd(stream_name, {"data": json.dumps(payload)}, maxlen=100_000)
```

### 인자
- `stream_name`: 스트림 키 (없으면 자동 생성).
- `fields`: dict. 값은 모두 string 으로 저장됨.
- `maxlen`: 스트림 trimming 임계.

### 반환
- 자동 생성된 message ID (`"1700000000000-0"` 형태: ms timestamp + sequence).
- 명시적 ID 도 가능: `xadd(stream, fields, id="1700000000000-1")`. 보통 안 씀 (자동이 안전).

### `maxlen` 의 두 모드

```python
xadd(stream, fields, maxlen=100_000)            # exact: 정확히 10만으로 trim
xadd(stream, fields, maxlen=100_000, approximate=True)  # 약식 (기본 후자)
```

- **exact**: 매번 RBT 조회로 정확하게 자름. 비용 큼.
- **approximate (~)**: radix tree 노드 단위로 자름. 100,001 이 잠깐 될 수 있지만 매우 빠름. **운영에서는 거의 항상 이쪽**.
- redis-py async 에서는 `approximate` 인자는 redis 버전/SDK 에 따라 다를 수 있음. CLI 의 `~` 와 동등.

### 왜 `maxlen` 인가
- 스트림은 영속 → 무한 증가하면 메모리 폭주.
- 100,000 = 약 1주~1개월의 최근 이벤트 보관 (트래픽에 따라).
- 더 오래된 데이터는 별도 영속 저장(이 프로젝트는 PostgreSQL).

---

## 2. JSON 페이로드를 단일 필드에 — `{"data": json.dumps(...)}`

위 코드는 모든 필드를 한 번에 JSON 으로 직렬화해 `data` 한 필드에 담는다.

```python
"data": json.dumps({"country": ..., "processed": ..., ...})
```

### 대안: 필드별 분리
```python
await redis.xadd(stream, {
    "country": "kr",
    "processed": str(123),  # 모든 값은 string 이어야 함
    "success": str(100),
})
```

### 트레이드오프
| 방식 | 장점 | 단점 |
|------|------|------|
| 단일 JSON | 컨슈머 코드 단순, 스키마 변경 자유 | Redis CLI 디버깅 불편 (`XRANGE` 결과 한 줄) |
| 필드별 | XINFO/XRANGE 가독성 ↑, 부분 업데이트 가능 | 모든 값 string 변환 + 컨슈머에서 다시 파싱 |

이 프로젝트는 단일 JSON 선택 — 핸들러 코드가 단순하고 스키마가 자주 바뀌어서.

---

## 3. Silent Fail 정책

```python
async def publish_progress(country, migration_data):
    if not _redis:
        return
    try:
        await _redis.xadd(...)
    except Exception as e:
        logger.warning(f"Failed to publish progress: {e}")
```

### 의도
- 모니터링 publish 가 실패해도 **메인 마이그레이션은 계속**.
- 이벤트는 보조 — 빠지면 모니터링 누락이지 마이그레이션 실패가 아님.
- Redis 단절을 마이그레이션 운영의 결정타로 만들지 않음.

### 한계
- 로그가 쌓일 수 있음 → log level 을 warning 으로 두고 별도 알람 임계 설정.
- 정말 중요한 이벤트(예: migration_completed) 도 silent fail 한다는 점 → 운영자 입장에선 이벤트 누락 가능성 인지 필요.

### 대안: 재시도 큐
```python
_pending_publishes: list[tuple] = []

async def publish_progress(country, data):
    payload = {...}
    try:
        await _redis.xadd(STREAM, {"data": json.dumps(payload)}, maxlen=...)
    except Exception:
        _pending_publishes.append((STREAM, payload))

# 별도 task가 _pending_publishes 를 주기적으로 drain 시도
```

→ 위 코드는 단순함을 우선해 안 씀.

---

## 4. 분류 헬퍼 — `_classify_error`

```python
def _classify_error(error_msg: str) -> str:
    msg = error_msg.lower()
    if "connection" in msg or "connect" in msg:
        return "db_connection" if any(k in msg for k in ("mysql", "maria", "pymysql")) else "es_connection"
    if "mapper_parsing" in msg or "mapping" in msg:
        return "es_mapping"
    ...
    return "unknown"
```

### 의도
- 컨슈머/대시보드에서 같은 카테고리로 집계하기 쉽도록 publisher 측에서 분류.
- 메시지가 바뀌어도 분류 로직만 업데이트하면 모든 컨슈머에 자동 반영.

### 패턴 분석
- 키워드 기반 매칭 — 단순하고 빠름.
- 우선순위 순서로 if-elif 체인 → 첫 매치 채택.
- 매치 안 되면 `"unknown"` → 운영 중 새 카테고리 발견하면 헬퍼에 추가.

### 트레이드오프
- 매우 단순. 정확도 ~80%.
- 정확하게 분류하려면 ES 응답 코드, MariaDB error code 등 구조화된 메타로 분류해야 함.
- 이 프로젝트는 "운영 중 카테고리별 빈도" 정도면 충분 → 단순 키워드.

---

## 5. publisher 의 lifespan 통합

```python
# main.py
@asynccontextmanager
async def lifespan(app):
    await init_redis()  # publisher 측 redis
    ...
    yield
    ...
    await close_redis()
```

핸들러 코드는 모듈 전역 `_redis` 를 참조. 시작 시 초기화, 종료 시 close.

### init 실패 허용
```python
async def init_redis():
    global _redis
    _redis = aioredis.Redis(host=..., port=...)
    try:
        await _redis.ping()
    except Exception as e:
        logger.warning(f"Redis connection failed (will retry on publish): {e}")
```

→ ping 실패해도 `_redis` 객체는 살아있음. publish 시점에 실제 명령에서 또 실패할 수 있고, 그것도 silent fail.

**의도**: 시작 자체를 막지 않음. Redis 가 잠깐 죽어 있어도 앱 부팅 가능.

→ 다만 redis 가 영영 안 살면 영영 publish 안 됨. 별도 health-check + 알람 필요.

---

## 6. 발행 vs 소비의 짝

| 측 | 코드 |
|----|------|
| Publisher | 마이그레이션 오케스트레이터 → `xadd` |
| Consumer | 모니터링 서버 → `xreadgroup` + `xack` |

핵심 디커플링:
- 마이그레이션 측은 모니터링 측을 모름.
- 모니터링 측은 마이그레이션 측이 죽어도 영향 없음.
- 둘이 같은 시점에 안 살아도 OK (Stream 이 buffer).

---

## 7. 응용 포인트

- 이벤트/메트릭 발행은 항상 Streams + `maxlen` 으로 자동 trimming.
- 단일 JSON 필드 vs 분리 필드 — 컨슈머 단순함 vs CLI 디버깅 편의 트레이드오프.
- 운영 중요도 낮은 이벤트는 silent fail (메인 흐름 보호).
- 중요 이벤트는 재시도 큐 또는 외부 영속 큐(Kafka 등) 고려.
- 발행 시 분류/정규화를 publisher 가 담당하면 컨슈머 일관성 ↑.
- init 실패 허용으로 Redis 일시 장애에 robust 한 시작.
