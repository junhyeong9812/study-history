# 12 — 문자열 / 숫자 / 인코딩

> Python 3 의 `str` 은 *유니코드 코드 포인트의 시퀀스*, `bytes` 는 *바이트의 시퀀스*. `0.1 + 0.2 != 0.3` 의 진실. ShopTracker 의 Money 가 왜 Decimal 인지. 이 글은 Python 의 문자/숫자 표현의 메커니즘.

---

## §0 str vs bytes (Python 2 → 3 의 가장 큰 변화)

```python
"hello"          # str — 유니코드
b"hello"         # bytes — 바이트
"한글"            # str
"한글".encode("utf-8")   # b'\xed\x95\x9c\xea\xb8\x80'
b'\xed\x95\x9c\xea\xb8\x80'.decode("utf-8")  # "한글"
```

| | str | bytes |
|---|---|---|
| 표현 | 유니코드 코드 포인트 | 바이트 (0~255) |
| literal | `"..."`, `'...'` | `b"..."` |
| 인덱싱 | 코드 포인트 | int (0~255) |
| 변환 | `.encode(enc)` → bytes | `.decode(enc)` → str |
| 사용 | 텍스트 | 파일/네트워크/이진 |

Python 3 = *명확한 분리*. 자동 변환 X. 안전성 ↑, 처음엔 헷갈림.

---

## §1 유니코드 / 인코딩

### 1.1 유니코드 = 문자에 번호

- "A" = U+0041
- "한" = U+D55C
- "🎉" = U+1F389

코드 포인트 = 추상적 *번호*. 메모리에 어떻게 저장할지는 인코딩이 결정.

### 1.2 인코딩 = 코드 포인트 → 바이트

| 인코딩 | 특징 |
|---|---|
| **UTF-8** | 가변 길이 (1~4 byte). ASCII 호환. 표준. |
| **UTF-16** | 가변 (2 또는 4 byte). Windows / Java 내부. |
| **UTF-32** | 고정 4 byte. 메모리 비효율. |
| **EUC-KR** | 한글 (옛날). |
| **CP949** | EUC-KR + 확장 (Windows 한글). |
| **ASCII** | 0~127 만. 영문. |

```python
"한".encode("utf-8")    # b'\xed\x95\x9c' (3 byte)
"한".encode("utf-16")   # b'\xff\xfe\\\xd5' (BOM + 2 byte)
"한".encode("euc-kr")   # b'\xc7\xd1' (2 byte)
"한".encode("ascii")    # UnicodeEncodeError
```

### 1.3 항상 UTF-8

웹 / DB / 파일 거의 다 UTF-8. 다른 인코딩은 *legacy* 또는 *특수 환경*.

```python
# 파일 읽기 (Python 3.7+ 기본 utf-8 가 아닐 수 있음)
with open(path, encoding="utf-8") as f:
    ...

# Python 3.15+ : utf-8 default 강제
```

명시적으로 `encoding="utf-8"`.

---

## §2 str 의 내부

CPython 의 str 은 *flexible representation* (PEP 393, 3.3+) :

```
모든 문자가 ASCII   → 1 byte / char (Latin-1 encoding 내부)
모든 문자가 BMP     → 2 byte / char (UCS-2)
4-byte 문자 포함    → 4 byte / char (UCS-4)
```

→ 메모리 적응적. ASCII-only 문자열은 작음.

```python
import sys
sys.getsizeof("hello")     # 54 (ASCII)
sys.getsizeof("한글")        # 76 (2 byte/char + 헤더)
sys.getsizeof("🎉")          # 80 (4 byte/char)
```

### 2.1 immutable

```python
s = "hello"
s[0] = "H"      # TypeError
s = s.upper()   # 새 객체
```

문자열 + 문자열 반복 = 매번 새 객체 → O(n²). join 이 더 빠름.

```python
# bad
s = ""
for x in xs: s += x

# good
s = "".join(xs)
```

---

## §3 f-string (PEP 498, 3.6+)

```python
name = "alice"; age = 30
f"{name} is {age}"             # "alice is 30"
f"{name=}"                      # "name='alice'"  (3.8+)
f"{age:>5}"                     # "   30"
f"{3.14159:.2f}"               # "3.14"
f"{date:%Y-%m-%d}"
f"{x!r}"                        # repr(x) 호출
```

vs `.format()`, `%` :
- f-string = 가장 빠르고 가독성 ★. 우선.
- `.format()` = 동적 템플릿.
- `%` = legacy.

ShopTracker 의 모든 문자열 포맷이 f-string.

---

## §4 숫자 타입

### 4.1 int

- *임의 정밀도* — 메모리 한도 내 무한 자릿수.
- 작은 정수 (-5 ~ 256) 캐시 (03 장).

```python
2**1000                # 작동, 거대 정수
sys.int_info           # bits per digit etc
```

### 4.2 float

- IEEE 754 double precision (64 bit).
- 약 15~17 유효 자리.
- 정확하지 않음.

```python
0.1 + 0.2              # 0.30000000000000004
0.1 + 0.2 == 0.3       # False
```

→ 비교는 `math.isclose(a, b)`.

### 4.3 Decimal

