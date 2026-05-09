# Pydantic 모델로 API 스키마 설계

> Request/Response 모델로 OpenAPI 문서, 검증, IDE 타입 힌트를 한꺼번에 얻는 패턴.

---

## 0. 분석 대상 코드

마이그레이션 라우터의 Request/Response 모델 모음.

```python
# example-app/routers/migration_router.py
from pydantic import BaseModel, Field
from typing import Optional, Dict, Any, List


class PrepareApplicationsRequest(BaseModel):
    """출원번호 파일 생성 요청"""
    force_recreate: bool = Field(False, description="기존 파일 덮어쓰기 여부")
    batch_size: int = Field(50000, description="DB 조회 배치 크기", ge=1000, le=100000)
    updated_after: Optional[str] = Field(None, description="이 일자 이후 업데이트된 데이터만 조회 (YYYY-MM-DD)")


class PrepareApplicationsResponse(BaseModel):
    """출원번호 파일 생성 응답"""
    status: str = Field(..., description="작업 상태")
    total_count: int = Field(..., description="총 출원번호 개수")
    file_path: str = Field(..., description="생성된 파일 경로")
    file_size_mb: float = Field(..., description="파일 크기 (MB)")
    execution_time: float = Field(..., description="실행 시간 (초)")
    updated_after: Optional[str] = Field(None, description="조회 기준 업데이트 일자")


class MigrationStartRequest(BaseModel):
    """마이그레이션 시작 요청"""
    start_index: int = Field(0, description="시작 인덱스", ge=0)
    end_index: Optional[int] = Field(None, description="종료 인덱스")
    batch_size: int = Field(1000, description="배치 크기", ge=100, le=10000)
    use_parallel: bool = Field(True, description="병렬 처리 사용 여부")
    max_workers: int = Field(10, description="병렬 처리 워커 수", ge=1, le=20)
    resume: bool = Field(False, description="체크포인트에서 재개 여부")


class MigrationStatusResponse(BaseModel):
    """마이그레이션 상태 응답"""
    is_running: bool = Field(..., description="실행 중 여부")
    use_parallel: bool = Field(..., description="병렬 처리 사용 여부")
    max_workers: int = Field(..., description="프로세스 풀 크기")
    current_batch: int = Field(..., description="현재 완료된 배치")
    total_batches: int = Field(..., description="전체 배치")
    processed: int = Field(..., description="처리된 건수")
    success: int = Field(..., description="성공 건수")
    failed: int = Field(..., description="실패 건수")
    progress_percentage: float = Field(..., description="진행률 (%)")
    elapsed_time: Optional[float] = Field(None, description="경과 시간 (초)")
    estimated_remaining: Optional[float] = Field(None, description="예상 남은 시간 (초)")
    processing_rate: Optional[float] = Field(None, description="처리 속도 (docs/sec)")
    errors: list = Field(default_factory=list, description="최근 에러 목록")
    success_file: Optional[str] = Field(None, description="성공 파일 경로")
    failed_file: Optional[str] = Field(None, description="실패 파일 경로")
```

---

## 1. `Field(...)` 의 진짜 역할

`Field` 는 단순한 기본값보다 훨씬 많은 일을 한다.

| 인자 | 의미 |
|------|------|
| `default` (첫 위치 인자) | 기본값. `...` (`Ellipsis`) 면 **필수**. |
| `description` | OpenAPI 문서에 노출되는 설명 |
| `ge` / `le` | 숫자 최소/최대 (≥, ≤) |
| `gt` / `lt` | 강한 부등호 (>, <) |
| `min_length` / `max_length` | 문자열·리스트 길이 제약 |
| `pattern` | 정규식 매칭 |
| `default_factory` | mutable 기본값 (list/dict) |
| `alias` | JSON 필드명과 파이썬 속성명을 다르게 |
| `examples` | OpenAPI 예시 값 |

### 1.1 `...` vs `default`

```python
status: str = Field(..., description="...")     # 필수
status: str = Field("ok", description="...")    # 기본값 "ok"
status: Optional[str] = Field(None, ...)        # 옵션, 기본값 None
```

`Ellipsis` 는 파이썬 기본 객체로 "값 없음과 다른, 의도적으로 필수"를 표현하는 관용구.

### 1.2 `default_factory` 의 함정 회피

```python
errors: list = Field(default_factory=list, ...)
```

직접 `errors: list = []` 하면 **모든 인스턴스가 동일 리스트 공유** (mutable default args 함정). `default_factory=list` 가 매번 새 리스트 만듦.

### 1.3 숫자 검증 — `ge`/`le`

```python
batch_size: int = Field(1000, ge=100, le=10000)
```

요청에 `batch_size: 50` 이 오면 422 응답:
```json
{
  "detail": [{
    "loc": ["body", "batch_size"],
    "msg": "Input should be greater than or equal to 100",
    "type": "greater_than_equal"
  }]
}
```

→ 라우터 안에서 일일이 검증 안 해도 됨.

---

## 2. Optional 의 의미와 함정

`Optional[X]` 는 `Union[X, None]` 의 단축.

