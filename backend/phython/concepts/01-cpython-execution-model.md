# 01 — CPython 실행 모델 (소스 → 바이트코드 → VM)

> Python 은 *"인터프리터 언어"* 라고 흔히 말하지만, 실제로는 컴파일 (소스 → 바이트코드) 후 가상 머신 (CPython VM) 이 실행한다. Java 와 구조가 닮았다 (`javac` → `.class` → JVM). 이 글은 그 5 단계를 들여다본다.

---

## §0 우리가 `python foo.py` 칠 때 뭐가 일어나나

```
foo.py (텍스트)
   │
   ▼  ① 토크나이저 (tokenize)
[NAME 'def', NAME 'foo', OP '(', OP ')', OP ':', ...]
   │
   ▼  ② 파서 (parse) — PEG parser (3.9+)
AST (Abstract Syntax Tree)
   │
   ▼  ③ 컴파일러 (compile)
바이트코드 (bytecode) — code object
   │
   ▼  ④ 마샬 (marshal) → __pycache__/foo.cpython-312.pyc
   │
   ▼  ⑤ CPython VM (ceval.c) 의 evaluation loop
실행
```

각 단계가 *직접 들여다볼 수 있는 객체* 를 만든다 — 이게 Python 의 디버깅 가능성의 출처.

---

## §1 토큰 보기 — `tokenize` 모듈

```python
import tokenize
from io import BytesIO

code = b"x = 1 + 2\n"
for tok in tokenize.tokenize(BytesIO(code).readline):
    print(tok)
```

```
TokenInfo(type=ENCODING, string='utf-8', ...)
TokenInfo(type=NAME, string='x', ...)
TokenInfo(type=OP, string='=', ...)
TokenInfo(type=NUMBER, string='1', ...)
TokenInfo(type=OP, string='+', ...)
TokenInfo(type=NUMBER, string='2', ...)
TokenInfo(type=NEWLINE, ...)
```

토큰 = *어휘 단위*. 띄어쓰기, 들여쓰기 (`INDENT`/`DEDENT`) 도 토큰.

---

## §2 AST — `ast` 모듈

```python
import ast
tree = ast.parse("x = 1 + 2")
print(ast.dump(tree, indent=2))
```

```
Module(
  body=[
    Assign(
      targets=[Name(id='x', ctx=Store())],
      value=BinOp(
        left=Constant(value=1),
        op=Add(),
        right=Constant(value=2)))],
  type_ignores=[])
```

AST = *문법 구조의 트리*. 모든 정적 분석 도구 (mypy, ruff, pylint, ast-grep) 가 이걸 본다.

ast 의 활용 :
- 코드 변형 (`ast.NodeTransformer`)
- linter 작성
- 매크로 시뮬

---

## §3 바이트코드 — `dis` 모듈

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
```

```
  2           0 RESUME                   0
  3           2 LOAD_FAST                0 (a)
              4 LOAD_FAST                1 (b)
              6 BINARY_OP                0 (+)
             10 RETURN_VALUE
```

각 줄 = *한 바이트코드 명령*.
- `LOAD_FAST 0` : 로컬 슬롯 0 (a) 을 스택에 push.
- `BINARY_OP 0` : 스택 위 두 값에 + 연산.
- `RETURN_VALUE` : 스택 top 을 함수 반환값으로.

CPython 의 VM 은 **stack-based** — JVM 도 같은 모델. (반면 Lua/V8 일부는 register-based.)

### 3.1 code object

```python
print(add.__code__)
# <code object add at 0x..., file "<stdin>", line 1>

print(add.__code__.co_code)        # bytes (raw bytecode)
print(add.__code__.co_varnames)    # ('a', 'b')
print(add.__code__.co_consts)      # (None,)
print(add.__code__.co_names)       # ()
```

함수의 본질 = `code object` + `__globals__` + `__defaults__` 등.

### 3.2 dis 의 다른 활용

```python
dis.dis(compile("[x*2 for x in range(10)]", "<str>", "eval"))
```

→ 리스트 컴프리헨션이 어떻게 컴파일되는지 (3.12 에서 inline 화됨, 이전엔 별도 함수).

---

## §4 .pyc 파일 — `__pycache__/`

`python foo.py` 를 처음 실행하면 :
- `__pycache__/foo.cpython-312.pyc` 가 생김.
- 이 파일 = 마샬 된 code object.
- 다음 실행에서 *.py 가 안 바뀌었으면* .pyc 로 직접 점프 — 컴파일 단계 생략.

`.pyc` 헤더 :
- magic number (Python 버전)
- timestamp / 소스 size
- code object (marshal)

```python
import dis, marshal, importlib.util
with open("__pycache__/foo.cpython-312.pyc", "rb") as f:
    f.read(16)                # 헤더 스킵 (3.7+)
    code = marshal.load(f)
    dis.dis(code)
