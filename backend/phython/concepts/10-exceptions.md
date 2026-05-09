# 10 — 예외 모델 (try / except / else / finally / EAFP)

> Python 의 예외는 *제어 흐름의 일부*. `for` 가 끝나는 것도 StopIteration, dict 키 없는 것도 KeyError. "예외는 비싸다" 의 통념과 달리 *적극적 활용* 이 권장된다 — EAFP.

---

## §0 EAFP vs LBYL

| | LBYL (Look Before You Leap) | EAFP (Easier to Ask Forgiveness than Permission) |
|---|---|---|
| 패턴 | 미리 검사 | 일단 시도 + 예외 처리 |
| 예 | `if k in d: d[k]` | `try: d[k] except KeyError: ...` |
| 동시성 안전 | race 가능 | atomic |
| Python 관용 | △ | ★ |

```python
# LBYL - race condition 가능
if os.path.exists(path):
    f = open(path)        # 그 사이 누가 지웠으면 FileNotFoundError

# EAFP
try:
    f = open(path)
except FileNotFoundError:
    ...
```

→ Python 은 EAFP 선호. 예외가 *흐름 제어 도구*.

---

## §1 try / except / else / finally

```python
try:
    risky()
except ValueError as e:
    print(f"value: {e}")
except (KeyError, IndexError):
    print("key/index")
except Exception as e:
    print(f"other: {e}")
else:
    print("no exception")     # try 블록이 정상 종료됐을 때만
finally:
    print("always")           # 예외든 정상이든
```

- `else` : 자주 잊지만 유용 — *예외 없을 때만* 실행. except 가 잡지 않은 다른 예외가 발생하면 else 안 돌고 그 예외 전파.
- `finally` : cleanup. 예외 / return 이든 항상 실행.

---

## §2 예외 계층

```
BaseException
 ├── SystemExit
 ├── KeyboardInterrupt
 ├── GeneratorExit
 └── Exception
      ├── ArithmeticError ── ZeroDivisionError, OverflowError, ...
      ├── LookupError ── KeyError, IndexError
      ├── ValueError
      ├── TypeError
      ├── OSError ── FileNotFoundError, PermissionError, ...
      ├── RuntimeError ── RecursionError, NotImplementedError
      ├── StopIteration / StopAsyncIteration
      ├── ImportError ── ModuleNotFoundError
      └── ... (수십 가지)
```

규칙 :
- **`except Exception`** 으로 잡으면 SystemExit / KeyboardInterrupt 는 *통과*. 정상.
- **`except BaseException`** 또는 *bare except* 는 모든 것 잡음 — Ctrl+C 도 안 먹음. 거의 안 좋음.

---

## §3 raise

### 3.1 새 예외

```python
raise ValueError("bad input")
raise ValueError(f"bad: {x}")
```

### 3.2 re-raise

```python
try:
    risky()
except Exception:
    log("...")
    raise           # 원래 예외 그대로 다시
```

### 3.3 chain

```python
try:
    f()
except DatabaseError as e:
    raise BusinessError("...") from e   # __cause__ 설정
```

traceback 에 "The above exception was the direct cause of..." 표시.

`from None` 은 chain 숨김.

---

## §4 사용자 정의 예외

```python
class DomainError(Exception): pass

class OrderNotFoundError(DomainError):
    def __init__(self, order_id):
        super().__init__(f"order not found: {order_id}")
        self.order_id = order_id
```

ShopTracker 의 패턴 :

```python
# orders/domain/exceptions.py
class InvalidOrderError(Exception): ...
class InvalidStatusTransition(Exception): ...
class OrderNotFoundError(Exception): ...
```

도메인 예외를 별도 모듈에. router 에서 → HTTPException 매핑 (12 장).

### 4.1 어디까지 만들 것인가

- *비즈니스 의미가 있는* 모든 실패 = 별도 클래스.
- 단순 internal 오류 = `RuntimeError`.
- 호출자가 *구분해서 잡고 싶은* 경우 = 별도 클래스.

---

## §5 Exception group (3.11+, PEP 654)

```python
try:
    raise ExceptionGroup("multiple", [
        ValueError("v"),
        TypeError("t"),
    ])
except* ValueError as eg:
    print("got value:", eg.exceptions)
except* TypeError as eg:
    print("got type:", eg.exceptions)
```

`asyncio.TaskGroup` 이 여러 task 의 예외를 묶어 ExceptionGroup 으로 던짐. concurrent 에러 처리에 핵심.

---

## §6 자주 쓰는 패턴

### 6.1 retry

```python
for attempt in range(3):
    try:
        return call()
    except RetryableError:
        if attempt == 2: raise
        time.sleep(2 ** attempt)
```