```python
end_index: Optional[int] = Field(None, ...)
```

JSON 으로 들어올 수 있는 값:
- `null` → `end_index = None`
- `100` → `end_index = 100`
- 키 자체 누락 → `end_index = None` (기본값)

**함정**:
- `Optional[int] = Field(...)` (필수 + 옵션) 이면 **요청에 키가 있어야 하지만 값은 null 가능** → 의도치 않은 의미가 됨.
- 키가 빠질 수 있으면 반드시 기본값 명시.

---

## 3. response_model 과 직렬화

```python
@router.post("/prepare-applications", response_model=PrepareApplicationsResponse)
async def prepare_applications(request: PrepareApplicationsRequest):
    ...
    return PrepareApplicationsResponse(
        status="completed",
        total_count=total_count,
        file_path=str(file_path),
        file_size_mb=file_size,
        execution_time=execution_time,
        updated_after=request.updated_after
    )
```

**`response_model` 의 효과**:
1. 응답 직렬화 시 모델에 정의된 필드만 노출 (의도치 않은 누출 방지).
2. OpenAPI 문서에 응답 스키마 자동 생성.
3. 응답 검증 (반환값이 모델과 안 맞으면 500).

**dict 반환도 가능**:
```python
return {"status": "completed", "total_count": ...}
```
→ FastAPI 가 모델로 자동 변환. 단 IDE 자동완성·정적 검사 약해짐.

---

## 4. 구버전 메서드 `request.dict()` vs 신버전 `model_dump()`

위 코드에 `request.dict()` 가 있는데:

```python
logger.info(f"Starting migration with request: {request.dict()}")
```

**Pydantic v1**: `model.dict()`
**Pydantic v2**: `model.model_dump()` (dict() 는 deprecated)

마이그레이션:
```python
# v1
data = req.dict()
data = req.dict(exclude={"password"})
json_str = req.json()

# v2
data = req.model_dump()
data = req.model_dump(exclude={"password"})
json_str = req.model_dump_json()
```

---

## 5. 모델 재사용 vs 분리

질문: 같은 도메인의 Request/Response 가 거의 같은 필드를 갖는다면?

**옵션 A — 분리 (위 예제처럼)**:
- 명확. 각 모델의 의미가 한눈에.
- 변경이 한쪽에만 적용되도록 격리.

**옵션 B — 상속**:
```python
class MigrationStatusBase(BaseModel):
    is_running: bool
    current_batch: int
    total_batches: int

class MigrationStatusResponse(MigrationStatusBase):
    progress_percentage: float
    elapsed_time: Optional[float]
```

**옵션 C — 모델 변형 (`model_validate`, `model_dump(include=...)`)**:
- 한 모델에서 view/dto 를 동적으로 파생.
- 복잡한 도메인엔 좋지만 작은 도메인엔 과함.

**경험칙**: API 모델은 도메인 모델과 분리. 도메인 모델은 비즈니스 로직 중심, API 모델은 외부 계약 중심.

---

## 6. validator 로 커스텀 검증

```python
from pydantic import BaseModel, field_validator

class PrepareApplicationsRequest(BaseModel):
    updated_after: Optional[str] = None

    @field_validator("updated_after")
    @classmethod
    def validate_date_format(cls, v):
        if v is None:
            return v
        try:
            datetime.strptime(v, "%Y-%m-%d")
        except ValueError:
            raise ValueError("must be YYYY-MM-DD format")
        return v
```

위 라우터 코드는 검증을 핸들러 안에서 직접 하고 있는데:
```python
if request.updated_after:
    try:
        datetime.strptime(request.updated_after, '%Y-%m-%d')
    except ValueError:
        raise HTTPException(400, "updated_after must be in YYYY-MM-DD format")
```

→ `field_validator` 로 옮기면 같은 검증을 다른 라우터에서도 자동 적용. 또한 422 응답으로 자동 변환되므로 일관성 ↑.

---

## 7. 모델 상속과 Config

Pydantic v2 의 model_config:

```python
class StrictModel(BaseModel):
    model_config = ConfigDict(
        extra="forbid",          # 정의 안 된 필드 들어오면 에러 (기본은 ignore)
        str_strip_whitespace=True,
        from_attributes=True,    # ORM 객체로부터 변환 허용 (구 orm_mode)
    )
```

**`extra="forbid"`** 가 보안에 도움 — 클라이언트가 권한 없는 필드(`is_admin: true`) 를 슬쩍 넣어도 무시되는 게 아니라 422 로 거부.

---

## 8. 응용 포인트

- 모든 라우터에서 Request/Response 를 `BaseModel` 로 정의 — dict 직접 다루기 금지.
- `Field` 의 `description` 을 충실히 → Swagger UI 가 거의 자동 문서.
- 숫자/문자열 제약은 `ge/le/min_length` 로 모델에 가둠. 라우터에 검증 흩지 않음.
- 보안 민감 API 는 `extra="forbid"` 로 잠금.
- v2 사용 시 `model_dump`, `field_validator` 로 마이그레이션.
