# FastAPI 기본 구조 — 라우터 분리, CORS, 헬스체크

> 다국가(KR/JP/CH/US/EU) 마이그레이션 API 5종을 동일한 FastAPI 골격으로 운영한 경험을 정리한 학습 노트.
> "한 번 잘 만든 골격을 5번 복제"가 운영 비용을 어떻게 줄였는지 함께 본다.

---

## 0. 분석 대상 코드

다국가 마이그레이션 API 중 KR 인스턴스의 `main.py` (5국가 모두 동일 구조).

```python
# kr-search-app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import uvicorn
import multiprocessing
import sys
from contextlib import asynccontextmanager

from src.core.config import settings
from src.core.logger import get_logger
from src.routers import migration_router
from src.routers import customer_migration_router
from src.routers import orchestration_router

logger = get_logger("app")


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    logger.info("=" * 50)
    logger.info(f"{settings.API_TITLE} v{settings.API_VERSION} Starting...")
    logger.info(f"Environment: {settings.ES_HOST}:{settings.ES_PORT}")
    logger.info(f"ES Index: {settings.ES_UNIFIED_INDEX_NAME}")
    logger.info(f"Python version: {sys.version}")
    logger.info(f"Multiprocessing start method: {multiprocessing.get_start_method()}")
    logger.info("=" * 50)

    yield

    # Shutdown
    logger.info("Application shutting down...")


app = FastAPI(
    title=settings.API_TITLE,
    version=settings.API_VERSION,
    description="Trademark Data Migration API",
    docs_url="/docs",
    redoc_url="/redoc",
    lifespan=lifespan
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(
    migration_router.router,
    prefix="/api/v1/migration",
    tags=["Migration"]
)
app.include_router(
    customer_migration_router.router,
    prefix="/api/v1/customer",
    tags=["Customer Migration"]
)
app.include_router(
    orchestration_router.router,
    tags=["Orchestration"]
)


@app.get("/")
async def root():
    return {
        "message": "Trademark Migration API",
        "version": settings.API_VERSION,
        "docs": "/docs"
    }


@app.get("/health")
async def health_check():
    import psutil
    return {
        "status": "healthy",
        "api_version": settings.API_VERSION,
        "system": {
            "cpu_percent": psutil.cpu_percent(),
            "memory_percent": psutil.virtual_memory().percent,
            "process_count": len(psutil.pids())
        }
    }


if __name__ == "__main__":
    try:
        multiprocessing.set_start_method('spawn', force=True)
    except RuntimeError:
        pass

    uvicorn.run(
        "main:app",
        host=settings.API_HOST,
        port=settings.API_PORT,
        reload=False,
        log_level="info",
        workers=1,
        loop="asyncio"
    )
```

---

## 1. 핵심 구성 요소

| # | 요소 | 역할 |
|---|------|------|
| 1 | `FastAPI(...)` | 앱 인스턴스 — 메타정보 + lifespan 등록 |
| 2 | `lifespan` (asynccontextmanager) | 시작/종료 훅. yield 전이 startup, 후가 shutdown |
| 3 | `CORSMiddleware` | 브라우저에서 다른 오리진에서 호출할 수 있도록 |
| 4 | `APIRouter` (`include_router`) | 도메인별 라우터 분리 — prefix + tags 로 OpenAPI 그룹화 |
| 5 | `/health` | 외부 모니터링/로드밸런서용. **psutil로 시스템 리소스까지 노출** |
| 6 | `uvicorn.run` | ASGI 서버. workers=1 + ProcessPoolExecutor 분리 운영 |

---

## 2. `FastAPI(...)` 생성자 옵션

```python
app = FastAPI(
    title=settings.API_TITLE,         # OpenAPI 문서 제목
    version=settings.API_VERSION,     # 버전
    description="...",                # 설명
    docs_url="/docs",                 # Swagger UI 경로 (None 하면 비활성)
    redoc_url="/redoc",               # ReDoc 경로
    lifespan=lifespan                 # 시작/종료 훅 (구 on_event 대체)
)
```

**핵심 포인트**:
- `docs_url=None` 으로 운영 환경에서 Swagger UI를 가릴 수 있다.
- `lifespan` 은 0.93+ 권장 방식. `@app.on_event("startup")` 은 deprecated.
- title/version 은 OpenAPI 스펙(`/openapi.json`) 메타에 그대로 노출됨 → 외부 노출 시 민감도 고려.

---

## 3. 라우터 분리 — `include_router`

```python
from src.routers import migration_router, customer_migration_router, orchestration_router

app.include_router(migration_router.router, prefix="/api/v1/migration", tags=["Migration"])
app.include_router(customer_migration_router.router, prefix="/api/v1/customer", tags=["Customer Migration"])
app.include_router(orchestration_router.router, tags=["Orchestration"])
```

라우터 정의 측:

```python
# src/routers/migration_router.py
from fastapi import APIRouter
router = APIRouter()

@router.post("/start", response_model=Dict[str, Any])
async def start_migration(request: MigrationStartRequest, background_tasks: BackgroundTasks):
    ...
```

**왜 분리하나**:
- 도메인 경계: migration / customer / orchestration 이 서로 의존 없이 진화.
- OpenAPI 그룹화: `tags=["Migration"]` 가 Swagger UI에서 섹션으로 묶임.
- 테스트 격리: 라우터 단위로 `TestClient(app)` 또는 단위 라우터만 마운트해서 테스트 가능.

