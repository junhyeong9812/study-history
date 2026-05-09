# G2P (Grapheme-to-Phoneme) — 영어를 한글 발음으로

> "STARBOOKS" → "스타북스" 변환. 트레이드마크 발음 유사 검색의 핵심.

---

## 0. 분석 대상 코드

```python
# trademark-name-search/g2p/g2p_korean_converter.py
"""
g2p-en 기반 영어→한글 발음 변환기

흐름: English → ARPAbet → Korean
예시: "SAMSUNG" → ['S', 'AE1', 'M', 'S', 'AH0', 'NG'] → "삼성"
예시: "starbooks" → ['S', 'T', 'AA1', 'R', 'B', 'UH2', 'K', 'S'] → "스타북스"
"""
import logging
from typing import Optional, List, Tuple
from dataclasses import dataclass


@dataclass
class G2pResult:
    korean: str                # 변환된 한글
    phonemes: List[str]        # ARPAbet 음소 리스트
    mapping_log: List[str]     # 매핑 로그 (디버깅용)


class G2pKoreanConverter:
    HANGUL_BASE = 0xAC00  # '가'
    CHOSUNG_COUNT = 19
    JUNGSUNG_COUNT = 21
    JONGSUNG_COUNT = 28

    CHOSUNG_LIST = ['ㄱ', 'ㄲ', 'ㄴ', 'ㄷ', 'ㄸ', 'ㄹ', 'ㅁ', 'ㅂ', 'ㅃ', 'ㅅ',
                    'ㅆ', 'ㅇ', 'ㅈ', 'ㅉ', 'ㅊ', 'ㅋ', 'ㅌ', 'ㅍ', 'ㅎ']
    JUNGSUNG_LIST = ['ㅏ', 'ㅐ', 'ㅑ', 'ㅒ', 'ㅓ', 'ㅔ', 'ㅕ', 'ㅖ', 'ㅗ', 'ㅘ',
                     'ㅙ', 'ㅚ', 'ㅛ', 'ㅜ', 'ㅝ', 'ㅞ', 'ㅟ', 'ㅠ', 'ㅡ', 'ㅢ', 'ㅣ']
    JONGSUNG_LIST = ['', 'ㄱ', 'ㄲ', 'ㄳ', 'ㄴ', 'ㄵ', 'ㄶ', 'ㄷ', 'ㄹ', 'ㄺ',
                     'ㄻ', 'ㄼ', 'ㄽ', 'ㄾ', 'ㄿ', 'ㅀ', 'ㅁ', 'ㅂ', 'ㅄ', 'ㅅ',
                     'ㅆ', 'ㅇ', 'ㅈ', 'ㅊ', 'ㅋ', 'ㅌ', 'ㅍ', 'ㅎ']

    CONSONANT_MAP = {
        'B': 'ㅂ', 'CH': 'ㅊ', 'D': 'ㄷ', 'DH': 'ㄷ', 'F': 'ㅍ', 'G': 'ㄱ',
        'HH': 'ㅎ', 'JH': 'ㅈ', 'K': 'ㅋ', 'L': 'ㄹ', 'M': 'ㅁ', 'N': 'ㄴ',
        'NG': 'ㅇ',  # 받침 ㅇ (종성 전용)
        'P': 'ㅍ', 'R': 'ㄹ', 'S': 'ㅅ', 'SH': 'ㅅ',  # sh → 시/쉬
        'T': 'ㅌ', 'TH': 'ㅅ',  # think → 싱크
        # ...
    }
```

자모 분해 유틸:

