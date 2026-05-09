# 09 — 값 객체 (Value Object) — `Money` 의 설계

> `Decimal` 그대로 다니지 않고 `Money(amount, currency)` 로 한 번 감싸면 *통화 단위 mismatch*, *음수 금액*, *float 부동소수점 오차* 같은 클래스의 버그가 컴파일 / 런타임 진입 직전에 막힌다. ShopTracker 의 `Money` 는 70 줄짜리 작은 클래스지만 값 객체의 모든 핵심 (불변성, 동등성, 폐쇄 연산, 검증) 을 담고 있다.

---

## §0 문제 정의 — 원시값 (Primitive) 의 함정

```python
# anti-pattern
def calculate_total(unit_price: Decimal, quantity: int, discount: Decimal) -> Decimal:
    return unit_price * quantity - discount
```

이 한 줄의 잠재 버그 :

1. **단위 혼란** : `unit_price` 가 KRW 인데 `discount` 가 USD 면? 그냥 빼짐. 결과는 의미 없음.
2. **음수 통과** : `discount` 가 우연히 1000000 이고 unit_price\*quantity 가 100 이면 결과가 -999900. 음수 금액이 시스템 어딘가로 흘러감.
3. **float 의 함정** : `0.1 + 0.2 != 0.3`. Decimal 이라면 OK 지만 `float` 으로 받으면 회계 시스템 망가짐.
4. **의도 손실** : 함수 시그니처 `Decimal` 만 봐서는 "이게 돈이야 쿠폰 수야 별점 평균이야" 알 수 없음.
5. **검증 흩뿌림** : 모든 호출자가 음수 체크, 통화 체크, 반올림 체크를 *각자* 해야.

해결 : *돈* 이라는 도메인 개념을 **타입** 으로 만들기.

---

## §1 본질 메커니즘 — Value Object (DDD)

### 1.1 정의

> *식별자가 없고, 값 자체로 동등성을 판단하며, 불변인 객체.*

| | Entity | Value Object |
|---|---|---|
| 식별자 | 있음 (id) | 없음 |
| 동등성 | id 비교 | 모든 필드 비교 |
| 가변성 | 가변 (상태 변화) | 불변 (frozen) |
| 예 | Order, User | Money, Address, DateRange, Email |

`Money(100, "KRW") == Money(100, "KRW")` → True (필드 모두 같으면 같음).
`Money(100, "KRW") + Money(50, "KRW") = Money(150, "KRW")` (새 객체).

### 1.2 4 가지 핵심 속성

1. **불변 (Immutability)** — 생성 후 변경 X. 모든 연산이 새 인스턴스 반환. 멀티스레드 안전, 부수효과 없음.
2. **값 동등성 (Value Equality)** — 필드 값으로 비교 (`__eq__`).
3. **자기 검증 (Self-Validation)** — 생성자에서 invariant 검증.
4. **연산 폐쇄 (Closed Operations)** — Money + Money = Money. 도메인 안에서 닫힘.

---

## §2 ShopTracker 코드 정독

### 2.1 클래스 선언과 `__slots__`

```python
# shared/value_objects.py:14-20
class Money:
    """돈을 표현하는 값 객체.

    __slots__: 메모리 최적화. 지정된 속성만 가질 수 있다.
    """
    __slots__ = ("amount", "currency")
```

- 일반 dataclass 가 아닌 **plain class**.
- `__slots__ = ("amount", "currency")` 의 효과 :
  - `__dict__` 를 만들지 않음 → 인스턴스당 메모리 ↓ (보통 200B → 64B 수준).
  - 정의되지 않은 속성 추가 불가 — `m = Money(...); m.foo = 1` → AttributeError.
  - **"불변성을 강제하진 않지만"** — `m.amount = ...` 는 여전히 가능. 단지 *컨벤션* 으로 안 한다는 약속.