### 6.2 fallback

```python
try:
    return primary()
except PrimaryError:
    return fallback()
```

### 6.3 logging + re-raise

```python
try:
    work()
except Exception:
    logger.exception("failed")    # traceback 자동 포함
    raise
```

### 6.4 context manager 가 예외 잡기

```python
from contextlib import suppress
with suppress(FileNotFoundError):
    os.remove(path)
```

### 6.5 사후 cleanup 보장

```python
try:
    work()
finally:
    cleanup()
```

또는 `with` (18 장).

---

## §7 ShopTracker 의 예외 패턴

### 7.1 도메인 예외 발생

```python
def _transition_to(self, target):
    if not self.status.can_transition_to(target):
        raise InvalidStatusTransition(self.status.value, target.value)
```

### 7.2 핸들러는 통과 (보통 안 잡음)

```python
async def handle(self, command):
    order = await self._repo.find_by_id(...)
    if order is None:
        raise OrderNotFoundError(...)
    order.cancel()                     # 예외 가능 — 통과
    await self._repo.update(order)
```

### 7.3 router 에서 HTTP 매핑

```python
try:
    await handler.handle(...)
except OrderNotFoundError:
    raise HTTPException(404, ...)
except InvalidStatusTransition as e:
    raise HTTPException(400, str(e))
```

또는 **FastAPI exception handler 로 중앙화** (12 장 §4.3).

### 7.4 EventBus 의 핸들러 격리

```python
for handler in handlers:
    try:
        await handler(event)
    except Exception as e:
        logger.error("handler failed", ...)
```

핸들러 하나 실패가 다른 핸들러를 막지 않게. 04 장.

---

## §8 함정

### 8.1 bare except

```python
try: f()
except:                # ← BaseException 까지 잡음. Ctrl+C 안 먹음.
    pass
```

→ 항상 `except Exception` 또는 구체적 예외.

### 8.2 except 너무 광범위

```python
try: f()
except Exception as e:
    log(e)             # 모든 버그도 삼킴
```

→ 필요한 예외만. 디버깅 어려워짐.

### 8.3 finally 의 return / break

```python
def f():
    try: return 1
    finally: return 2
f()                    # 2 - finally 가 이김!
```

→ finally 에서 return / break 안 좋음.

### 8.4 예외가 너무 비싸다는 미신

Python 의 예외는 *발생하지 않으면* 거의 무료 (try/except 자체는 빠름). *발생할 때* 만 비용. 따라서 "정상 흐름이 try 통과" 인 EAFP 는 OK.

### 8.5 async 의 예외

```python
async def f():
    raise ValueError()

asyncio.run(f())       # ValueError + traceback

# 잘못
task = asyncio.create_task(f())   # 결과 안 await 하면 예외 silent
```

→ task 의 예외는 await 또는 done callback 에서 확인.

### 8.6 ExceptionGroup 잡는 방법

```python
try: ...
except ExceptionGroup as eg:    # 모든 그룹 잡음
    ...

try: ...
except* ValueError:              # 그룹 안의 ValueError 만 (3.11+)
    ...
```

`except*` 는 새 문법 — 그룹 풀어서 매칭.

---

## §9 다른 언어 비교

| 언어 | 예외 |
|---|---|
| **Python** | EAFP, 흐름의 일부 |
| **Java** | checked exceptions (declare or catch) |
| **C#** | unchecked, similar to Python |
| **Go** | error 값 반환, no exception (panic 만 special) |
| **Rust** | Result<T, E>, no exception (panic = abort) |
| **JS** | try/catch, EAFP 가능 |

Java 의 checked 가 가장 strict, Go/Rust 는 *예외를 안 쓰는* 모델.

---

## §10 학습 포인트 (한 줄 요약)

1. **EAFP 우선** — 시도하고 예외 처리.
2. **try / except / else / finally** — else 도 활용.
3. **`except Exception`** 권장, **bare except 금지**.
4. **`raise ... from e`** — 예외 chain.
5. **사용자 정의 예외** = 비즈니스 의미 있는 실패.
6. **ExceptionGroup (3.11+)** = 여러 동시 예외.
7. **logging + re-raise** = 흔한 패턴.
8. **finally 에서 return X** — 흐름 꼬임.
9. **예외는 정상 흐름에서 거의 무료** — 미신 X.
10. **router 에서 도메인 예외 → HTTP 매핑** — 중앙화.

---

## 참고

- Python docs — Built-in exceptions, Exception handling
- PEP 654 (Exception Groups)
- "Effective Python" Brett Slatkin — 예외 챕터