```python
# trademark-name-search/g2p/jamo.py
HANGUL_BASE = 0xAC00
HANGUL_END = 0xD7A3

CHO_LIST = list("ㄱㄲㄴㄷㄸㄹㅁㅂㅃㅅㅆㅇㅈㅉㅊㅋㅌㅍㅎ")
JUNG_LIST = list("ㅏㅐㅑㅒㅓㅔㅕㅖㅗㅘㅙㅚㅛㅜㅝㅞㅟㅠㅡㅢㅣ")
JONG_LIST = ["", "ㄱ", "ㄲ", "ㄳ", "ㄴ", "ㄵ", "ㄶ", "ㄷ",
             "ㄹ", "ㄺ", "ㄻ", "ㄼ", "ㄽ", "ㄾ", "ㄿ", "ㅀ",
             "ㅁ", "ㅂ", "ㅄ", "ㅅ", "ㅆ", "ㅇ", "ㅈ", "ㅊ",
             "ㅋ", "ㅌ", "ㅍ", "ㅎ"]

CHO_MAP = {c: i for i, c in enumerate(CHO_LIST)}
JUNG_MAP = {c: i for i, c in enumerate(JUNG_LIST)}
JONG_MAP = {c: i for i, c in enumerate(JONG_LIST) if c}

DOUBLE_JONG = {
    3:  (1,  9),   # ㄳ → ㄱ + ㅅ
    5:  (4,  12),  # ㄵ → ㄴ + ㅈ
    9:  (8,  0),   # ㄺ → ㄹ + ㄱ
    # ...
}
```

---

## 1. G2P 의 두 단계

### 1.1 영어 → ARPAbet
- ARPAbet: 영어 발음을 표기하는 음소 시스템.
- "SAMSUNG" → `['S', 'AE1', 'M', 'S', 'AH0', 'NG']`
- 숫자(0/1/2)는 강세 표시.
- 사용 라이브러리: `g2p-en` (CMU dictionary 기반).

```python
import g2p_en
g2p = g2p_en.G2p()
phonemes = g2p("SAMSUNG")  # → ['S', 'AE1', 'M', 'S', 'AH0', 'NG']
```

### 1.2 ARPAbet → 한글 음절
- ARPAbet 의 자음/모음을 한글 자모로 매핑.
- 자음 + 모음 + (받침) → 한글 음절 합성.

핵심 매핑:
- `S` → ㅅ, `M` → ㅁ, `K` → ㅋ
- `AE` → ㅐ (cat), `AH` → ㅓ/ㅏ (sun, but)
- `NG` → 받침 ㅇ (sing)
- `R` → ㄹ (rotic)
- `TH` → ㅅ (think)

→ "S AE1 M" → "샘", "S AH0 NG" → "성".

---

## 2. 한글 유니코드 합성 공식

```python
HANGUL_BASE = 0xAC00  # '가'
CHOSUNG_COUNT = 19
JUNGSUNG_COUNT = 21
JONGSUNG_COUNT = 28

# 한글 음절 코드포인트 = 0xAC00 + (초성 × 21 + 중성) × 28 + 종성
def compose(cho_idx, jung_idx, jong_idx=0):
    return chr(HANGUL_BASE + (cho_idx * 21 + jung_idx) * 28 + jong_idx)

compose(9, 0, 21)  # → '강'  (ㄱ + ㅏ + ㅇ)
```

### 분해
```python
def decompose(c):
    if not (0xAC00 <= ord(c) <= 0xD7A3):
        return None
    n = ord(c) - 0xAC00
    cho = n // (21 * 28)
    jung = (n % (21 * 28)) // 28
    jong = n % 28
    return CHO_LIST[cho], JUNG_LIST[jung], JONG_LIST[jong]
```

→ "삼" 분해: ord("삼") - 0xAC00 = 12152
- 12152 // 588 = 20 → ㅅ (오타: 정확히는 9). 단순 산식 검증은 실제 코드 참고.

---

## 3. 자모 인덱스 빠른 매핑

```python
CHO_LIST = list("ㄱㄲㄴㄷㄸㄹㅁㅂㅃㅅㅆㅇㅈㅉㅊㅋㅌㅍㅎ")
CHO_MAP = {c: i for i, c in enumerate(CHO_LIST)}
```