- 왜 `@dataclass(frozen=True)` 안 썼는가? — 가능. 단, custom `__init__` (`amount < 0` 검증) 이 필요한데 frozen dataclass 는 `__post_init__` 을 통해 검증해야 해서 살짝 불편. 직접 작성이 더 명시적.

### 2.2 생성자 — 검증 + 기본 통화

```python
# value_objects.py:22-27
def __init__(self, amount: Decimal, currency: str = "KRW") -> None:
    if amount < 0:
        raise ValueError("금액은 음수일 수 없습니다")
    self.amount = amount
    self.currency = currency
```

- **자기 검증** : `amount < 0` 음수 차단. 이게 *Money 의 invariant*.
- **`currency: str = "KRW"`** : 기본 통화. 한국 서비스라 KRW 가 디폴트 — 실수로 currency 안 적어도 KRW 로. 글로벌 서비스라면 디폴트 두지 말고 명시 강제 (위험).
- `amount: Decimal` — 타입 힌트는 Decimal. 호출자가 float 을 넣으면? Python 은 런타임 체크 안 함. 명시적 검증 추가하려면 `if not isinstance(amount, Decimal): raise TypeError(...)`.
- **0 은 허용** (`amount < 0` 만 차단). "무료 상품" / "0원 결제" 같은 케이스. 비즈니스 결정.

### 2.3 통화 검사 — 같은 통화끼리만 연산

```python
# value_objects.py:29-32
def _check_currency(self, other: "Money") -> None:
    if self.currency != other.currency:
        raise ValueError(f"통화가 다릅니다: {self.currency} vs {other.currency}")
```

- private 헬퍼. 모든 산술 메서드 시작 부분에서 호출.
- `Money(100, "KRW") + Money(100, "USD")` → 명시적으로 막힘. *환율 변환* 은 별도 객체 (예 : `ExchangeRate.convert(money)`) 의 책임.

### 2.4 산술 연산 — 모두 새 객체 반환

```python
# value_objects.py:34-52
def add(self, other: "Money") -> "Money":
    self._check_currency(other)
    return Money(self.amount + other.amount, self.currency)

def subtract(self, other: "Money") -> "Money":
    self._check_currency(other)
    return Money(self.amount - other.amount, self.currency)

def multiply(self, quantity: int) -> "Money":
    return Money(self.amount * quantity, self.currency)

def apply_rate(self, rate: Decimal) -> "Money":
    return Money(self.amount * rate, self.currency)
```

핵심 :

- **모두 새 Money 반환** — 원본 불변. 함수형 스타일.
- `add` / `subtract` 는 Money + Money. `multiply` 는 Money + int (수량). `apply_rate` 는 Money + Decimal (비율).
- **연산 폐쇄** : Money 연산 결과는 항상 Money. 외부로 Decimal 이 새 나가지 않음.
- **`subtract` 가 음수 결과를 만들면?** : `Money.__init__` 의 `amount < 0` 가 작동 → `ValueError`. **invariant 가 합성에서도 보존**.
  - 예 : `Money(100).subtract(Money(200))` → `ValueError`. 호출자가 사전에 체크해야.
- `multiply(quantity: int)` — 정수만. 0.5 개를 곱하면? Python 은 `Decimal * float` 가능하지만 결과가 float 이 됨. 의도적으로 int 만.
- `apply_rate(rate: Decimal)` — 비율은 Decimal (예 : "0.1" = 10%). 결과 amount 가 Decimal 의 산술 정밀도를 유지.

> **빠진 연산자 오버로딩** : 코드는 `__add__` / `__sub__` / `__mul__` 가 없음. 호출자는 `m1.add(m2)` 라고 써야 하지 `m1 + m2` 안 됨. 의도적 — *연산이 명시적* 이게. 호불호 갈림.

### 2.5 술어 (Predicate)

```python
# value_objects.py:54-57
@property
def is_positive(self) -> bool:
    return self.amount > 0
```

- `@property` — 메서드를 *속성처럼* 호출 (`money.is_positive`). 인자 없는 술어에 적절.
- "is_positive" 같은 도메인 술어를 두면 호출 코드가 `if order.total.is_positive:` 처럼 자연어에 가까움.

