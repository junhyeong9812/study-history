# 12 — FastAPI Router + Pydantic (HTTP layer 의 책임)

> Router 는 *얇아야 한다 (thin controller)*. HTTP 의 RFC 적 관심사 (status code, header, JSON 직렬화, 예외 → HTTPException) 만 다루고, 비즈니스 로직은 핸들러에 위임. ShopTracker 의 `orders/presentation/router.py` 는 이 원칙의 모범적 예다 (몇 가지 누수 빼고).

---

## §0 문제 정의 — Fat Controller 의 함정

```python
# anti-pattern (Fat Controller)
@router.post("/orders")
async def create_order(body, db: Session):
    if not body.customer_name:
        raise HTTPException(400, ...)
    items = [OrderItem(...) for item in body.items]
    total = sum(item.unit_price * item.quantity for item in items)
    order = Order(id=uuid4(), customer_name=body.customer_name, items=items, total=total, ...)
    db.add(order)
    db.commit()
    await pg_api.charge(order.id, total)       # ← HTTP layer 가 결제까지
    await mail.send(body.customer_name, ...)   # ← HTTP layer 가 메일까지
    return order
```

문제 :
- 비즈니스 로직 (검증, 총액 계산, 결제, 메일) 이 router 안에.
- 테스트하려면 HTTP 호출까지 시뮬.
- 다른 진입점 (예 : CLI, gRPC) 이 생기면 로직 복제.

해결 : **Router = HTTP ↔ Application 의 단순 어댑터**.

---

## §1 본질 — Hexagonal 의 입력 어댑터 (Driving Adapter)

```
HTTP request
    │
    ▼
┌───────────────────────────────┐
│ Router (presentation)         │
│  - JSON → Pydantic (검증)     │
│  - Pydantic → Command DTO     │
│  - Handler 호출               │
│  - Domain → Response DTO      │
│  - Domain Exception → HTTP    │
└───────────────────────────────┘
    │
    ▼
Application (Handler)
    │
    ▼
Domain
```

Router 의 *유일한* 책임은 **HTTP 와 Application 사이의 변환**. 그 외는 모두 다른 레이어.

---

## §2 ShopTracker 코드 정독

### 2.1 Router 선언

```python
# orders/presentation/router.py
router = APIRouter(
    prefix="/api/v1/orders",
    tags=["Orders"],
    route_class=DishkaRoute,
)
```

- `prefix` : 모든 엔드포인트 앞 경로.
- `tags` : Swagger UI 그룹핑.
- `route_class=DishkaRoute` : Dishka 의 `FromDishka[T]` 가 동작하기 위한 설정 (03 장).

### 2.2 POST — 생성

```python
@router.post("/", status_code=201)
async def create_order(
    body: CreateOrderRequest,
    handler: FromDishka[CreateOrderHandler],
) -> OrderResponse:
    try:
        command = CreateOrderCommand(
            customer_name=body.customer_name,
            items=[
                OrderItemDTO(
                    product_name=item.product_name,
                    quantity=item.quantity,
                    unit_price=Decimal(str(item.unit_price)),
                )
                for item in body.items
            ],
        )
        order_id = await handler.handle(command)
        order = await handler._repo.find_by_id(order_id)
        return _order_to_response(order)
    except InvalidOrderError as e:
        raise HTTPException(status_code=400, detail=str(e))
```

읽어보기 :

- `body: CreateOrderRequest` — Pydantic 모델. FastAPI 가 JSON → Pydantic 자동 변환 + 검증 (필수 필드 누락이면 422).
- `handler: FromDishka[CreateOrderHandler]` — DI 가 주입.
- **3 단 변환** :
  1. `CreateOrderRequest` (Pydantic) → `CreateOrderCommand` (dataclass) — application 경계 안으로.
  2. `unit_price=Decimal(str(item.unit_price))` — float → Decimal. **여기가 Money 의 정밀도 보호 시작점** (Pydantic 모델은 float 으로 받을 수 있고 변환 시 str 경유 필수).
  3. `Order` (도메인) → `OrderResponse` (Pydantic) — 응답.
- **누수** : `await handler._repo.find_by_id(order_id)` — router 가 핸들러의 *private 필드* (`_repo`) 에 접근. 이건 Layer 위반 (응답을 위해 Query handler 를 따로 쓰는 게 정석).
- `except InvalidOrderError` — 도메인 예외 → 400.

### 2.3 GET (목록)

```python
@router.get("/")
async def list_orders(
    handler: FromDishka[ListOrdersHandler],
    customer_name: str | None = None,
    status: str | None = None,
    page: int = 1,
    size: int = 20,
) -> OrderListResponse:
    query = ListOrdersQuery(...)
    result = await handler.handle(query)
    return OrderListResponse(
        items=[_order_to_response(o) for o in result.items],
        total=result.total, page=result.page, size=result.size,
    )
```

