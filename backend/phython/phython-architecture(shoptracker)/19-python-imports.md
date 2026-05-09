# 19 — Python Imports / Packages / 모듈 시스템

> ShopTracker 의 `from app.orders.domain.entities import Order` 같은 import 한 줄에 *모듈 경로 / 패키지 / 순환 참조 / sys.path / __init__.py* 같은 숨은 메커니즘이 다 들어 있다. 이 글은 Python 모듈 시스템의 작동 원리와 모듈러 모놀리스에서의 import 규율을 정리.

---

## §0 모듈 vs 패키지

| | 모듈 (module) | 패키지 (package) |
|---|---|---|
| 형태 | `.py` 파일 | 디렉토리 |
| 식별 | 파일 이름 | 디렉토리 이름 + `__init__.py` (또는 namespace package) |
| 예 | `app/orders/domain/entities.py` → `app.orders.domain.entities` | `app/orders/` → `app.orders` |
| import | `from a.b import c` | 동일 |

`__init__.py` 가 있으면 *regular package*, 없으면 *namespace package* (3.3+).

---

## §1 import 의 작동

```python
import app.orders.domain.entities
```

1. `sys.modules` 캐시에 있으면 그것 반환.
2. 없으면 `sys.path` 의 디렉토리들을 순회.
3. `app/orders/domain/entities.py` 발견 → 실행 → `sys.modules["app.orders.domain.entities"]` 에 등록.
4. `app`, `app.orders`, `app.orders.domain` 도 같은 방식으로 로드.

`from X import Y` 는 :
- X 를 위 흐름으로 로드.
- `Y = X.Y` 또는 `import X.Y` 후 Y 추출.

---

## §2 sys.path 와 PYTHONPATH

```python
import sys
print(sys.path)
# ['', '/usr/lib/python3.12', '/usr/lib/python3.12/site-packages', ...]
```

- `''` = 현재 작업 디렉토리.
- 환경변수 `PYTHONPATH` 가 추가됨.
- `pip install` 한 패키지는 `site-packages` 에.
- `pip install -e .` (editable) 은 *프로젝트의 src* 가 sys.path 에 추가됨.

---

## §3 src layout vs flat layout

### 3.1 flat layout

```
project/
├── app/
│   ├── __init__.py
│   └── orders/
└── tests/
```

`from app.orders import ...` 가 cwd 에서 작동.

### 3.2 src layout (ShopTracker)

```
project/
├── src/
│   └── app/
│       ├── __init__.py
│       └── orders/
└── tests/
```

`from app.orders import ...` 가 작동하려면 `src/` 가 sys.path 에 있어야 함. `pyproject.toml` 에서 :

```toml
[tool.setuptools.packages.find]
where = ["src"]
```

`pip install -e .` 시 src 자동 등록.

장점 :
- 테스트가 *설치된 패키지* 를 import — 실 사용 환경과 동일.
- import 순환 / 누락이 더 잘 잡힘.

ShopTracker 가 src layout 채택.

---

## §4 absolute vs relative import

```python
# absolute (권장)
from app.orders.domain.entities import Order

# relative
from .domain.entities import Order        # 현재 패키지
from ..shared.value_objects import Money  # 부모 패키지
```

PEP 8 : **absolute 권장**. 단 같은 패키지 내부에서는 relative 도 OK (refactor 시 안전).

ShopTracker : 모두 absolute. 모듈 경로가 명시적 → 어느 모듈에 속하는지 한눈에.

---

## §5 모듈러 모놀리스에서의 import 규율

### 5.1 권장 규칙

```
app/
├── shared/         ← 모두 import 가능
├── orders/
│   ├── domain/     ← 다른 모듈 import 금지
│   ├── application/← 같은 모듈 + shared 만
│   ├── infrastructure/ ← 같은 모듈만
│   └── presentation/ ← 같은 모듈만
├── payments/       ← orders 를 import 금지!
├── shipping/       ← payments 를 import 금지!
└── tracking/
```

원칙 :
- **모듈 간 상호 import 금지** — 결합 방지.
- **shared 만 모두가 import** — 공통 계약.
- **레이어 방향** : presentation → application → domain. 역방향 X.

### 5.2 어떻게 강제?

- 코드 리뷰.
- 정적 도구 :
  - `import-linter` (Python) — 규칙을 INI 로 정의 → CI 에서 검증.

```ini
[importlinter]
root_package = app

[importlinter:contract:modules]
name = Modules don't depend on each other
type = independence
modules =
    app.orders
    app.payments
    app.shipping
    app.tracking
```

ShopTracker 는 명시적으로 import-linter 를 안 쓰지만 코드 리뷰로 강제.

---

## §6 순환 참조

```python
# app/a.py
from app.b import B
class A: ...

# app/b.py
from app.a import A    # ← 순환!
class B: ...
```

→ `ImportError: cannot import name 'A'`.

해결 :
- **함수 안에서 import** (lazy) — 호출 시점에 import.
  ```python
  # app/b.py
  class B:
      def use_a(self):
          from app.a import A
          return A()
  ```
- **TYPE_CHECKING** — 타입 힌트용 import 만 :
  ```python
  from typing import TYPE_CHECKING
  if TYPE_CHECKING:
      from app.a import A
  ```
- **구조 재설계** — 정말 필요하면 모듈을 잘못 나눈 신호.