### 2.6 동등성 + 비교

```python
# value_objects.py:59-71
def __eq__(self, other: object) -> bool:
    if not isinstance(other, Money):
        return NotImplemented
    return self.amount == other.amount and self.currency == other.currency

def __gt__(self, other: "Money") -> bool:
    self._check_currency(other)
    return self.amount > other.amount

def __ge__(self, other: "Money") -> bool:
    self._check_currency(other)
    return self.amount >= other.amount
```

읽어보기 :

- `__eq__` — Money 끼리만 비교. 다른 타입은 `NotImplemented` 반환 — Python 이 reverse 비교 시도하게 함 (`other.__eq__(self)`). 그래도 답 없으면 `False`.
  - **`return False` 가 아니라 `return NotImplemented` 인 이유** : `False` 반환하면 `Money(100) == "100"` 이 False 로 *확정* → 다른 타입의 가능성을 차단. `NotImplemented` 는 "내가 모르겠다, 다른 쪽에 물어봐" 라 Python 이 반대편의 `__eq__` 를 호출.
- `__gt__` / `__ge__` 는 통화 검사 후 비교. 다른 통화 비교는 *예외*, 거짓이 아니라.
- **`__lt__`, `__le__` 가 없음** — 의도적 또는 누락. 있으면 더 완전.
- `__ne__` 도 없음 — Python 은 `__eq__` 가 있으면 자동으로 `__ne__` 를 반대로 정의해줌.

### 2.7 해싱

```python
# value_objects.py:73-75
def __hash__(self) -> int:
    return hash((self.amount, self.currency))
```

- **Value object 가 hashable 해야 하는 이유** : dict 키, set 원소. 예 : "통화별 합계" 를 dict 로 만들 때 Money 가 키.
- `__hash__` 가 있으려면 `__eq__` 와 *consistent* 해야 함 — `a == b` 면 `hash(a) == hash(b)`. tuple 로 묶어 hash 위임.
- `__eq__` 만 정의하고 `__hash__` 정의 안 하면 Python 이 *자동으로 None 으로* 만들어 hashable 이 깨짐. 그래서 명시.
- 단, **mutable 한 데이터를 hash 하면 위험** — Money 는 슬롯만 있고 컨벤션상 불변이므로 OK.

### 2.8 표현 (debugging)

```python
# value_objects.py:77-78
def __repr__(self) -> str:
    return f"Money({self.amount}, '{self.currency}')"
```

- `print(money)` / 디버거 / pytest 실패 메시지에서 보임.
- `__str__` 미정의 → `__repr__` 가 fallback.
- 구분 시작 / 끝 표현이 명확 (`Money(100, 'KRW')` 가 평문 "100 KRW" 보다 낫다 — 대상 클래스가 즉시 보임).

---

## §3 직접 구현 — Value Object 의 다른 사례

### 3.1 Email

```python
import re
from dataclasses import dataclass

@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self):
        if not re.match(r"[^@]+@[^@]+\.[^@]+", self.value):
            raise ValueError(f"잘못된 이메일: {self.value}")

    @property
    def domain(self) -> str:
        return self.value.split("@", 1)[1]

# 사용
e = Email("alice@example.com")
e.domain                      # "example.com"
Email("not-an-email")         # ValueError
```

- frozen dataclass 로 불변 + auto __eq__ + auto __hash__ + auto __init__.
- `__post_init__` 로 자기 검증.
- `domain` 같은 도메인 술어.

### 3.2 DateRange

```python
@dataclass(frozen=True)
class DateRange:
    start: date
    end: date

    def __post_init__(self):
        if self.end < self.start:
            raise ValueError("end < start")

    def days(self) -> int:
        return (self.end - self.start).days + 1

    def overlaps(self, other: "DateRange") -> bool:
        return not (self.end < other.start or other.end < self.start)
```

