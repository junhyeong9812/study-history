# 다국가 격리 운영 — 5개국 독립 인스턴스 패턴

> 한 시스템이 여러 지역을 다룰 때, 단일 코드베이스 vs 격리 인스턴스의 트레이드오프.
> 5개국(KR/JP/CH/US/EU) 마이그레이션 + 검색을 **국가별 독립 컨테이너** 로 운영한 경험.

---

## 1. 격리 단위

```
GPU 서버
├── kr-search-v2/  ← KR 전용
│   ├── docker-compose.yml
│   ├── kr-elasticsearch-v2  (포트 10001)
│   ├── kr-kibana-v2         (포트 10011)
│   └── kr-migration-api-v2  (포트 10021)
├── jp-search-v2/  ← JP 전용 (10002/10012/10022)
├── ch-search-v2/
├── us-search-v2/
└── eu-search-v2/
```

각 국가:
- 자체 docker-compose.
- 자체 ES 인덱스 + Kibana + migration-api.
- 별도 포트 (10001~10025).
- 별도 데이터 디렉토리.

→ 한 국가 장애가 다른 국가에 미치지 않음.

---

## 2. 코드 단위 격리

```
example-app/
├── kr-search-app/   ← KR 코드 (FastAPI 인스턴스)
├── jp-search-app/   ← JP 코드
├── ch-search-app/
├── us-search-app/
└── eu-search-app/
```

### 각 국가별 코드의 차이

대부분 거의 동일하지만:
- 데이터 변환 로직 (KR 만 customer_indexer 가 있음).
- 발음 처리 (KR/EU 는 한글 자모 분해 + g2p, JP 는 로마자, US 는 영어 전용).
- 매핑/분석기 일부 차이.

### 공통 라이브러리화 안 한 이유

**시도하지 않은 것이 아니라, 의도적으로 안 함**:
- 국가별 스키마가 점점 벌어지는 추세.
- 공통화 시 한 곳 변경이 5개국 동시 영향 → 부담.
- 복제 5개 + 각자 진화 → 변경 격리.
- 한 국가 빌드 실패가 다른 국가 빌드/배포에 영향 없음.

비용:
- 진짜 공통 변경 (예: 헬스체크 추가) 은 5번 복붙.
- 5개 인스턴스가 서로 다른 라이브러리 버전 사용 가능.

→ "DRY 가 항상 옳지는 않다. 격리 가치가 코드 중복 비용보다 클 수 있다."

---

## 3. 마이그레이션 워크플로 — 3단계

```
Step 1: 출원번호 추출
  │
  │ DB 쿼리 (전체 또는 increment)
  │ 결과를 텍스트 파일로 저장
  │
  ▼
Step 2: 배치 병렬 처리
  │
  │ ProcessPoolExecutor, 워커 N개
  │ 워커별 독립 DB 커넥션
  │ 관련 테이블 수집 + DataTransformer 변환 + bulk_index
  │
  ▼
Step 3: 실패 건 재처리
  │
  │ failed 리스트 기반 retry
  │ 한계 도달 시 최종 실패 기록
```

각 단계는 멱등 (재실행 가능).

### HTTP API
| 경로 | 동작 |
|------|------|
| `POST /orch/prepare` | Step 1 (출원번호 추출) |
| `POST /orch/start` | Step 2 (배치 시작) |
| `POST /orch/prepare-incremental` | 증분 Step 1 |
| `POST /orch/start-incremental` | 증분 Step 2 |
| `POST /orch/retry` | Step 3 (실패 재처리) |
| `POST /orch/stop` | 강제 중지 |
| `GET /orch/status` | 진행 상태 |

→ orch 측 (관제) 가 이 API 를 호출.

---

## 4. 체크포인트 — 재기동 안전

```python
class CheckpointManager:
    def save_checkpoint(self, processed_batches, success, failed):
        with open(self.checkpoint_file, 'w') as f:
            f.write(f"{processed_batches},{success},{failed},{datetime.now().isoformat()}\n")

    def load_checkpoint(self):
        if self.checkpoint_file.exists():
            with open(self.checkpoint_file) as f:
                line = f.readline().strip()
                parts = line.split(',')
                return {
                    "processed_batches": int(parts[0]),
                    "success": int(parts[1]),
                    "failed": int(parts[2]),
                    "timestamp": parts[3],
                }
        return None
```

### 패턴
- 매 N배치마다 체크포인트 저장.
- 시작 시 `resume=True` 면 체크포인트에서 이어서.
- 텍스트 파일 (CSV) — 단순/견고.

### 정합성
- 마지막 체크포인트 ~ 다음 체크포인트 사이의 배치는 재처리 가능.
- ES 색인은 `_id` 기반 멱등 → 중복 처리해도 OK.

