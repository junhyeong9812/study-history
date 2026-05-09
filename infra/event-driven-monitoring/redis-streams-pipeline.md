# Redis Streams 기반 모니터링 파이프라인

> 검색 서비스의 운영 이벤트를 Redis Streams 로 내보내고, 별도 모니터링 시스템이 영속 저장하는 패턴.
> 핵심 서비스 측 부담 최소화 + 모니터링 측의 독립적 진화.

---

## 1. 시스템 경계

```
[검색 서비스 (핵심 비즈니스)]
   │ 이벤트 발생: 검색, 에러, 블랙리스트, 키워드, ...
   │ XADD stream:search:log {data: {...}}
   ▼
[Redis Primary] (Streams + maxlen 으로 자동 trimming)
   │ 7개 stream 보관
   │ Consumer Group: monitoring-group
   ▼
[monitoring (별도 컨테이너)]
   │ XREADGROUP block=5s, count=10
   │ 7개 핸들러 분기
   ▼
[PostgreSQL] (영구 저장 + 통계 집계)
   │
   ▼
[모니터링 대시보드 UI]
```

핵심: 검색 서비스 → Redis 까지가 **publish 책임**, 그 이후는 **monitering 책임**.

---

## 2. 왜 Redis Streams 인가

### Pub/Sub vs Queue (List) vs Streams 비교

| 항목 | Pub/Sub | List | Streams |
|------|---------|------|---------|
| persistence | ✗ | △ (RDB/AOF 의존) | ✓ |
| 컨슈머 그룹 | ✗ | ✗ | ✓ |
| 재처리 | ✗ | pop 시 사라짐 | ACK 기반 pending |
| 순서 보장 | 송신 순서 | 순차 | ID 단조 |
| 백프레셔 | drop | block | maxlen 자동 |
| 사용처 | 즉시 알림 | 단순 큐 | 이벤트 스트리밍 |

운영 이벤트의 요구:
- **영속**: 일시적 monitering 다운에도 손실 없어야.
- **재처리**: 핸들러 실패 시 재시도.
- **확장**: 추후 monitering 인스턴스 늘릴 때 컨슈머 그룹.
- **순서**: 같은 사용자의 이벤트는 순서 보존.

→ Streams 가 모든 요구 충족.

---

## 3. Decoupling 의 가치

### 3.1 검색 서비스 측은 publish 만

```python
# 검색 서비스 (개념)
async def search_handler(req):
    result = await es_search(req)
    # publish 만, 응답에 영향 없음
    asyncio.create_task(redis.xadd("stream:search:log", {
        "data": json.dumps({
            "country": req.country,
            "user_pk": req.user_pk,
            "query": req.query,
            "timestamp": now_iso(),
        })
    }, maxlen=100_000))
    return result
```

- **fire-and-forget**: publish 실패해도 검색 응답은 정상.
- 검색 서비스는 monitering 의 존재를 모름.
- 검색 서비스 입장에서 monitering 측 코드 변경은 무관.

### 3.2 monitering 측의 자유로운 진화

- 새 통계, 새 대시보드, 새 알림 — 검색 서비스 영향 없이 추가.
- 핸들러 변경, DB 마이그레이션, 인덱스 추가 — 모두 monitering 안에서.

### 3.3 동시 다중 컨슈머 가능

- 한 stream 을 여러 그룹이 각자 소비:
  - `monitoring-group`: monitering 시스템.
  - `analytics-group`: 별도 분석 시스템.
  - `archival-group`: S3 등 콜드 스토리지.
- 각 그룹은 독립적으로 진행 (last-delivered-id 별도).
- 검색 서비스는 변경 없이 신규 소비자 추가 가능.

---

## 4. at-least-once + 멱등성

### 4.1 ACK 기반 재처리
```python
try:
    await handler(data)
    await redis.xack(stream, group, msg_id)
except json.JSONDecodeError:
    await redis.xack(...)  # poison message → ACK
except Exception:
    pass  # no-ACK → pending → 다음에 재처리
```

→ "최소 한 번" 처리. 같은 메시지가 2번 처리될 수 있음.

### 4.2 핸들러 측 멱등성

```python
# 블랙리스트 핸들러
existing = await s.execute(select(Blacklist).filter_by(ip=ip, is_active=True))
row = existing.scalar_one_or_none()
if row:
    row.level = level  # 업데이트
    row.expires_at = expires_at
else:
    s.add(Blacklist(ip=ip, ..., is_active=True))
await s.commit()
```

→ 같은 메시지 2번 도착해도 DB 는 동일 상태.

### 4.3 멱등성 확보 방법
- UPSERT (PostgreSQL `ON CONFLICT`).
- request_id 같은 unique key 로 dedupe.
- 단순 INSERT 면 unique 제약 + 충돌 시 무시.