### 3.3 Latitude / Longitude / Coordinate

```python
@dataclass(frozen=True)
class Latitude:
    value: float
    def __post_init__(self):
        if not -90 <= self.value <= 90:
            raise ValueError("위도는 -90~90")

@dataclass(frozen=True)
class Longitude:
    value: float
    def __post_init__(self):
        if not -180 <= self.value <= 180:
            raise ValueError("경도는 -180~180")

@dataclass(frozen=True)
class Coordinate:
    lat: Latitude
    lng: Longitude
```

→ `Coordinate(Latitude(37.5), Longitude(127.0))`. lat / lng 를 바꿔 넣는 실수 (전형적 버그) 가 컴파일 시 잡힘.

### 3.4 ShopTracker 가 추가로 만들 만한 VO

- `CustomerName` (3~50자 검증, strip)
- `Quantity` (1 이상 정수)
- `DiscountRate` (0 ≤ x ≤ 1)
- `TrackingNumber` (포맷 검증)

각각 원시값 함정을 막아주는 작은 객체.

---

## §4 함정

### 4.1 float 사용

```python
Money(0.1) + Money(0.2)
# 만약 Decimal 이 아니라 float 이면 → 0.30000000000000004
```

→ 회계 / 결제 시스템에서는 **반드시 Decimal**. ShopTracker 는 타입 힌트 + Decimal 산술로 안전.

### 4.2 호출자가 Decimal 안 쓰고 int / float 넘김

```python
Money(100)         # ← Decimal 아님. 작동은 함 (Python 은 동적).
Money(100.5)       # ← float. amount 가 float 이 되어 부동소수점 오염
```

해결 : `__init__` 에서 `if not isinstance(amount, Decimal): raise TypeError(...)`. 또는 자동 변환 `amount = Decimal(str(amount))`.

ShopTracker 는 호출자 신뢰 — Pydantic 모델 단계에서 Decimal 변환을 강제.

### 4.3 통화 변환을 Money 자신이 하기

```python
class Money:
    def to_usd(self) -> "Money":   # ← 안 좋음
        rate = SomeApi.get_rate()
        ...
```

문제 :
- Money 가 외부 API 의존 → 인프라 결합.
- Money 가 *순수* 값 객체가 아님.

해결 : 변환은 별도 service.

```python
class CurrencyConverter:
    def __init__(self, rate_provider: ExchangeRateProvider): ...
    def convert(self, money: Money, to: str) -> Money: ...
```

### 4.4 산술 결과의 정밀도

`Decimal("100") * Decimal("0.105")` → `Decimal("10.500")`. 끝의 0 이 의미를 가짐 (KRW 는 정수만 의미 있는데 소수점 3 자리). 보통 :
- `Money` 가 통화별 *의미 있는 자릿수* 를 알아야 함 (KRW=0, USD=2, JPY=0).
- `quantize` 로 반올림 정책 적용.

ShopTracker 는 단순화 — 확장 시 `Money.normalize()` 메서드 추가.

### 4.5 Money 안에 currency 가 string

```python
Money(Decimal("100"), "KrW")       # ← typo. 다른 통화로 인식.
Money(Decimal("100"), "krw")       # ← 또 다른 typo
```

해결 : Currency Enum.

```python
class Currency(str, Enum):
    KRW = "KRW"
    USD = "USD"
    EUR = "EUR"

Money(Decimal("100"), Currency.KRW)
```

### 4.6 mutable container 안에 있는 Money

```python
prices: list[Money] = [Money(...), Money(...)]
prices[0] = Money(...)              # OK — 리스트가 mutable, Money 는 그대로
```

Money 자체는 불변이지만 담는 컨테이너가 mutable 일 수 있음. 의도 따라 컨테이너도 `tuple` 로 :

```python
prices: tuple[Money, ...] = (Money(...), Money(...))
```

---

## §5 다른 언어 / 프레임워크

### 5.1 Java