**전체 vs 부분 prefix**:
- prefix를 라우터 등록 시점에 주면, 라우터 코드 안에서는 짧은 경로(`/start`)만 다룬다 → 라우터를 다른 prefix로 재사용하기 쉬움.
- `orchestration_router` 처럼 prefix 없이 등록하면 라우터 안에 절대 경로(`/orch/...`)가 들어 있는 형태.

---

## 4. CORS 미들웨어

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

| 옵션 | 의미 | 운영 권장 |
|------|------|-----------|
| `allow_origins` | 허용 출처 | 운영에서는 `["https://app.example.com"]` 처럼 화이트리스트 |
| `allow_credentials` | 쿠키/Authorization 허용 | True 면 origins 에 `*` 사용 시 브라우저가 거부 (스펙) |
| `allow_methods` | 허용 HTTP 메서드 | `["GET","POST"]` 처럼 좁히는 게 안전 |
| `allow_headers` | 허용 요청 헤더 | 동일 |

**알려진 함정**: `allow_origins=["*"]` + `allow_credentials=True` 는 브라우저가 무시한다. 둘 중 하나만 써야 함. 위 코드는 내부망 한정으로 무시하고 있는 케이스.

---

## 5. `/health` 엔드포인트 — 모니터링과의 계약

```python
@app.get("/health")
async def health_check():
    import psutil
    return {
        "status": "healthy",
        "api_version": settings.API_VERSION,
        "system": {
            "cpu_percent": psutil.cpu_percent(),
            "memory_percent": psutil.virtual_memory().percent,
            "process_count": len(psutil.pids())
        }
    }
```

**핵심**:
- 단순 `{"status":"healthy"}` 가 아니라 **시스템 리소스를 함께 반환** → 외부 에이전트가 메트릭 수집 시 1번의 호출로 처리.
- `psutil.cpu_percent()` 첫 호출은 `0.0` 을 반환 (직전 측정 기준점이 없어서). 정확값이 필요하면 `psutil.cpu_percent(interval=0.1)`.
- 단, interval 을 주면 응답이 그 시간만큼 지연됨 → 빈번한 호출은 별도 스케줄러에서 수집해서 캐시.

**Liveness vs Readiness**:
- 위 `/health` 는 liveness. "프로세스 살아있나?"
- readiness 는 "트래픽 받을 준비 됐나?" → DB/ES 연결 체크 포함이 필요. 이 프로젝트는 분리하지 않고 단순화.

---

## 6. `uvicorn.run` 옵션과 `workers=1` 결정

```python
uvicorn.run(
    "main:app",
    host=settings.API_HOST,
    port=settings.API_PORT,
    reload=False,
    log_level="info",
    workers=1,
    loop="asyncio"
)
```

**왜 `workers=1`?**
- 이 앱은 내부에서 `ProcessPoolExecutor` 로 마이그레이션 워커를 직접 띄운다.
- uvicorn의 `workers>1` 은 **fork된 워커 프로세스마다 별도 ProcessPoolExecutor** 를 만들게 됨 → 자식 프로세스 폭증 + GIL 회피 의미 없음 + 자원 충돌.
- 결론: **CPU 병렬성을 ProcessPoolExecutor 가 책임지면 uvicorn workers는 1**.

**`loop="asyncio"` vs `uvloop`**:
- `uvloop` 가 빠르지만 일부 라이브러리 호환성 이슈 있음 (특히 멀티프로세싱과 조합 시).
- 명시적으로 `asyncio` 지정 → 안전 우선.

**`reload=False`**:
- 운영. 개발에서는 `reload=True` + `reload_dirs=["src"]` 로 핫리로드.

**`multiprocessing.set_start_method('spawn', force=True)`**:
- Linux 기본값은 `fork` 인데, fork 는 부모의 모든 메모리/락/파일디스크립터를 공유 → asyncio loop 와 결합 시 deadlock 빈번.
- `spawn` 은 새 인터프리터 시작 → 안전. 단 워커가 모듈 import 를 다시 하므로 시작이 살짝 느림.
- macOS/Windows 는 이미 spawn 이 기본.

---

## 7. 5개국 동일 골격 운영 — DRY 의 한계와 트레이드오프

5개 국가(KR/JP/CH/US/EU) 마이그레이션 API 가 **이 main.py를 거의 그대로 복제**하고 있다. 차이는:

- `settings.API_TITLE` 의 국가명
- `settings.ES_HOST/PORT` (각 국가별 ES)
- 일부 국가별 데이터 변환 로직 (트랜스포머 모듈)

**왜 라이브러리화하지 않았나?**
- 국가별 스키마 차이가 점점 벌어짐 (KR 만 customer_indexer 가 있음, US/EU 는 등록정보 구조가 다름).
- 공통 라이브러리화 시 "공통 부분의 한 변경이 5개국 동시 영향" → 부담.
- 복제본 5개 + 각자 진화 → 변경 격리 (한 국가 장애가 다른 국가 빌드/배포에 영향 없음).

**그 대가**:
- 진짜 공통인 변경(예: 헬스체크 추가)은 **5번 복붙해야 함**.
- 5개국 컨테이너가 서로 다른 라이브러리 버전을 쓸 수도 있음.

**교훈**: "DRY 가 항상 옳지는 않다. 격리 가치가 코드 중복 비용보다 클 수 있다."

---

## 8. 응용 포인트

- 새 API 프로젝트 시작 시 위 골격 그대로 사용 가능.
- 운영 환경에서는 `docs_url=None`, CORS origins 화이트리스트, `/health` 분리(liveness/readiness) 추가.
- ProcessPool/스레드풀로 CPU 병렬성을 직접 관리한다면 `workers=1` 유지. uvicorn 워커에 맡기면 `workers=N` (N=CPU 코어수)로.
