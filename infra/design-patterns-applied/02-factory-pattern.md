# Factory Pattern — ParameterFactory

> 프로젝트: search-platform

---

## 해결한 문제

기존에는 5개국 × 4종 검색 = **20개 요청 스키마**가 각각 별도 파일로 중복 존재했다. 유효성 검증 로직도 국가마다 복사되어 있어 공통 필드 변경 시 20곳을 수정해야 했다.

```
# Before — schemas/ 디렉토리에 78개 중복 파일
schemas/ko/expert_search_request.py
schemas/us/expert_search_request.py
schemas/jp/expert_search_request.py   # 거의 동일한 내용
schemas/eu/expert_search_request.py
schemas/ch/expert_search_request.py
```

---

## 구조

```
ParameterFactory
  ├── create_free(country, raw)    → BaseFreeParameter 계열 객체
  ├── create_detail(country, raw)  → BaseDetailParameter 계열 객체
  ├── create_expert(country, raw)  → BaseExpertParameter 계열 객체
  └── create_expert_text(country, raw) → BaseExpertTextParameter 계열 객체

Base 클래스 계층:
  BaseFreeParameter (Pydantic BaseModel)
    ├── KOFreeParameter (pass — 차이 없음)
    ├── USFreeParameter (pass)
    ├── JPFreeParameter (pass)
    ├── EUFreeParameter (pass)
    └── CNFreeParameter (pass)

  BaseDetailParameter (Pydantic BaseModel, ~169줄)
    ├── KODetailParameter (KO 전용 필드 추가)
    ├── USDetailParameter (US 전용 정렬 필드)
    └── ...
```

---

## 핵심 코드

### 팩토리 (`domain/common/parameter_factory.py`)

```python
class ParameterFactory:
    _free = {
        Country.KO: KOFreeParameter,
        Country.US: USFreeParameter,
        Country.EU: EUFreeParameter,
        Country.JP: JPFreeParameter,
        Country.CN: CNFreeParameter,
    }
    _detail = { ... }  # 동일 구조
    _expert = { ... }
    _expert_text = { ... }

    @classmethod
    def create_free(cls, country: Country, raw: dict) -> BaseFreeParameter:
        return cls._free[country](**raw)

    @classmethod
    def create_detail(cls, country: Country, raw: dict) -> BaseDetailParameter:
        return cls._detail[country](**raw)
```

### 베이스 파라미터 (`domain/common/parameter/base_free.py`)

```python
class BaseFreeParameter(BaseModel):
    """5개국 공통 자유검색 파라미터"""
    trademark_name: Optional[str] = Field(None)
    assignProductMainCodes: Optional[List[str]] = Field(None)

    @field_validator('trademark_name', mode='before')
    @classmethod
    def validate_trademark_name(cls, v):
        return empty_str_to_none(v)

    def has_search_conditions(self) -> bool:
        return bool(self.trademark_name or self.assignProductMainCodes)

    def get_validation_error_message(self) -> Optional[str]:
        if not self.has_search_conditions():
            return "키워드 또는 상품 분류코드 중 하나 이상을 입력해주세요."
        return None
```

### 국가별 오버라이드 — 차이 없으면 pass

```python
# domain/ko/parameter/free.py
class KOFreeParameter(BaseFreeParameter):
    """한국(KO) 자유검색 파라미터"""
    pass

# domain/us/parameter/free.py
class USFreeParameter(BaseFreeParameter):
    """미국(US) 자유검색 파라미터"""
    pass
```

### 국가별 오버라이드 — 차이 있으면 확장

```python
# domain/ko/parameter/detail.py (예시)
class KODetailParameter(BaseDetailParameter):
    """한국 상세검색 — KO 전용 필드 추가"""
    similar_code: Optional[str] = None
    applicant_code: Optional[str] = None
    # BaseDetailParameter의 모든 공통 필드 + 유효성 검증은 상속
```

---

## 사용 예시

```python
# SearchService에서
param = ParameterFactory.create_detail(Country.US, raw_dict)
# → USDetailParameter(**raw_dict) 호출
# → Pydantic이 자동으로 타입 검증, 기본값 설정, validator 실행
```

---

## 효과

| Before | After |
|--------|-------|
| 78개 스키마 파일 (대부분 중복) | 공통 Base 4개 + 국가별 오버라이드 (차이점만) |
| 공통 필드 수정 시 20곳 수정 | Base 수정 1곳 → 5개국 자동 반영 |
| 유효성 검증 로직 중복 | Base에서 1회 정의, Pydantic `@field_validator` 상속 |
| raw dict 직접 사용 (타입 안전성 없음) | `ParameterFactory.create_*(country, raw)` → 타입화된 객체 |
