# Singleton Pattern — DB 인스턴스

> 프로젝트: search-platform

---

## 해결한 문제

기존에는 국가별 DB 클래스가 24개 파일로 분리되어 있었고, 동일한 ES/MariaDB 연결 로직이 중복 구현되어 있었다. 연결 관리도 산발적이었다.

```
# Before — 24개 DB 파일
database/ko/es_database.py      # KO ES 연결
database/us/es_database.py      # US ES 연결 (거의 동일한 코드)
database/jp/es_database.py
database/ko/maria_database.py
# ... × 5개국
```

---

## 구조

```
instances.py (모듈 레벨 싱글톤)
  ├── ko_es_db = ElasticsearchDatabase(CountryCode.KR)
  ├── us_es_db = ElasticsearchDatabase(CountryCode.US)
  ├── jp_es_db = ElasticsearchDatabase(CountryCode.JP)
  ├── ch_es_db = ElasticsearchDatabase(CountryCode.CH)
  ├── eu_es_db = ElasticsearchDatabase(CountryCode.EU)
  ├── ko_maria_db = MariaDatabase(CountryCode.KR)       # KO만
  └── subcode_es_db = SubcodeElasticsearchDatabase()     # KO만

ElasticsearchDatabase (통합 1개 클래스, 536줄)
  ├── __init__(country) → 설정 로드 + sync/async 연결
  ├── search() / async_search()
  ├── detailed_search() / async_detailed_search()
  ├── aggregate_search()
  ├── get_document() / bulk_search() / count_documents()
  └── health_check() / close()

MariaDatabase (통합 1개 클래스, 245줄)
  ├── __init__(country) → 설정 로드 + sync/async 연결
  ├── execute_query() / async_execute_query()
  ├── execute_scalar() / execute_insert_update()
  ├── get_session() (context manager)
  └── health_check() / close()
```

---

## 핵심 코드

### 싱글톤 인스턴스 생성 (`infrastructure/database/instances.py`)

```python
# 모듈 로드 시 1회 생성 — 이후 동일 객체 재사용
from server.search.infrastructure.database.elasticsearch_database import ElasticsearchDatabase
from server.search.infrastructure.database.maria_database import MariaDatabase

ko_es_db = ElasticsearchDatabase(CountryCode.KR)
us_es_db = ElasticsearchDatabase(CountryCode.US)
jp_es_db = ElasticsearchDatabase(CountryCode.JP)
ch_es_db = ElasticsearchDatabase(CountryCode.CH)
eu_es_db = ElasticsearchDatabase(CountryCode.EU)

ko_maria_db = MariaDatabase(CountryCode.KR)
subcode_es_db = SubcodeElasticsearchDatabase()
```

### 통합 ES 클래스 (`infrastructure/database/elasticsearch_database.py`)

```python
class ElasticsearchDatabase:
    """5개국 공용 Elasticsearch 클라이언트"""

    def __init__(self, country: CountryCode):
        self.country = country
        self.es_config = get_es_config(self.country)     # 국가별 설정(호스트, 포트) 로드
        self.indexes = get_es_indexes(self.country)       # 국가별 인덱스명 로드
        self.client: Optional[Elasticsearch] = None
        self.async_client: Optional[AsyncElasticsearch] = None
        self._connect()
        self._async_connect()

    def _connect(self):
        self.client = Elasticsearch(**self.es_config)
        if self.client.ping():
            self.logger.info(f"{self.country_name} ES 연결 성공")

    def _async_connect(self):
        self.async_client = AsyncElasticsearch(**self.es_config)

    async def async_search(self, index_name, query, size=10, from_=0):
        self._validate_async_connection()
        self._validate_index(index_name)
        return await self.async_client.search(
            index=index_name, body=query,
            size=size, from_=from_,
            track_total_hits=True, timeout="30s"
        )

    async def async_detailed_search(self, index_name, query_body):
        """상세검색 전용 — 스코어 디버깅 로깅 포함"""
        ...
```

### 통합 MariaDB 클래스 (`infrastructure/database/maria_database.py`)

```python
class MariaDatabase:
    """KO 전용 MariaDB 클라이언트"""

    def __init__(self, country: CountryCode):
        self.db_url = get_db_url(self.country)
        self.schema = settings.get_db_schema(self.country)
        # SQLAlchemy 동기 엔진
        self.engine = create_engine(self.db_url, pool_size=10, max_overflow=20)
        self.SessionLocal = sessionmaker(bind=self.engine)
        # SQLAlchemy 비동기 엔진 (aiomysql)
        async_url = self.db_url.replace('mysql+pymysql', 'mysql+aiomysql')
        self.async_engine = create_async_engine(async_url, pool_size=10, max_overflow=20)
        self.AsyncSessionLocal = async_sessionmaker(self.async_engine)

    @contextmanager
    def get_session(self):
        session = self.SessionLocal()
        try:
            yield session
            session.commit()
        except Exception:
            session.rollback()
            raise
        finally:
            session.close()
```

### CountryComponents에서 주입

```python
# domain/ko/components.py
def create_components() -> CountryComponents:
    return CountryComponents(
        ...,
        es_db=ko_es_db,           # 싱글톤 인스턴스 주입
        maria_db=ko_maria_db,     # 싱글톤 인스턴스 주입
        subcode_es_db=subcode_es_db,
    )
```

---

## 효과

| Before | After |
|--------|-------|
| 24개 DB 파일 (5국가별 중복) | **통합 2개 클래스** (ES 536줄 + Maria 245줄) |
| 국가별 연결 로직 중복 | 생성자에 `country` 전달 → 설정 자동 분기 |
| 연결 인스턴스 산발적 관리 | `instances.py`에서 모듈 레벨 1회 생성 |
| sync/async 별도 구현 | 1개 클래스에 동기+비동기 메서드 공존 |
| 커넥션 풀 설정 중복 | `pool_size=10, max_overflow=20` 1곳에서 관리 |