- 쿼리 파라미터는 함수 파라미터로. FastAPI 가 `?customer_name=alice&page=2` 자동 파싱.
- `handler` 가 **첫 번째 파라미터가 아닌 위치** 에 있음 — 의도적 (FastAPI 는 어느 위치든 OK, 단 Pydantic body 는 보통 앞).
- 응답은 `OrderListResponse` — Pydantic.

### 2.4 GET (상세) + 예외 매핑

```python
@router.get("/{order_id}")
async def get_order(
    order_id: str,
    handler: FromDishka[GetOrderHandler],
) -> OrderResponse:
    try:
        order = await handler.handle(GetOrderQuery(order_id=order_id))
        return _order_to_response(order)
    except OrderNotFoundError:
        raise HTTPException(status_code=404, detail="주문을 찾을 수 없습니다")
```

- path param `{order_id}` → 함수 파라미터.
- 도메인 예외 → HTTP. **이게 router 의 진짜 책임** — domain 의 어휘를 HTTP 어휘로 번역.

### 2.5 응답 변환 헬퍼

```python
def _order_to_response(order: Order) -> OrderResponse:
    return OrderResponse(
        id=str(order.id),
        customer_name=order.customer_name,
        status=order.status.value,
        total_amount=float(order.total_amount.amount),
        currency=order.total_amount.currency,
        items=[
            OrderItemResponse(
                product_name=item.product_name,
                quantity=item.quantity,
                unit_price=float(item.unit_price.amount),
                subtotal=float(item.subtotal.amount),
            )
            for item in order.items
        ],
        created_at=order.created_at,
        updated_at=order.updated_at,
    )
```

- 도메인의 `Order` 를 *외부 표현* (`OrderResponse`) 으로 변환.
- 응답은 *flatten* — `Money.amount` / `Money.currency` 분리. 클라이언트가 다루기 편한 형태.
- 이 변환이 *방화벽* — 도메인 객체가 HTTP 까지 안 노출됨.
- 단, `subtotal=float(item.subtotal.amount)` — Decimal → float. 정밀도 손실 가능성. `str` 로 보내는 게 더 안전 (Pydantic 의 `Decimal` 직렬화도 옵션).

### 2.6 Pydantic 스키마 (`schemas.py`)

(코드 부분만 발췌됨, schemas.py 의 흔한 구조 :)

```python
class CreateOrderRequest(BaseModel):
    customer_name: str = Field(min_length=1, max_length=100)
    items: list["OrderItemRequest"] = Field(min_length=1)

class OrderItemRequest(BaseModel):
    product_name: str
    quantity: int = Field(gt=0)
    unit_price: float = Field(gt=0)

class OrderResponse(BaseModel):
    id: str
    customer_name: str
    status: str
    total_amount: float
    ...

    model_config = ConfigDict(from_attributes=True)   # dataclass 에서 변환 가능
```

- `Field(min_length=...)`, `gt=0` — **검증 규칙이 스키마에 박힘**. router 가 직접 검증할 필요 없음 (실패 시 422).
- `model_config = ConfigDict(from_attributes=True)` — `OrderResponse.model_validate(order)` 가 dataclass / ORM 에서 자동 변환 가능 (Pydantic v2).

---

## §3 직접 구현 — 미니 router

```python
from fastapi import FastAPI, APIRouter, HTTPException
from pydantic import BaseModel, Field

class CreateUserRequest(BaseModel):
    name: str = Field(min_length=1, max_length=50)
    email: EmailStr

class UserResponse(BaseModel):
    id: int
    name: str
    email: str

router = APIRouter(prefix="/users", tags=["Users"])

@router.post("/", status_code=201)
async def create_user(
    body: CreateUserRequest,
    handler: FromDishka[CreateUserHandler],
) -> UserResponse:
    try:
        user_id = await handler.handle(CreateUserCommand(name=body.name, email=body.email))
        return UserResponse(id=user_id, name=body.name, email=body.email)
    except DuplicateEmailError as e:
        raise HTTPException(409, str(e))
```

총 15 줄. *얇은 router* 의 모범.

---

## §4 함정

### 4.1 Router 가 핸들러 내부 접근 (`handler._repo`)

ShopTracker 의 `_order_to_response(await handler._repo.find_by_id(...))` :
- 캡슐화 깸.
- 정석은 별도 Query handler 호출 :
  ```python
  order_id = await create_handler.handle(command)
  order = await get_handler.handle(GetOrderQuery(str(order_id)))
  return _order_to_response(order)
  ```
- 또는 Command handler 가 *생성된 entity* 를 반환 (도메인 노출 트레이드오프).

### 4.2 비즈니스 검증을 router 에서

```python
# anti-pattern
if body.customer_name in BANNED_LIST:
    raise HTTPException(403, ...)
```

