# Registry Pattern — DomainRegistry

> 프로젝트: search-platform

---

## 해결한 문제

기존에는 5개국 라우터가 하드코딩으로 분리되어 있었다. 국가를 추가하거나 변경할 때마다 main.py, 라우터, 서비스 로직 등 여러 곳을 수정해야 했다.

```python
# Before — main.py에 5개국 라우터 하드코딩
from server.router.ko.ko_router import router as ko_router
from server.router.us.us_router import router as us_router
from server.router.jp.jp_router import router as jp_router
# ... 국가 추가 시 여기도 수정 필요
```

---

## 구조

```
DomainRegistry (class-level dict)
  ├── Country.KO → CountryComponents(validator, query_builder, query_parser, normalizer, report_generator, es_db, maria_db, ...)
  ├── Country.US → CountryComponents(validator, query_builder, query_parser, normalizer, report_generator, es_db)
  ├── Country.JP → CountryComponents(...)
  ├── Country.EU → CountryComponents(...)
  └── Country.CN → CountryComponents(...)
```

---

## 핵심 코드

### 레지스트리 정의 (`domain/registry.py`)

```python
class CountryComponents:
    """한 국가의 모든 인프라·비즈니스 컴포넌트를 묶는 컨테이너"""
    def __init__(self, validator, query_builder, query_parser,
                 normalizer, report_generator, es_db,
                 customer_es_db=None, customer_index_name=None,
                 subcode_es_db=None, maria_db=None):
        self.validator = validator
        self.query_builder = query_builder
        self.query_parser = query_parser
        self.normalizer = normalizer
        self.report_generator = report_generator
        self.es_db = es_db
        self.customer_es_db = customer_es_db          # KO only
        self.customer_index_name = customer_index_name  # KO only
        self.subcode_es_db = subcode_es_db              # KO only
        self.maria_db = maria_db                        # KO only

class DomainRegistry:
    _registry: Dict[Country, CountryComponents] = {}

    @classmethod
    def register(cls, country, components):
        cls._registry[country] = components

    @classmethod
    def get(cls, country) -> CountryComponents:
        if country not in cls._registry:
            raise ValueError(f"미등록 국가: {country}")
        return cls._registry[country]
```

### 5개국 등록 (`search/__init__.py`)

```python
from server.search.domain.ko.components import create_components as ko_components
from server.search.domain.us.components import create_components as us_components
from server.search.domain.jp.components import create_components as jp_components
from server.search.domain.eu.components import create_components as eu_components
from server.search.domain.cn.components import create_components as ch_components

DomainRegistry.register(Country.KO, ko_components())
DomainRegistry.register(Country.US, us_components())
DomainRegistry.register(Country.JP, jp_components())
DomainRegistry.register(Country.EU, eu_components())
DomainRegistry.register(Country.CN, ch_components())
```

### 서비스에서 사용 (`application/search_service.py`)

```python
class SearchService:
    @staticmethod
    async def detailed_search(country: Country, raw: dict):
        c = DomainRegistry.get(country)           # 1. 국가별 컴포넌트 조회
        param = ParameterFactory.create_detail(country, raw)
        c.validator.validate_detailed_search(param)  # 2. 유효성 검증
        query = c.query_builder.build_detailed_search(param)  # 3. 쿼리 빌드
        result = await c.es_db.async_detailed_search(index, query)  # 4. 검색 실행
        normalized = c.normalizer.normalize_detailed_search(hits)   # 5. 결과 정규화
        return normalized
```

11개 서비스 메서드가 모두 `DomainRegistry.get(country)` → 컴포넌트 사용 패턴을 따른다.

---

## 효과

| Before | After |
|--------|-------|
| 국가 추가 시 main.py, 라우터, 서비스 등 N곳 수정 | `create_components()` 구현 + `register()` 1줄 추가 |
| 서비스 로직이 국가별 분기로 복잡 | `c = DomainRegistry.get(country)` 한 줄로 디스패치 |
| 국가별 DB/검색/정규화 의존성 산발적 | `CountryComponents`에 9개 컴포넌트로 묶어서 주입 |
