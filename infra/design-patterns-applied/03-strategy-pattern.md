# Strategy Pattern — 국가별 검색 전략

> 프로젝트: search-platform

---

## 해결한 문제

기존에는 각 국가의 base 파일(1,000~2,000줄)에 검증, 쿼리 빌딩, 파싱, 결과 정규화가 모두 혼재되어 있었다. 동일한 인터페이스이면서도 국가마다 다른 세부 로직이 하나의 거대 파일에 뒤섞여 역할 파악과 변경이 어려웠다.

```
# Before — 1,980줄짜리 파일에 모든 역할 혼재
ko/base/unified_expert_search_base.py  ← 파싱 + 쿼리 빌드 + 필드 매핑 + 결과 추출
ch/base/expert_search_base.py          ← 1,572줄, 동일 구조
```

---

## 구조

SearchService는 **국가 코드만으로** 적절한 전략을 선택한다. 각 국가의 components.py가 동일한 인터페이스(Validator, QueryBuilderFacade, QueryParserFacade, Normalizer, ReportGenerator)를 구현한다.

```
SearchService.detailed_search(country=US, raw)
    │
    ▼
DomainRegistry.get(Country.US) → CountryComponents
    │
    ├── c.validator.validate_detailed_search(param)     ← US Validator 전략
    ├── c.query_builder.build_detailed_search(param)    ← US QueryBuilder 전략
    ├── c.es_db.async_detailed_search(...)              ← US ES 인스턴스
    └── c.normalizer.normalize_detailed_search(hits)    ← US Normalizer 전략
```

---

## 핵심 코드

### KO 전략 (`domain/ko/components.py`)

```python
class Validator:
    def validate_free_search(self, param: KOFreeParameter):
        error_message = param.get_validation_error_message()
        if error_message:
            raise SearchValidationError(error_message)

    def validate_detailed_search(self, param: KODetailParameter):
        pass  # Pydantic이 이미 검증 완료 — KO는 추가 룰 없음

class QueryBuilderFacade:
    def __init__(self):
        self._vienna_qb = ViennaQueryBuilder(ko_maria_db.get_schema_name())

    def build_detailed_search(self, param):
        return KoDetailedSearchBuilder.build_search_query(param)

    # KO 전용: 출원인, 대리인, 도형코드 검색
    def build_applicant_search(self, request): ...
    def build_agent_search(self, request): ...
    def build_vienna_count_query(self, keyword): ...

class Normalizer:
    def normalize_detailed_search(self, hits):
        return normalize_ko_es_list(hits)

    def extract_highlight_keywords(self, param, mode="detail"):
        return HighlightKeywords.from_detailed_search_ko(param).model_dump(...)

    # KO 전용: 출원인/대리인/서브코드 정규화
    def normalize_applicant_list(self, hits): ...
    def normalize_agent_list(self, hits): ...

class ReportGenerator:
    def generate(self, sources, title, target, lang='ko'):
        if target == "excel":
            wb = create_excel_report(sources, title, lang)  # KO 보고서 생성기
            return upload_excel_to_s3(wb, ...)
```

### US 전략 (`domain/us/components.py`) — 같은 인터페이스, 다른 구현

```python
class Validator:
    def validate_detailed_search(self, param: USDetailParameter):
        if not param.has_search_conditions():
            raise SearchValidationError("검색 조건을 1개 이상 입력해주세요")
        # KO와 다른 검증 규칙

class QueryBuilderFacade:
    def build_detailed_search(self, param):
        return UsDetailedSearchQueryBuilder.build_search_query(param)

    # US 전용: 등록번호 검색, 유사 검색 (한글→영어 변환 포함)
    def build_single_registration(self, reg_number):
        return UsSingleRegistrationQueryBuilder.build_term_query(reg_number)

    def build_similar_search(self, raw):
        korean_converter = KoreanToEnglishConverter()
        origin_request = USSimplifiedSearchRequest(**raw)
        request = origin_request.to_detail()
        if korean_converter.is_korean(request.trademark_name):
            request.trademark_name = korean_converter.convert_for_trademark_search(request.trademark_name)
        return UsDetailedSearchQueryBuilder.build_search_query(request)

class Normalizer:
    def normalize_detailed_search(self, hits):
        return normalize_us_es_list(hits)  # US 필드 매핑으로 정규화

    def normalize_similar_search(self, hits):
        # US 전용: 유사 검색 결과에 score, status 매핑 포함
        ...
```

### 국가별 전략 차이 요약

| 컴포넌트 | KO | US | CN |
|---------|----|----|-----|
| **Validator** | Pydantic으로 충분 (pass) | `has_search_conditions()` 체크 | `has_search_conditions()` 체크 |
| **QueryBuilder 전용 메서드** | 출원인, 대리인, 도형코드 | 등록번호, 유사검색 | 유사검색 + 결과 제한 체크 |
| **Normalizer 전용 메서드** | 출원인/대리인/서브코드 정규화 | 유사검색 정규화 | 유사검색 정규화 |
| **하이라이트 필드** | 7개 (similar_code 포함) | 7개 (vienna_code 포함) | 7개 (trademark_pinyin 포함) |
| **DB 인스턴스** | ES + MariaDB + SubcodeES | ES만 | ES만 |

---

## 효과

| Before | After |
|--------|-------|
| 1,000~2,000줄 base 파일에 모든 역할 혼재 | 역할별 ~200줄 전략 클래스로 분리 |
| 국가별 차이 파악에 전체 파일 비교 필요 | components.py 비교로 차이점 즉시 파악 |
| 공통 로직 변경 시 5개국 각각 수정 | 공통 인터페이스는 SearchService에서 1곳 |
| 국가별 전용 기능 추가 시 기존 코드 오염 | 해당 국가 components.py에만 메서드 추가 |
