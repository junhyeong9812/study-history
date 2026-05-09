# elasticsearch-py — 클라이언트 + parallel_bulk + 검색

> 5천만 건 트레이드마크 데이터를 ES 에 색인할 때의 표준 패턴.

---

## 0. 분석 대상 코드

### 0.1 클라이언트 초기화

```python
# example-app/src/database/elastic.py
from elasticsearch import Elasticsearch, helpers
from elasticsearch.exceptions import ConnectionError, RequestError


class ElasticsearchConnection:
    def __init__(self):
        self.client: Optional[Elasticsearch] = None
        self._initialize_connection()

    def _build_connection_config(self) -> Dict:
        config = {
            "hosts": [f"http://{settings.ES_HOST}:{settings.ES_PORT}"],
            "timeout": settings.ES_REQUEST_TIMEOUT,
            "max_retries": 3,
            "retry_on_timeout": True
        }
        if settings.ES_USER and settings.ES_PASSWORD:
            config["http_auth"] = (settings.ES_USER, settings.ES_PASSWORD)
        return config

    def _test_connection(self):
        info = self.client.info()
        es_version = info['version']['number']
        logger.info(f"Connected to Elasticsearch version {es_version}")

        health = self.client.cluster.health()
        cluster_status = health['status']
        node_count = health['number_of_nodes']

        if cluster_status == 'red':
            logger.warning("Cluster status is RED - some shards are not allocated")
        elif cluster_status == 'yellow':
            logger.warning("Cluster status is YELLOW - replica shards are not allocated")
```

### 0.2 Bulk 색인

```python
def bulk_index(self, documents: List[Dict], index_name: Optional[str] = None) -> Dict:
    """대량 색인"""
    index_name = index_name or settings.ES_UNIFIED_INDEX_NAME
    doc_count = len(documents)

    actions = []
    for doc in documents:
        doc_copy = doc.copy()
        doc_id = doc_copy.pop('_id', None)
        action = {
            "_index": index_name,
            "_id": doc_id,
            "_source": doc_copy
        }
        actions.append(action)

    success_count = 0
    failed_docs = []

    for success, info in helpers.parallel_bulk(
        self.client,
        actions,
        chunk_size=settings.ES_BATCH_SIZE,
        raise_on_error=False,
        thread_count=4
    ):
        if success:
            success_count += 1
        else:
            failed_docs.append(info)
            logger.warning(f"Failed to index document: {info}")

    return {
        "success": success_count,
        "failed": failed_docs,
        "total": doc_count
    }
```

### 0.3 검색

```python
def search(self, query: Dict, index_name: Optional[str] = None) -> Dict:
    index_name = index_name or settings.ES_UNIFIED_INDEX_ALIAS or settings.ES_UNIFIED_INDEX_NAME

    response = self.client.search(index=index_name, body=query)
    hits = response.get('hits', {})
    total_hits = hits.get('total', {}).get('value', 0)
    took_ms = response.get('took', 0)

    if 'aggregations' in response:
        logger.debug(f"Aggregations returned: {list(response['aggregations'].keys())}")

    return response
```

---

## 1. 동기 vs 비동기 클라이언트

`elasticsearch-py` 는 두 가지를 제공:
- **동기**: `from elasticsearch import Elasticsearch` (위 코드)
- **비동기**: `from elasticsearch import AsyncElasticsearch`

이 프로젝트는 동기 — ProcessPoolExecutor 워커가 동기 환경에서 호출하기 때문. async 라우터에서 직접 호출할 때는 AsyncElasticsearch 권장.

---

## 2. 클라이언트 설정 옵션

```python
config = {
    "hosts": ["http://host:9200"],
    "timeout": 60,
    "max_retries": 3,
    "retry_on_timeout": True,
    "http_auth": (user, password),
}
```

| 옵션 | 의미 |
|------|------|
| `hosts` | 노드 목록. 클러스터면 여러 개. 클라이언트가 라운드로빈 |
| `timeout` | 단일 요청 타임아웃 (초). 디폴트 10s — 큰 bulk 면 부족 |
| `max_retries` | 실패 시 재시도 횟수 |
| `retry_on_timeout` | 타임아웃도 재시도 대상으로 |
| `http_auth` | 기본 인증 (튜플) |
| `verify_certs` | TLS 인증서 검증 (기본 True) |
| `sniff_on_start` | 시작 시 클러스터 토폴로지 sniff |

### timeout 의 함정
- bulk 가 timeout 으로 실패해도 ES 측은 부분 처리 됐을 수 있음.
- 멱등하게 다시 보낼 수 있도록 `_id` 명시 필수.

---

## 3. 클러스터 헬스 체크

```python
health = self.client.cluster.health()
# health = {"status": "green", "number_of_nodes": 3, ...}

if health['status'] == 'red':
    logger.warning("일부 샤드가 할당 안 됨")
elif health['status'] == 'yellow':
    logger.warning("레플리카 샤드 미할당")
```

| 상태 | 의미 |
|------|------|
| green | 모든 primary + replica 활성 |
| yellow | primary 는 OK, replica 일부 미할당 |
| red | primary 일부 미할당 — 데이터 일부 접근 불가 |

운영 시작 시 yellow 는 단일 노드 클러스터에서 흔함 (replica 둘 곳이 없어서).

---

## 4. `helpers.parallel_bulk` — 병렬 bulk

```python
for success, info in helpers.parallel_bulk(
    self.client,
    actions,
    chunk_size=500,
    raise_on_error=False,
    thread_count=4
):
    if success: ...
    else: ...
```