---

## 5. 증분 마이그레이션 — `updated_after`

```python
class PrepareApplicationsRequest(BaseModel):
    force_recreate: bool = Field(False)
    batch_size: int = Field(50000, ge=1000, le=100000)
    updated_after: Optional[str] = Field(None, description="YYYY-MM-DD")
```

### 의도
- 전체 5천만 건 처음 색인은 며칠.
- 이후 매일 변경분만 색인 → 분 단위 처리.
- DB 쿼리에 `WHERE updated_at > '2026-04-29'` 추가.

### 파일명 분리
```python
if updated_after:
    date_suffix = updated_after.replace('-', '')
    file_name = f"migrate_applications_updated_{date_suffix}.txt"
else:
    file_name = "migrate_applications.txt"
```

→ 전체 vs 증분 결과를 다른 파일로 → 동시 운영 안전.

---

## 6. 스케줄링 — 국가별 독립

orch 의 scheduler:
```python
_schedules = {
    "kr": {"cron": "0 2 * * *", "type": "incremental"},  # 매일 02:00 KST
    "jp": {"cron": "0 3 * * *", "type": "incremental"},  # 매일 03:00
    # ...
}

async def scheduler_loop():
    while True:
        for country, sched in _schedules.items():
            if cron_matches(sched["cron"], now):
                run_id = uuid.uuid4().hex
                asyncio.create_task(_trigger(country, run_id))
        await asyncio.sleep(60 - now.second)  # 분 정렬
```

각 국가는 독립 trigger:
- KR 스케줄 발화 → KR 만 마이그레이션 시작.
- KR 가 hang 해도 JP 스케줄 정상.
- 동시 실행 가능 (충돌 없음).

---

## 7. agent 측 라우팅 — country sentinel

```python
async def handle_command(msg):
    action = msg.get("action", "")
    country = msg.get("country", "")

    if action.startswith("system."):
        # country="__self__" sentinel — agent 자신 명령
        return await _handle_system_command(action, ..., start)

    server = _find_server(country)
    if not server:
        return _error_response(f"Unknown country: {country}")

    # country 별 server 메타로 라우팅
    base = server.migration_api  # http://localhost:10021
    # ...
```

### `_find_server`
```python
SERVERS = [
    ServerConfig(country="kr", migration_api="http://localhost:10021", compose_path="/data/...kr.../docker-compose.yml"),
    ServerConfig(country="jp", migration_api="http://localhost:10022", compose_path="..."),
    # ...
]

def _find_server(country):
    return next((s for s in SERVERS if s.country == country), None)
```

→ 명령에 `country` 만 있으면 agent 가 적절한 인스턴스로 라우팅.

---

## 8. ES 색인 단위 격리

각 국가 ES:
- 자체 인덱스 (`kr_trademark_v2`, `jp_trademark_v2`, ...).
- 자체 settings/mappings (분석기 일부 다름).
- 자체 노드 / heap.

→ 한 국가 인덱스 corruption 이 다른 인덱스에 영향 없음.

검색 시:
- 국가별 별도 ES 엔드포인트 호출.
- 또는 cross-cluster search 로 통합 (이 프로젝트는 별도 호출).

---

## 9. 운영 부담의 균형

### 격리의 비용
- 5개국 같은 변경 5번.
- 5개국 코드 동기 유지보수.
- 5개 docker-compose 파일 관리.
- 디스크 용량 5배 (각자 ES 데이터).

### 격리의 가치
- 한 국가 장애 다른 국가 영향 없음.
- 국가별 진화 자유도.
- 단계적 배포 (한 국가만 새 코드로 먼저 시도).
- 특정 국가의 운영 정책 (예: 정전 시간) 별도 적용.

### 절충점 — 부분 공통화
- 데이터 변환의 공통 부분만 라이브러리.
- DB 클라이언트는 각자 인스턴스.
- 라우터/매핑은 국가별 별도.

이 프로젝트는 "거의 완전 격리" 선택. 운영 단순함과 변경 자유도 우선.

---

## 10. 응용 포인트

- 다국가/다지역 운영은 격리 인스턴스 패턴이 일반적으로 유리.
- 코드 공통화 욕심을 누르고, 격리 가치를 우선 평가.
- 마이그레이션은 멱등성 (체크포인트 + `_id` 색인) 으로 재기동 안전.
- 증분 처리는 `updated_after` 같은 파라미터 + 별도 파일명.
- 스케줄링은 국가별 독립 trigger.
- agent 라우팅은 country 필드 + 자기 자신 sentinel.
- 부분 공통화는 변환 로직만 등 좁게.