```java
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        if (amount.signum() < 0) throw new IllegalArgumentException();
        this.amount = amount;
        this.currency = currency;
    }

    public Money add(Money other) {
        check(other);
        return new Money(amount.add(other.amount), currency);
    }

    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
}
```

`final class`, `private final` 필드 → 진짜 불변. Python 보다 강제력이 강함.

### 5.2 Kotlin / Scala — `data class` / `case class`

`equals`, `hashCode`, `toString` 자동. ShopTracker 의 `@dataclass(frozen=True)` 와 같은 사상. Money 같은 값 객체에 최적.

### 5.3 Rust — `#[derive(...)]`

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct Money { amount: i64, currency: Currency }
```

`Copy` 가 있으면 모든 패스가 값 복사 — 진짜 불변.

### 5.4 라이브러리

- Python : `py-moneyed` (Money + Currency + 환율)
- Java : `joda-money`
- JS : `dinero.js`

직접 구현 vs 라이브러리 — 학습 단계는 직접.

---

## §6 테스트 전략

### 6.1 invariant

```python
def test_negative_money_rejected():
    with pytest.raises(ValueError):
        Money(Decimal("-1"))

def test_currency_mismatch_rejected():
    with pytest.raises(ValueError):
        Money(Decimal("100"), "KRW").add(Money(Decimal("100"), "USD"))
```

### 6.2 동등성 / 해싱

```python
def test_equality_by_value():
    assert Money(Decimal("100"), "KRW") == Money(Decimal("100"), "KRW")
    assert Money(Decimal("100"), "KRW") != Money(Decimal("100"), "USD")

def test_hash_consistent_with_eq():
    a = Money(Decimal("100"), "KRW")
    b = Money(Decimal("100"), "KRW")
    assert hash(a) == hash(b)

def test_money_usable_as_dict_key():
    d = {Money(Decimal("100"), "KRW"): "small", Money(Decimal("1000"), "KRW"): "big"}
    assert d[Money(Decimal("100"), "KRW")] == "small"
```

### 6.3 산술이 새 객체 반환

```python
def test_add_returns_new_instance():
    a = Money(Decimal("100"))
    b = Money(Decimal("50"))
    c = a.add(b)
    assert c.amount == Decimal("150")
    assert a.amount == Decimal("100")    # 원본 불변
    assert c is not a
```

### 6.4 합성에서 invariant 보존

```python
def test_subtract_to_negative_raises():
    a = Money(Decimal("100"))
    b = Money(Decimal("200"))
    with pytest.raises(ValueError):
        a.subtract(b)        # 결과가 -100 → 생성자에서 차단
```

---

## §10 학습 포인트 (한 줄 요약)

1. **Value Object** = 식별자 없음 + 값 동등성 + 불변 + 자기 검증.
2. **원시값 (Decimal, str) 그대로 다니지 마라** — 타입이 도메인 의미를 담아야.
3. **Money 의 invariant** : 음수 금지 + 같은 통화끼리만 연산.
4. **모든 산술이 새 인스턴스 반환** — 함수형 스타일, 부수효과 0.
5. **연산 폐쇄** : Money + Money = Money. 도메인 안에서 닫힘.
6. **`__eq__` + `__hash__` 둘 다 정의** — dict / set 사용 가능. Mutable 이면 hash 금지.
7. **`__slots__`** — 메모리 ↓ + 속성 추가 차단. frozen dataclass 도 같은 효과.
8. **통화 변환은 외부 service 책임** — Money 는 순수.
9. **float 절대 금지** — 회계는 Decimal.
10. **다른 VO 도 같은 패턴** — Email, DateRange, Coordinate. 작은 타입을 많이 만들어라.

---

## 추가 참고

- Eric Evans, *DDD*, Value Object 챕터
- Martin Fowler, *Money Pattern*, https://martinfowler.com/eaaCatalog/money.html
- ShopTracker 다음 글 : `10-mapper-orm-domain.md` (Entity 와 SQLAlchemy Model 의 분리)
