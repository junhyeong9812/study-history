# Facade Pattern — CountryComponents

> 프로젝트: search-platform

---

## 해결한 문제

각 국가의 검색 시스템은 내부적으로 10개 이상의 쿼리 빌더, 다수의 정규화 함수, 파서 설정, 보고서 생성기 등 복잡한 서브시스템으로 구성된다. 서비스 계층이 이 내부 구조를 직접 알 필요 없이, **5개의 Facade 클래스**가 단순한 인터페이스를 제공한다.

---

## 구조

```
SearchService
    │
    └── c = DomainRegistry.get(country)
         │
         ├── c.validator           ← Validator Facade
         ├── c.query_builder       ← QueryBuilderFacade  ← 10+ 내부 빌더를 감춤
         ├── c.query_parser        ← QueryParserFacade   ← 파서 + 설정을 감춤
         ├── c.normalizer          ← Normalizer Facade   ← 5+ 정규화 함수를 감춤
         └── c.report_generator    ← ReportGenerator Facade
```

---

## 핵심 코드

### QueryBuilderFacade — 10+ 빌더를 하나의 인터페이스로

```python
# domain/ko/components.py
class QueryBuilderFacade:
    """KO의 10+ 쿼리 빌더를 단일 인터페이스로 제공"""

    def __init__(self):
        self._vienna_qb = ViennaQueryBuilder(ko_maria_db.get_schema_name())

    # 내부의 KoFreeSearchQueryBuilder를 감춤
    def build_free_search(self, param):
        return KoFreeSearchQueryBuilder.build_search_query(param)

    # 내부의 KoDetailedSearchBuilder를 감춤
    def build_detailed_search(self, param):
        return KoDetailedSearchBuilder.build_search_query(param)

    # 내부의 KoDetailedSearchFilterBuilder를 감춤
    def build_detailed_search_filter(self, param):
        return KoDetailedSearchFilterBuilder.build_filter_aggregation_query(param)

    # 내부의 KoExpertSearchQueryBuilder를 감춤
    def build_expert_search(self, param):
        return KoExpertSearchQueryBuilder.build_search_query(param)

    # 내부의 SingleApplicationQueryBuilder를 감춤
    def build_single_application(self, app_number):
        return SingleApplicationQueryBuilder.build_search_query(app_number)

    # 내부의 ApplicationNumberQueryBuilder를 감춤
    def build_application_number_search(self, numbers, sort_field, sort_order):
        return ApplicationNumberQueryBuilder.build_ids_query(numbers, sort_field, sort_order)

    # 내부의 ReportSearchQueryBuilder를 감춤
    def build_report_search(self, numbers):
        return ReportSearchQueryBuilder.build_search_query(numbers)

    # 내부의 ViennaQueryBuilder를 감춤 (MariaDB 기반)
    def build_vienna_count_query(self, keyword):
        return self._vienna_qb.build_count_query(keyword)

    def build_vienna_list_query(self, keyword, sort_field, sort_order, page, size):
        return self._vienna_qb.build_list_query(keyword, sort_field, sort_order, page, size)
```

### QueryParserFacade — 파서 + 국가별 설정을 감춤

```python
class QueryParserFacade:
    """통합 파서 + KO 파서 설정을 하나의 인터페이스로"""

    def __init__(self):
        from server.search.domain.common.query_parser import QueryParser
        from server.search.domain.ko.expert.ko_parser_config import create_ko_config
        self._parser = QueryParser(create_ko_config())  # 내부 복잡성 감춤

    def parse(self, query_string: str) -> dict:
        parsed_filter = self._parser.parse(query_string)
        result = parsed_filter.model_dump()
        # has_trademark_name 판정 로직도 감춤
        if result.get("trademark_name"):
            result["has_trademark_name"] = True
        elif result.get("query_ast"):
            result["has_trademark_name"] = self._ast_has_trademark_name(result["query_ast"])
        return result

    def parse_expert_query(self, raw: dict) -> dict:
        """하위 호환: parse() + pagination 병합"""
        result = self.parse(raw.get("query_string", ""))
        for key in ("page", "size", "sort_field", "sort_order", "search_type"):
            if key in raw:
                result[key] = raw[key]
        return result
```

### create_components() — 모든 Facade를 CountryComponents로 조립

```python
def create_components() -> CountryComponents:
    return CountryComponents(
        validator=Validator(),
        query_builder=QueryBuilderFacade(),
        query_parser=QueryParserFacade(),
        normalizer=Normalizer(),
        report_generator=ReportGenerator(),
        es_db=ko_es_db,
        customer_es_db=ko_es_db,
        customer_index_name=settings.ES_INDEX_KR_CUSTOMER,
        subcode_es_db=subcode_es_db,
        maria_db=ko_maria_db,
    )
```

### 서비스에서 사용 — 내부 구조를 모른 채 호출

```python
# SearchService — QueryBuilderFacade 뒤의 10+ 빌더를 알 필요 없음
query = c.query_builder.build_detailed_search(param)
filter_query = c.query_builder.build_detailed_search_filter(param)
report_query = c.query_builder.build_report_search(numbers)
```

---

## 국가별 Facade 내부 복잡도 비교

| 국가 | QueryBuilder 내부 빌더 수 | Normalizer 내부 함수 수 | 추가 인프라 |
|------|:----------------------:|:--------------------:|-----------|
| KO | 12+ (ES 7종 + Maria 3종 + 부가) | 8+ (검색 + 출원인 + 대리인 + 서브코드) | MariaDB, SubcodeES |
| US | 8+ (ES 7종 + 유사검색) | 5+ (검색 + 유사검색) | — |
| CN | 8+ (ES 7종 + 유사검색 + 결과 제한 체크) | 5+ (검색 + 유사검색) | — |
| JP/EU | 7 (ES 기본 7종) | 4 (검색 기본) | — |

---

## 효과

| Before | After |
|--------|-------|
| 서비스가 내부 빌더 클래스를 직접 import | Facade 메서드 1개 호출로 충분 |
| 빌더 이름/경로 변경 시 서비스 수정 필요 | Facade 내부만 수정, 서비스 영향 없음 |
| 파서 초기화 + 설정 로딩이 호출부에 노출 | Facade `__init__`에서 1회 처리 |
| 국가 추가 시 서비스 로직에 분기 추가 | 새 국가의 components.py에 Facade 구현만 추가 |