ShopTracker : 모듈 간 import 금지 원칙 → 순환 거의 발생 안 함.

---

## §7 `__init__.py` 의 역할

```python
# app/orders/__init__.py
from .domain.entities import Order
from .domain.value_objects import OrderStatus

__all__ = ["Order", "OrderStatus"]
```

- 패키지 import 시 실행됨.
- *공개 API* 정의 — `from app.orders import Order` 가 바로 됨.
- 비어 있어도 OK (그냥 패키지 marker).

ShopTracker 는 `__init__.py` 가 거의 빈 상태 — 명시적 import 경로 선호.

---

## §8 import 순서 (PEP 8)

```python
# 1. 표준 라이브러리
from datetime import datetime
from uuid import UUID

# 2. third-party
import structlog
from fastapi import APIRouter
from sqlalchemy import select

# 3. local (자기 프로젝트)
from app.orders.domain.entities import Order
from app.shared.events import OrderCreatedEvent
```

각 그룹 사이 빈 줄. `isort` / `ruff` 가 자동 정렬.

---

## §9 함정

### 9.1 sys.path 조작

```python
import sys
sys.path.insert(0, "/some/path")
import foo
```

→ 더럽고 fragile. *editable install* 이나 PYTHONPATH 사용.

### 9.2 모듈 vs 패키지 혼동

`from app.orders import entities` — `entities` 가 모듈인지 객체인지 헷갈림. 풀로 적기 : `from app.orders.domain.entities import Order`.

### 9.3 글로벌 import 부작용

```python
# heavy_module.py
import slow_thing       # 5 초 걸림
START_TIME = compute()  # 매 import 마다
```

→ 모듈 최상위에서 무거운 작업 X. lazy 또는 명시적 init 함수.

### 9.4 동일 모듈을 여러 경로로 import

```python
# 같은 파일을 두 경로로
from app.foo import X       # sys.modules['app.foo']
from foo import X           # sys.modules['foo']  ← 다른 모듈로 인식!
```

`isinstance` 체크가 깨짐. 항상 한 경로로.

### 9.5 `__init__.py` 의 무거운 작업

```python
# bad
# app/__init__.py
from app.orders.domain.entities import Order   # 무거우면 import 시 다 로드
```

→ 큰 패키지의 `__init__.py` 는 비워 두거나 *명시적 export* 만.

### 9.6 namespace package 의 함정

`__init__.py` 없는 폴더는 namespace package. 의도치 않게 다른 곳의 같은 이름과 합쳐질 수 있음. 명시적 `__init__.py` 권장.

---

## §10 ShopTracker 의 import 패턴

### 10.1 도메인은 표준 라이브러리만

```python
# orders/domain/entities.py
from dataclasses import dataclass
from datetime import datetime, UTC
from uuid import UUID, uuid4

from app.shared.value_objects import Money
from app.orders.domain.value_objects import OrderStatus
from app.orders.domain.exceptions import InvalidOrderError, InvalidStatusTransition
```

- third-party 0 (FastAPI, SQLAlchemy 등 없음).
- 같은 모듈 내부 + shared 만.
- 다른 모듈 (payments) import 0.

### 10.2 application 은 도메인 + shared

```python
# orders/application/command_handlers.py
from app.orders.domain.entities import Order, OrderItem
from app.orders.domain.interfaces import OrderRepositoryProtocol
from app.shared.event_bus import EventBus
from app.shared.events import OrderCreatedEvent
```

- 자기 도메인 + shared 의 계약.
- infrastructure 미참조 (Protocol 만 의존).

### 10.3 infrastructure 는 외부 라이브러리 OK

```python
# orders/infrastructure/repository.py
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession
from app.orders.domain.entities import Order
from app.orders.infrastructure.models import OrderModel
```

- SQLAlchemy 등 third-party.
- 도메인 import 는 entity 만.

### 10.4 presentation 은 FastAPI + 자기 application

```python
# orders/presentation/router.py
from fastapi import APIRouter, HTTPException
from dishka.integrations.fastapi import DishkaRoute, FromDishka
from app.orders.application.command_handlers import CreateOrderHandler, ...
```

---

## §11 학습 포인트 (한 줄 요약)

1. **모듈** = 파일, **패키지** = 디렉토리 (+ `__init__.py`).
2. **sys.modules 캐시** — 같은 모듈은 한 번만 로드.
3. **src layout** — 테스트가 설치된 패키지를 본다.
4. **absolute import 권장** — 명시적, 안전.
5. **모듈러 모놀리스의 import 규율** — 모듈 간 import 금지, shared 만 공유.
6. **순환 참조 = 설계 신호** — lazy import 또는 재설계.
7. **`__init__.py` 는 비우거나 명시적 export 만**.
8. **PEP 8 import 순서** — stdlib / third-party / local.
9. **import-linter** — CI 에서 규칙 강제.
10. **TYPE_CHECKING** — 타입 힌트만 위한 import 분리.

---

## 추가 참고

- Python docs : https://docs.python.org/3/reference/import.html
- import-linter : https://github.com/seddonym/import-linter
- PEP 328 (relative imports), PEP 420 (namespace packages)

---

이 글로 ShopTracker 의 study/ 19 개 doc 시리즈가 완성된다. 처음으로 돌아가 [00-index.md](00-index.md) 에서 흐름을 다시 보거나, 본격 코드 작성으로 넘어가자.
