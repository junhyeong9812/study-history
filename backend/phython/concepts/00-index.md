# Python Concepts — 기본기 / 내부 동작

> `study/14~19` 가 *실용 (타입 힌트, dataclass, async, 데코레이터)* 였다면, 이 폴더는 그 아래에 깔린 *Python 자체의 동작 원리* 를 다룬다. CPython 의 바이트코드, GIL, 메모리, 객체 모델, MRO. ShopTracker 코드를 *왜 그렇게 짜야 하는가* 의 1 단계 아래 답.

---

## 학습 순서

| # | 제목 | 한 줄 |
|---|---|---|
| 01 | CPython 실행 모델 | 소스 → AST → 바이트코드 → VM. `dis` 로 들여다보기 |
| 02 | GIL | 왜 멀티스레드가 CPU 병렬에 무력한가 |
| 03 | 메모리 관리 | Reference counting + 세대 GC |
| 04 | 객체 모델 | "Everything is object", type 과 class |
| 05 | mutable / id / hash | `is` vs `==`, `__hash__` 와 `__eq__` 의 계약 |
| 06 | scope / LEGB / closure | 변수가 어디서 찾아지는가 |
| 07 | iterator / generator | `for` 의 내부, lazy evaluation |
| 08 | 함수의 정체 | first-class, lambda, partial, signature |
| 09 | 클래스 / MRO | super, C3 linearization, descriptor, metaclass |
| 10 | 예외 모델 | EAFP, try-except-else-finally, exception group |
| 11 | 동시성 비교 | threading vs multiprocessing vs asyncio — 언제 무엇을 |
| 12 | 문자열 / 숫자 | unicode, encoding, Decimal/float 정밀도 |

---

## 어떻게 읽는가

- 순서대로 읽으면 *Python 의 사고방식* 이 잡힌다.
- 각 글의 §0 (문제) → §1 (메커니즘) → §2 (실험) → §함정.
- 모든 코드는 손으로 쳐서 `python -i` 로 확인 가능.

---

## 무엇이 빠져 있는가

- C 확장 / CFFI : 별도 깊은 주제 (이 시리즈 밖)
- PyPy / Cython 등 대체 구현 : 비교 한 줄만 언급
- 패턴 / 디자인 : `study/01~13` 의 영역
- 실용 도구 (typing, dataclass 등) : `study/14~19` 의 영역