at-least-once + 멱등 = 사실상 exactly-once 효과.

---

## 5. 손실/지연 시나리오

### 5.1 Redis 다운
- 검색 서비스 publish 실패 → silent fail (검색 응답에 영향 없음).
- 그동안 발생한 이벤트는 **손실**.
- 핵심 서비스 SLO 우선 → 모니터링 누락은 감내.

대안:
- 검색 서비스 측에 in-memory buffer + retry → 복잡도 ↑.
- Kafka 같은 더 안정적인 큐로 교체 → 비용 ↑.

### 5.2 monitering 다운
- Redis 가 메시지 buffer.
- monitering 재시작 시 pending 복구.
- maxlen=100_000 까지는 안전.
- 더 오래 다운되면 maxlen 초과로 오래된 이벤트 trimming.

### 5.3 PostgreSQL 다운
- 핸들러 실패 → no-ACK → pending 누적.
- pending 누적 → memory 압박 가능 (Redis 쪽).
- PG 복구 후 자동 재처리.

### 5.4 핸들러 버그 (영원히 실패)
- pending 영원히 남음.
- 모니터링 alert 필요: `XPENDING stream group` 의 lag.
- 도입 권장: DLQ (Dead Letter Queue).

---

## 6. 7개 스트림의 분리 — 도메인별 책임

| 스트림 | 핸들러 |
|--------|-------|
| `stream:search:log` | search_log_handler — 검색 통계 |
| `stream:search:error` | search_error_handler — 에러 추적 |
| `stream:search:blacklist` | blacklist_handler — IP 차단 |
| `stream:search:keyword` | keyword_handler — 키워드 분석 |
| `stream:migration:progress` | migration_progress_handler — 마이그레이션 모니터링 |
| `stream:migration:error` | migration_error_handler |
| `stream:migration:event` | migration_event_handler |

분리 이유:
- 백프레셔 격리: 검색 로그가 폭주해도 마이그레이션 이벤트는 정상.
- maxlen 별도 설정 가능 (트래픽 다른 스트림은 다른 보관량).
- 컨슈머 측 분기 단순 (스트림별 핸들러).

vs 단일 스트림 + type 분기:
- 통합 스트림 1개 + 메시지에 `event_type` 필드.
- 단순함 ↑, 격리 ↓.

---

## 7. 메시지 스키마 — 단일 JSON 필드 패턴

```python
await redis.xadd(stream, {
    "data": json.dumps({
        "country": ...,
        "user_pk": ...,
        ...
    })
}, maxlen=100_000)
```

vs 필드별 분리:
- 단일 JSON: 컨슈머 단순 (json.loads 한 번), 스키마 변경 자유.
- 필드별: Redis CLI 디버깅 ↑ (XRANGE 결과 가독성), 스키마 강제.

운영에서는 단일 JSON 이 흔함 — Redis 는 메시지 brokering 만 담당하고, 스키마는 publisher/consumer 합의.

---

## 8. maxlen — 자동 trimming

```python
xadd(stream, fields, maxlen=100_000)
```

- approximate (~) 기본 — 100,001 잠깐 가능, 단 매우 빠름.
- exact 모드는 매번 정확 trim — 비싸서 거의 안 씀.

100K 의 의미:
- 트래픽에 따라 1주~1개월의 최근 이벤트.
- 더 오래된 데이터는 PostgreSQL 측 영속 저장.

→ Redis 는 **단기 buffer**, PG 는 **영구 archive**.

---

## 9. 컨슈머 그룹의 수평 확장

현재: 단일 컨슈머 (`worker-{hostname}`).

확장:
```bash
# 각 인스턴스마다 다른 hostname / pid → 다른 consumer name
docker-compose up -d --scale monitoring-worker=3
```

- 같은 그룹 안의 N개 컨슈머가 메시지를 분산 받음.
- 각 메시지는 한 컨슈머에게만 deliver (그룹 단위 last-delivered-id).
- `XCLAIM` 으로 죽은 컨슈머의 pending 인계 가능.

→ 트래픽 증가 시 모니터링 시스템만 scale-out, 검색 서비스 무관.

---

## 10. 응용 포인트

- 핵심 서비스의 운영 이벤트는 영속 큐 (Redis Streams / Kafka) 로.
- publish 는 fire-and-forget — 핵심 서비스 응답에 영향 X.
- 컨슈머는 ACK + 멱등성 → at-least-once + 결과적 exactly-once.
- 도메인별 스트림 분리 → 백프레셔 격리.
- maxlen 으로 자동 trimming + 영구 저장은 별도 DB.
- 모니터링 시스템이 죽어도 핵심 서비스는 정상.
- 수평 확장은 컨슈머 그룹 + N 인스턴스.