- 정확한 십진수.
- 금융 / 회계 필수.

```python
from decimal import Decimal, getcontext
Decimal("0.1") + Decimal("0.2")    # Decimal("0.3") 정확

# 정밀도 설정
getcontext().prec = 50
```

ShopTracker 의 Money 가 Decimal (09 장 §4.1).

### 4.4 Fraction

```python
from fractions import Fraction
Fraction(1, 3) + Fraction(1, 6)    # Fraction(1, 2)
```

분수 표현. 거의 안 씀.

### 4.5 complex

```python
1 + 2j
```

수치 계산용.

### 4.6 bool

`True`, `False`. *int 의 서브클래스*.

```python
isinstance(True, int)    # True
True + 1                  # 2
```

---

## §5 숫자 변환

```python
int("42")                # 42
int("0x10", 16)          # 16
int(3.7)                 # 3 (truncate)
float("3.14")            # 3.14
float("inf")             # inf
str(3.14)                # '3.14'

# Decimal 의 함정
Decimal(0.1)             # Decimal('0.10000000000000000555...')  ← float 오차 그대로
Decimal("0.1")           # Decimal('0.1') ← str 경유 정확
Decimal(str(0.1))        # Decimal('0.1') ← 같은 효과
```

→ Decimal 은 *반드시 str 또는 int 에서*. ShopTracker 의 mapper (10 장 §2.4) 가 `Decimal(str(model.total_amount))` 쓰는 이유.

---

## §6 str 의 주요 메서드

```python
s.lower(), s.upper(), s.title(), s.capitalize()
s.strip(), s.lstrip(), s.rstrip()
s.split(), s.split(",")
s.join([...])
s.replace("a", "b")
s.startswith("..."), s.endswith("...")
s.find("..."), s.index("...")    # find 는 -1, index 는 ValueError
s.count("...")
s.format(...)
s.encode("utf-8")

# 검사
s.isalpha(), s.isdigit(), s.isalnum(), s.isspace()
s.isupper(), s.islower()
```

모두 *새 객체 반환* (immutable).

---

## §7 정규표현식

```python
import re

re.match(r"\d+", "123abc")          # at start
re.search(r"\d+", "abc123")         # 어디든
re.findall(r"\d+", "1 a 2 b 3")     # ['1', '2', '3']
re.sub(r"\d+", "N", "1 a 2")        # 'N a N'

# 컴파일
pattern = re.compile(r"\d+")
pattern.match(...)                  # 반복 사용 시 빠름
```

raw string (`r"..."`) 권장 — `\` 가 escape 되지 않음.

---

## §8 함정

### 8.1 float 비교

```python
0.1 + 0.2 == 0.3                # False!
math.isclose(0.1 + 0.2, 0.3)    # True
```

### 8.2 Decimal(float) 정확도

```python
Decimal(0.1)                    # 부정확
Decimal("0.1")                  # 정확
```

### 8.3 인코딩 추측

```python
data = open("file.txt").read()  # 시스템 default encoding 사용 — 환경마다 다름
data = open("file.txt", encoding="utf-8").read()  # 명시
```

### 8.4 BOM

UTF-8 BOM (`﻿`) 이 파일 앞에 붙어 있으면 첫 글자가 BOM. `encoding="utf-8-sig"` 로 자동 제거.

### 8.5 surrogate pair

UTF-16 으로 인코딩된 4-byte 문자가 잘못 들어오면 surrogate. `encode("utf-8", errors="surrogatepass")` 등으로 처리.

### 8.6 String concatenation 의 성능

```python
result = ""
for x in xs:
    result += x         # O(n²) - 매번 새 str
result = "".join(xs)    # O(n)
```

CPython 은 일부 케이스에 최적화 (refcnt 1 이면 in-place 시도) 하지만 의존 X.

### 8.7 bool 이 int

```python
sum([True, False, True])    # 2
[1, 2][True]                 # 2 (True == 1)
```

의도치 않은 결과 가능.

---

## §9 ShopTracker 와 연결

- **Money = Decimal** (09 장).
- **Decimal(str(x))** 변환 패턴 (10 장).
- **f-string** 모든 곳.
- **DB 컬럼** : `String(36)` (UUID), `Numeric(12, 2)` (금액).
- **인코딩** : 모든 파일 / DB / HTTP 가 UTF-8 가정.

---

## §10 학습 포인트 (한 줄 요약)

1. **str = 유니코드, bytes = 바이트** — 명확히 분리 (Python 3).
2. **항상 UTF-8** — 다른 인코딩은 legacy.
3. **str.encode(enc) ↔ bytes.decode(enc)**.
4. **str 은 immutable** — `+=` 루프 X, `join` 사용.
5. **f-string** 우선.
6. **int = 임의 정밀도**.
7. **float = IEEE 754** — 비교는 `math.isclose`.
8. **Decimal = 회계** — `Decimal(str(x))` 변환.
9. **bool 은 int 서브클래스** — sum / 인덱싱 시 의도 확인.
10. **인코딩은 명시** — `encoding="utf-8"`.

---

## 참고

- Unicode HOWTO : https://docs.python.org/3/howto/unicode.html
- decimal docs
- "Joel Spolsky on Unicode" 1.0