### 인자
- `actions`: bulk action 제너레이터/리스트.
- `chunk_size`: 한 번의 bulk 호출에 포함할 액션 수. 보통 500~5000.
- `raise_on_error=False`: 한 문서 실패가 전체 중단을 의미하지 않게.
- `thread_count`: 클라이언트 쓰레드 풀 크기. 4~8 권장 (너무 크면 ES 측 큐 과부하).

### 반환
- 제너레이터. `(success, info)` 튜플을 yield.
- `info` 는 dict — 성공이면 `{"index": {"_id": ..., "result": "created"}}`, 실패면 error 정보 포함.

### `parallel_bulk` vs `bulk` vs `streaming_bulk`

| 함수 | 동작 |
|------|------|
| `bulk` | 단일 요청. 동기적 |
| `streaming_bulk` | 단일 thread, generator yield |
| `parallel_bulk` | 멀티 thread, 동시 chunk 전송 |

대량 색인은 `parallel_bulk` 가 throughput 최고.

---

## 5. action 구조

```python
action = {
    "_index": index_name,
    "_id": doc_id,
    "_source": doc_copy
}
```

세 가지 형식 (`_op_type` 으로 분기):
- 색인 (기본): `_op_type=index` 또는 생략
- 생성: `_op_type=create` (이미 있으면 실패)
- 업데이트: `_op_type=update` + `doc` 또는 `script`
- 삭제: `_op_type=delete` (`_source` 없음)

```python
{
    "_op_type": "update",
    "_index": "...",
    "_id": "123",
    "doc": {"field": "new_value"},
}
```

---

## 6. 실패 처리 패턴

위 코드는 chunk_size=500 으로 한 번에 500개씩 보냄. 실패한 문서는 `failed_docs` 에 누적.

이후 마이그레이션 코드에서 실패 분류:
```python
for failed_info in batch_failed_docs:
    if isinstance(failed_info, dict):
        if 'index' in failed_info:
            index_info = failed_info['index']
            doc_id = index_info.get('_id', 'unknown')
            error = index_info.get('error', {})
            error_type = error.get('type', 'unknown_error')
            error_reason = error.get('reason', '')
            if 'caused_by' in error:
                error_reason += f" (Caused by: {error['caused_by'].get('reason', '')})"
```

흔한 실패 타입:
- `mapper_parsing_exception` — 매핑 불일치 (string 컬럼에 dict 같은 거)
- `version_conflict_engine_exception` — 동시성 충돌
- `string_index_out_of_bounds` — 분석기 (analyzer) 버그
- `timeout_exception` — ES 측 과부하

→ 카테고리별 카운트 + 상위 N개 reason 로깅.

---

## 7. 매핑 (mapping) 의 중요성

색인 시작 전 매핑 정의:
```python
mapping = {
    "properties": {
        "trademark_name_kor": {
            "type": "text",
            "analyzer": "korean_analyzer",
            "fields": {
                "raw": {"type": "keyword"},
                "ngram": {"type": "text", "analyzer": "ngram_analyzer"}
            }
        },
        "application_date": {"type": "date", "format": "yyyy-MM-dd"},
        "applicant_country": {"type": "keyword"},
    }
}
self.client.indices.create(index=name, body={"mappings": mapping, "settings": {...}})
```

**multi-field**:
- `trademark_name_kor` 자체는 `text` (full-text 검색).
- `.raw` 는 `keyword` (정확 일치, aggregation).
- `.ngram` 은 부분 일치.

→ 한 컬럼을 용도별로 동시 색인.

상세 매핑/토크나이저는 [text-search-engine 폴더](../text-search-engine/) 참고.

---

## 8. 검색 — DSL

```python
query = {
    "query": {
        "bool": {
            "must": [
                {"match": {"trademark_name_kor": "삼성"}},
            ],
            "filter": [
                {"term": {"applicant_country": "KR"}},
                {"range": {"application_date": {"gte": "2020-01-01"}}}
            ]
        }
    },
    "size": 20,
    "from": 0,
    "sort": [{"application_date": "desc"}],
}
response = client.search(index="trademark", body=query)
```

핵심 차이:
- `must`: 점수 영향 (relevance scoring).
- `filter`: 점수 영향 없음 + 캐싱.
- 정확 일치는 항상 `filter` (성능).
- full-text 검색은 `must` (점수 사용).

---

## 9. Aggregations — 집계

```python
query = {
    "query": {"match_all": {}},
    "size": 0,
    "aggs": {
        "by_country": {
            "terms": {"field": "applicant_country", "size": 10}
        },
        "by_year": {
            "date_histogram": {"field": "application_date", "calendar_interval": "year"}
        }
    }
}
response = client.search(index="trademark", body=query)
buckets = response["aggregations"]["by_country"]["buckets"]
```

`size: 0` — 문서 자체는 안 가져오고 집계만 (성능).

---

## 10. 응용 포인트

- 대량 색인은 `helpers.parallel_bulk` + chunk_size 500~5000 + thread_count 4~8.
- timeout 은 길게 (60초+), `retry_on_timeout=True`.
- `_id` 를 명시해서 idempotent 색인 보장 (timeout 후 재시도 안전).
- 실패 응답은 dict 형태 — 카테고리별 분류 후 운영 메트릭화.
- 매핑은 multi-field 로 한 컬럼을 여러 용도로.
- 정확 일치 / 범위는 `filter`, full-text 는 `must`.
- `size: 0` + aggs 로 집계 전용 쿼리.
- async 라우터면 AsyncElasticsearch 사용.