```

---

## §5 CPython VM — evaluation loop

`ceval.c` 안의 거대한 switch 루프 :

```c
for (;;) {
    opcode = NEXTOP();
    switch (opcode) {
        case LOAD_FAST: ...; break;
        case BINARY_OP: ...; break;
        case RETURN_VALUE: ...; break;
        ...
    }
}
```

- 한 명령씩 fetch & dispatch.
- 스택 (`f_valuestack`) 에 값을 push/pop.
- 프레임 (`PyFrameObject`) 이 함수 호출 단위 — 로컬 변수 / 명령 포인터 / 예외 상태.

Python 3.11 부터 *Specializing Adaptive Interpreter* (PEP 659) — 자주 실행되는 명령을 *특화 (specialize)* 해서 속도 ↑. 3.13 의 JIT 실험도 같은 흐름.

---

## §6 다른 Python 구현

| 구현 | 특징 |
|---|---|
| **CPython** | 표준. C 로 작성. 우리가 쓰는 그것. |
| **PyPy** | JIT 컴파일러. CPython 보다 5~10x 빠름 (warmup 후). C 확장 호환성 일부 X. |
| **Jython** | JVM 위에서 실행. 더 이상 활발하지 않음. |
| **IronPython** | .NET CLR 위. |
| **MicroPython** | 임베디드. 메모리 적게. |
| **Pyodide** | WebAssembly 빌드. 브라우저에서 실행. |

ShopTracker 는 CPython 3.12 만 가정.

---

## §7 직접 실험 — 같은 표현이 다르게 컴파일

```python
import dis

# (1) 함수 안 변수
def f():
    x = 1
    return x

# (2) 모듈 레벨 변수
g_x = 1
def g():
    return g_x

dis.dis(f); print("---"); dis.dis(g)
```

차이 :
- `f` : `STORE_FAST 0`, `LOAD_FAST 0` — 로컬 슬롯 (배열 인덱스).
- `g` : `LOAD_GLOBAL 0 (g_x)` — 이름으로 dict 조회 (느림).

**이게 "함수 안에서 글로벌 접근하지 말라" 의 성능적 이유**. 06 장 LEGB 와 연결.

---

## §8 함정

### 8.1 .pyc 가 stale

수동으로 .pyc 만 복사 + .py 수정 → magic/timestamp 일치 안 하면 재컴파일. 캐싱은 timestamp 기반이라 *시간 동기화 안 된 환경* (Docker bind mount) 에서 가끔 이상 동작.

### 8.2 dis 결과가 버전마다 다름

3.10, 3.11, 3.12 가 명령 집합을 자주 바꿈. 책 / 블로그의 dis 출력은 버전 명시 확인.

### 8.3 컴파일 시점 vs 실행 시점

```python
def f(x=[]):     # ← x 의 default 는 *함수 정의 시점* 에 한 번 평가
    x.append(1)
    return x

f(); f(); f()    # [1, 1, 1] — 같은 list!
```

mutable default 의 함정 — 컴파일이 아닌 *함수 객체 생성 시점*에 default 가 한 번 만들어짐.

---

## §10 학습 포인트 (한 줄 요약)

1. **소스 → 토큰 → AST → 바이트코드 → VM** 의 5 단계.
2. **`tokenize` / `ast` / `dis`** — 각 단계를 직접 본다.
3. **CPython VM = stack-based** — `LOAD_FAST` / `BINARY_OP`.
4. **code object** = 함수의 본질. `__code__` 로 접근.
5. **.pyc** = marshal 된 code object 캐시.
6. **PEP 659 specializing interpreter (3.11+)** — 자주 쓰는 명령 가속.
7. **로컬 (LOAD_FAST) > 글로벌 (LOAD_GLOBAL)** 성능.
8. **PyPy / MicroPython** — 같은 언어, 다른 구현.
9. **mutable default** = 함수 정의 시점에 평가됨.
10. **dis 출력은 버전마다 다름** — 명시 확인.

---

## 참고
- CPython 소스 : `Python/ceval.c`
- PEP 659 : Specializing Adaptive Interpreter
- Brett Cannon, "From source to code" 시리즈
