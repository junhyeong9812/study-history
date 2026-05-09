# Template Method Pattern — BaseDetailSearchQueryBuilder

> 프로젝트: search-platform

---

## 해결한 문제

5개국의 상세검색/전문검색 쿼리 빌드 과정은 **골격은 동일하되 세부 단계만 다르다**: 공통으로 base 쿼리 구성 → 필터 적용 → 정렬 적용 → 옵션(min_score, source_includes) 적용 → 최종 쿼리 조립. 기존에는 이 골격이 5개국에 각각 복사되어 있었다.

---

## 구조

```
BaseDetailSearchQueryBuilder (ABC)          ← 템플릿 (골격 정의)
  │
  │  build_search_query(request)            ← 공통 알고리즘 (변경 불가)
  │    ├── _build_base_query(request)       ← 추상 (국가별 필수 구현)
  │    ├── _build_filters(request)          ← 추상 (국가별 필수 구현)
  │    ├── _build_sort(request, has_tm)     ← 추상 (국가별 필수 구현)
  │    ├── _get_min_score(request)          ← 훅 (기본 None, 필요 시 오버라이드)
  │    └── _get_source_includes()           ← 훅 (기본 None, 필요 시 오버라이드)
  │
  ├── KoDetailedSearchBuilder              ← KO 구현 (4개 추상 + 2개 훅 오버라이드)
  ├── UsDetailedSearchQueryBuilder         ← US 구현 (4개 추상 + 1개 훅 오버라이드)
  ├── JpDetailedSearchQueryBuilder         ← JP 구현
  ├── EuDetailedSearchQueryBuilder         ← EU 구현
  └── CnDetailedSearchQueryBuilder         ← CN 구현
```

---

## 핵심 코드

### 베이스 템플릿 (`domain/common/query_builder/base_search_query_builder.py`)

```python
class BaseDetailSearchQueryBuilder(ABC):
    """상세검색 쿼리 빌더 — 공통 템플릿"""

    # === 추상 메서드: 국가별 반드시 구현 ===
    @staticmethod
    @abstractmethod
    def _country_code() -> str: ...

    @staticmethod
    @abstractmethod
    def _build_base_query(request) -> Dict: ...

    @classmethod
    @abstractmethod
    def _build_filters(cls, request) -> Dict: ...

    @classmethod
    @abstractmethod
    def _build_sort(cls, request, has_trademark_name: bool) -> Dict: ...

    # === 훅 메서드: 기본값 제공, 필요 시 오버라이드 ===
    @classmethod
    def _get_min_score(cls, request) -> Optional[float]:
        return None  # KO/CN만 오버라이드

    @classmethod
    def _get_source_includes(cls) -> Optional[List[str]]:
        return None  # KO/US만 오버라이드

    # === 템플릿 메서드: 알고리즘 골격 (변경 불가) ===
    @classmethod
    def build_search_query(cls, request) -> Dict:
        label = cls._country_code()
        logger.info(f"[{label}] 검색 쿼리 생성")

        base_query = cls._build_base_query(request)          # Step 1
        filter_query = cls._build_filters(request)            # Step 2
        has_tm = cls._has_trademark_name(request)
        sort_query = cls._build_sort(request, has_tm)         # Step 3
        min_score = cls._get_min_score(request)               # Step 4 (훅)
        source_includes = cls._get_source_includes()          # Step 5 (훅)

        builder = SearchQueryBuilder()
        kwargs = dict(
            base_query=base_query,
            filter_query=filter_query,
            size=request.size,
            from_offset=(request.page - 1) * request.size,
            sort_query=sort_query,
            track_total_hits=True,
            track_scores=True,
        )
        if source_includes:
            kwargs["source_includes"] = source_includes
        if min_score:
            kwargs["min_score"] = min_score

        return builder.build(**kwargs)
```

### KO 구현 — 추상 메서드 + 훅 오버라이드

```python
class KoDetailedSearchBuilder(KODetailedSearchBase, BaseDetailSearchQueryBuilder):

    @staticmethod
    def _country_code() -> str:
        return "KO"

    @classmethod
    def _build_filters(cls, request: KODetailParameter) -> Dict:
        return FilterConditions.build_detail_filters(request=request)

    @classmethod
    def _build_sort(cls, request: KODetailParameter, has_trademark_name: bool) -> Dict:
        return SortConditions.build_unified_sort_query(
            sort_field=request.sort_field.value,
            sort_order=request.sort_order.value,
            has_keyword=has_trademark_name,
            keyword_text=request.trademark_name.strip() if request.trademark_name else "",
            search_type=request.search_type.value,
            first_flag=request.first_flag
        )

    @classmethod
    def _get_min_score(cls, request: KODetailParameter) -> Optional[float]:
        if request.search_type.value == "similar_sound" and request.trademark_name:
            return 45  # KO 유사 발음 검색 시 최소 스코어
        return None

    @classmethod
    def _get_source_includes(cls) -> Optional[List[str]]:
        return INCLUDE_DATA  # KO 전용 필드 목록
```

### US 구현 — 다른 필터/정렬 전략

```python
class UsDetailedSearchQueryBuilder(DetailedSearchBase, BaseDetailSearchQueryBuilder):

    @staticmethod
    def _country_code() -> str:
        return "US"

    @classmethod
    def _build_filters(cls, request: USDetailParameter) -> Dict:
        return USFilterConditions.build_filters(request)  # US 전용 필터

    @classmethod
    def _build_sort(cls, request: USDetailParameter, has_trademark_name: bool) -> Dict:
        return USSortConditions.build_sort_query(
            sort_field=request.sort_field.value,
            sort_order=request.sort_order.value,
            has_trademark_name=has_trademark_name,
            script_dict=request.sort_script,  # US 전용: 스크립트 정렬
            first_flag=request.first_flag
        )

    # _get_min_score: 오버라이드 안 함 → 기본값 None
    
    @classmethod
    def _get_source_includes(cls) -> Optional[List[str]]:
        return US_INCLUDE_DATA  # US 전용 필드 목록
```

---

## 국가별 오버라이드 현황

| 메서드 | 유형 | KO | US | JP | EU | CN |
|--------|:----:|:--:|:--:|:--:|:--:|:--:|
| `_country_code()` | 추상 | O | O | O | O | O |
| `_build_base_query()` | 추상 | O | O | O | O | O |
| `_build_filters()` | 추상 | O | O | O | O | O |
| `_build_sort()` | 추상 | O | O | O | O | O |
| `_get_min_score()` | 훅 | O (45) | — | — | — | O |
| `_get_source_includes()` | 훅 | O | O | — | — | — |

---

## 효과

| Before | After |
|--------|-------|
| 쿼리 빌드 골격이 5개국에 복사 | 골격 1곳 (Base), 세부만 국가별 구현 |
| 빌드 순서 변경 시 5곳 수정 | Base 템플릿 1곳만 수정 |
| 훅(min_score 등) 미사용 국가도 코드 존재 | 기본값 제공, 필요한 국가만 오버라이드 |
| 새 빌드 단계 추가 시 5곳 수정 | Base에 단계 추가 → 전 국가 자동 반영 |