→ "ㅅ" → 9, "ㅁ" → 6 처럼 즉시 인덱스 조회.

`enumerate` + dict comprehension 으로 한 줄 매핑.

---

## 4. 겹받침 처리

한글 받침에는 ㄳ, ㄺ 같은 **겹받침** 28개 중 12개. 발음/연음 시 분해 필요.

```python
DOUBLE_JONG = {
    # 받침 인덱스 → (앞 자음 종성 인덱스, 뒤 자음 초성 인덱스)
    3:  (1,  9),   # ㄳ → ㄱ + ㅅ
    9:  (8,  0),   # ㄺ → ㄹ + ㄱ
    11: (8,  7),   # ㄼ → ㄹ + ㅂ
    18: (17, 9),   # ㅄ → ㅂ + ㅅ
}
```

연음 시: "값이" → "갑시"
- "값" 의 받침 ㅄ (인덱스 18) → 앞 ㅂ (종성 17), 뒤 ㅅ (초성 9).
- "이" 의 초성 ㅇ 자리에 ㅅ 이 들어감.

대표음:
```python
DOUBLE_JONG_DEFAULT = {
    3:  1,   # ㄳ → ㄱ
    9:  1,   # ㄺ → ㄱ (자음 앞)
    # ...
}
```

→ 자음 앞에서 발음되는 단일 자음.

---

## 5. 검색 활용 — 자모 인덱스

ES 매핑에서 자모 분리한 필드를 추가:

```python
# 색인 시
trademark_name_jamo = jamo_split("삼성")  # "ㅅㅏㅁㅅㅓㅇ"

doc = {
    "trademark_name": "삼성",
    "trademark_name_jamo": trademark_name_jamo,
}
```

검색 시 사용자가 "ㅅㅅ" 만 입력해도 자모 매칭으로 "삼성" 발견.

---

## 6. 발음 유사 검색

```python
# 색인
korean_pron = G2pKoreanConverter().convert("STARBOOKS")  # "스타북스"
doc["english_korean_pron"] = korean_pron

# 검색
사용자가 "스타북스" 입력 → english_korean_pron 필드 매칭 → STARBOOKS 트레이드마크 hit
```

→ 영문 트레이드마크를 한글로 검색 가능.

---

## 7. 함정과 한계

### 7.1 동음이의 처리
- "Lead" 는 "리드" 또는 "레드" (명사/동사 차이).
- g2p-en 은 단어 단위 추론 — 문맥 고려 안 함.
- 결과가 항상 정답은 아님.

### 7.2 고유명사 처리
- "Samsung" 같은 고유명사는 사전에 따라 결과가 다름.
- 일부 트레이드마크는 의도적 변형 (예: "Lyft" → "리프트").

### 7.3 외래어 표기법 vs 발음 검색
- 표준 외래어 표기법: "Starbucks" → "스타벅스".
- 사용자 검색: "스타북스", "스타박스" 등 변형.
- 두 결과를 모두 색인하면 recall ↑.

### 7.4 성능
- g2p-en 은 단어당 ~ms.
- 대량 색인 시 5천만 건 × ms = 수시간.
- 워커 자원 캐시 (한 번 로드 후 워커 수명 내 재사용) 필수.

---

## 8. 응용 포인트

- 다국어 트레이드마크 검색은 G2P + 자모 인덱스 결합 필수.
- 한글 자모 합성/분해 공식: `0xAC00 + (cho × 21 + jung) × 28 + jong`.
- enum 매핑은 list + `{c: i for i, c in enumerate(list)}` 한 줄로.
- ARPAbet ↔ 한글 매핑은 음운적 근사 — 완벽한 변환은 어려움.
- 검색 recall 을 위해 외래어 표기법과 사용자 변형 모두 색인.
- 대량 처리 시 워커 자원 캐시 + ProcessPool.