→ 도메인 규칙은 도메인에. router 는 *구조적 검증* (필드 존재, 형식) 만.

### 4.3 도메인 예외를 router 에서 매번 try

5 개 엔드포인트마다 같은 `except DomainException → HTTPException` 반복.

해결 : **Exception handler** 를 FastAPI 에 등록.

```python
@app.exception_handler(OrderNotFoundError)
async def order_not_found_handler(request, exc):
    return JSONResponse(status_code=404, content={"detail": str(exc)})

@app.exception_handler(InvalidOrderError)
async def invalid_order_handler(request, exc):
    return JSONResponse(status_code=400, content={"detail": str(exc)})
```

→ router 코드가 try 없이 깔끔.

### 4.4 응답 DTO 와 도메인 entity 의 의존 방향 역전

```python
# anti-pattern - 도메인이 Pydantic 알게
class Order(BaseModel):       # ← Pydantic 상속
    ...
```

도메인이 인프라 (Pydantic) 에 종속. ShopTracker 는 도메인 = dataclass + Pydantic = presentation 으로 분리.

### 4.5 status_code 누락

POST 에 `status_code=201` 안 적으면 200 으로 응답 — REST convention 위배. ShopTracker 는 명시.

### 4.6 응답에 도메인 비밀 노출

`OrderResponse` 가 도메인의 `internal_notes` 같은 필드까지 모두 직렬화하면 보안 사고. **응답 DTO 는 *명시적 화이트리스트***.

### 4.7 from_attributes 의 마법 의존

Pydantic 의 `model_validate(domain_obj)` 가 자동 변환 — 편하지만 *어떤 필드가 노출되는지* 가 안 보임. ShopTracker 는 명시적 매핑 (`_order_to_response`) — 더 verbose 지만 안전.

---

## §5 다른 환경

| 환경 | Router 패턴 |
|---|---|
| **Spring** | `@RestController` + `@RequestMapping`. `@Valid` 로 자동 검증. `ControllerAdvice` 로 예외 매핑. |
| **NestJS** | `@Controller` + `@Get/@Post`. `class-validator` 로 DTO 검증. `ExceptionFilter` 로 매핑. |
| **Express + zod** | 라우트 핸들러 안에서 `zod.parse(req.body)` — 명시적. |
| **Django Rest Framework** | `Serializer` + `ViewSet` — Django ORM 결합 강함. |

FastAPI + Pydantic 은 Spring 의 자동 검증 + ControllerAdvice 와 NestJS 의 데코레이터 스타일을 혼합한 느낌.

---

## §6 테스트 전략

### 6.1 TestClient

```python
from fastapi.testclient import TestClient

def test_create_order_returns_201(client: TestClient):
    res = client.post("/api/v1/orders/", json={
        "customer_name": "alice",
        "items": [{"product_name": "a", "quantity": 1, "unit_price": 100}],
    })
    assert res.status_code == 201
    assert res.json()["customer_name"] == "alice"

def test_invalid_payload_returns_422(client):
    res = client.post("/api/v1/orders/", json={"customer_name": ""})
    assert res.status_code == 422   # Pydantic 검증 실패
```

### 6.2 Domain exception → HTTP 매핑

```python
def test_get_unknown_order_returns_404(client):
    res = client.get("/api/v1/orders/00000000-0000-0000-0000-000000000000")
    assert res.status_code == 404
    assert "찾을 수 없" in res.json()["detail"]
```

### 6.3 Handler mocking via DI override

```python
def test_create_order_calls_handler(client, container):
    fake = FakeCreateOrderHandler()
    container.override(CreateOrderHandler, fake)
    client.post("/api/v1/orders/", json={...})
    assert fake.calls == 1
```

---

## §10 학습 포인트 (한 줄 요약)

1. **Thin Controller** : router 에 비즈니스 X.
2. **Pydantic = HTTP 어휘**, **Command/Domain = Application 어휘** — 경계에서 변환.
3. **FromDishka[T]** : DI 주입을 라우트 시그니처에서.
4. **검증은 Pydantic Field** — router 코드에 if 검증 X.
5. **도메인 예외 → HTTPException** 매핑이 router 의 진짜 책임.
6. **반복 try/except 는 Exception handler** 로 중앙화.
7. **응답 DTO 는 명시적 화이트리스트** — 도메인 비밀 노출 차단.
8. **status_code 명시** — REST convention.
9. **`Decimal(str(float))`** 변환은 router 에서 *시작*.
10. **TestClient** 로 통합 검증, **DI override** 로 단위 검증.

---

## 추가 참고

- FastAPI docs : https://fastapi.tiangolo.com/
- Pydantic v2 docs : https://docs.pydantic.dev/
- ShopTracker 다음 글 : `13-testing-strategies.md`
