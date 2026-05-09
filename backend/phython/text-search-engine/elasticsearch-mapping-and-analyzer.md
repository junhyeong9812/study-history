# Elasticsearch 매핑·분석기 — 한글 트레이드마크 검색

> 한글/영문 트레이드마크명을 다양한 방식으로 검색 가능하게 만드는 인덱스 설계.
> Nori 형태소 + 커스텀 코덱스 필터 + 영어 음운 분석기.

---

## 0. 분석 대상 코드

```python
# example-app/src/index/settings.py
def get_index_settings() -> dict:
    return {
        "number_of_shards": 1,
        "number_of_replicas": 0,
        "refresh_interval": "30s",
        "max_result_window": 50000,
        "max_ngram_diff": 40,
        "mapping": {
            "nested_objects": {
                "limit": 100000   # 기본 10000 → 100000
            }
        },
        "analysis": {
            "filter": {
                # 한글 검색 관련 필터 (커스텀 플러그인)
                "mark_kodex":     {"type": "mark_kodex"},
                "mark_kodex_v2":  {"type": "mark_kodex_v2"},
                "mark_jamo":      {"type": "mark_jamo"},
                "origin_kodex":   {"type": "mark_kodex"},

                # 통합 코덱스 필터 (한글+영문)
                "unified_kodex":    {"type": "unified_kodex"},
                "unified_kodex_v2": {"type": "unified_kodex_v2"},

                # 문자 정렬 필터 (애너그램 시그널)
                "char_sort": {"type": "char_sort"},

                # 영어 발음 검색 (Double Metaphone)
                "double_metaphone_filter": {
                    "type": "phonetic",
                    "encoder": "double_metaphone",
                    "replace": False,
                    "max_code_len": 10
                },
                "soundex_filter": {
                    "type": "phonetic",
                    "encoder": "soundex",
                    "replace": False
                },

                # 단어 분할
                "word_delimiter_graph_filter": {
                    "catenate_all": True,
                    "type": "word_delimiter_graph",
                    "preserve_original": True,
                    "stem_english_possessive": False
                },

                # Nori 형태소 + 품사 필터링
                "part_of_speech_stop_sp": {
                    "type": "nori_part_of_speech",
                    "stoptags": ["SP"]
                },
                "nori_part_of_speech": {
                    "type": "nori_part_of_speech",
                    "stoptags": [
                        "E", "IC", "J", "MAG", "MAJ", "MM",
                        "SP", "SSC", "SSO", "SC", "SE", "XPN",
                        "XSA", "XSN", "XSV", "UNA", "NA", "VSV"
                    ]
                },
                "edge_ngram_filter": {
                    "type": "edge_ngram",
                    "min_gram": 1,
                    "max_gram": 20
                }
            },

            "analyzer": {
                "kodex_analyzer": {
                    "filter": ["mark_kodex", "word_delimiter_graph_filter"],
                    "type": "custom",
                    "tokenizer": "whitespace"
                },
                "mark_kodex_v2_analyzer": {
                    "type": "custom",
                    "tokenizer": "whitespace",
                    "filter": ["lowercase", "asciifolding", "mark_kodex_v2"]
                },
                # ... (더 많은 분석기)
            }
        }
    }
```

---

## 1. 인덱스 settings 의 핵심 항목

| 항목 | 의미 |
|------|------|
| `number_of_shards: 1` | 단일 노드 가정. 클러스터로 가면 늘림 |
| `number_of_replicas: 0` | 레플리카 없음 (단일 노드) — 운영 클러스터는 1+ |
| `refresh_interval: 30s` | 색인 후 검색 가능까지 30초. 기본 1s 보다 길어 색인 throughput ↑ |
| `max_result_window: 50000` | from+size 최대 50000 (기본 10000) |
| `max_ngram_diff: 40` | edge_ngram 의 min~max 차이 허용 |
| `mapping.nested_objects.limit: 100000` | 한 문서의 nested object 최대 (기본 10000) |

### refresh_interval 트레이드오프
- **짧으면 (1s)**: 색인 후 즉시 검색 가능. 단 segment 빈번 생성으로 색인 throughput ↓.
- **길면 (30s)**: throughput ↑. 단 색인 후 30초 동안 검색 결과에 안 나타남.
- 대량 색인 시기에는 `-1` (refresh 끔) 으로 두고 끝나면 `_refresh` 명시 호출.

---

## 2. analysis = filter + tokenizer + analyzer

ES 의 텍스트 분석 파이프라인:
```
원본 텍스트 → char_filter → tokenizer → token_filter[] → 인덱싱/검색 토큰
```

- **char_filter**: 글자 수준 변환 (HTML 제거, 매핑 등).
- **tokenizer**: 텍스트를 토큰으로 분할 (whitespace, ngram, nori_tokenizer 등).
- **token_filter**: 토큰 변환/제거 (lowercase, asciifolding, custom).

### `analyzer` 의 정의
```json
"my_analyzer": {
    "type": "custom",
    "tokenizer": "whitespace",
    "filter": ["lowercase", "asciifolding", "mark_kodex_v2"]
}
```

### 매핑에서 사용
```json
"properties": {
    "trademark_name": {
        "type": "text",
        "analyzer": "my_analyzer",
        "search_analyzer": "my_search_analyzer"  // 검색 시 다른 분석기
    }
}
```

`analyzer` 와 `search_analyzer` 분리 → "색인은 넓게, 검색은 좁게" 같은 비대칭 분석 가능.

---

## 3. 한글 분석 — Nori 형태소

### Nori tokenizer
- 한국어 형태소 분석기 (8.x 부터 ES 공식 플러그인).
- "삼성전자" → ["삼성", "전자"] 처럼 분해.

### `nori_part_of_speech` filter
```json
"nori_part_of_speech": {
    "type": "nori_part_of_speech",
    "stoptags": ["E", "J", "MAG", "MAJ", "MM", "SP", ...]
}
```

→ 어미(E), 조사(J), 부사(MAG) 등 검색에 도움 안 되는 품사 토큰 제거.
→ "삼성의 전자" 에서 "의" 제거.

### 품사 코드 (Nori 기준 일부)
| 코드 | 의미 |
|------|------|
| `NNG` | 일반명사 |
| `NNP` | 고유명사 |
| `VV` | 동사 |
| `VA` | 형용사 |
| `J` | 조사 |
| `E` | 어미 |
| `MM` | 관형사 |
| `MAG` | 부사 |
| `SP` | 공백 |

---

## 4. 커스텀 코덱스 필터 — `mark_kodex` / `unified_kodex`

`type: mark_kodex` 등은 표준 ES 가 아니라 **자체 ES 플러그인** 의 필터 타입.

용도:
- 한글 자모 분해 + 정규화.
- "스타벅스" 와 "스타북스" 같은 발음 변형 매칭.
- "ㅅㅌㅂㄱㅅ" 자음만으로 검색 (코덱스).

`unified_kodex_v2` 는 한글+영문 혼용 트레이드마크 (예: "iPhone에어팟") 정규화.

플러그인 자체는 별도 Java 모듈(이 repo 의 `trademark-platform-elasticsearch-hangultokenfilter`).

---

## 5. 영어 음운 분석 — Double Metaphone, Soundex

### Double Metaphone
```json
"double_metaphone_filter": {
    "type": "phonetic",
    "encoder": "double_metaphone",
    "replace": False,
    "max_code_len": 10
}
```

- "Smith", "Smyth" → 둘 다 `SM0` (같은 음운 코드).
- 발음 비슷한 영어 이름 매칭.
- `replace: False` → 원본 토큰도 유지 (정확/발음 동시 검색).

### Soundex
- Double Metaphone 보다 단순. 흔히 영어 검색에 함께 사용.

→ 영어권 트레이드마크 (US/EU) 검색에서 발음 유사도 매칭에 핵심.

---

## 6. Edge n-gram — 부분 일치

```json
"edge_ngram_filter": {
    "type": "edge_ngram",
    "min_gram": 1,
    "max_gram": 20
}
```

→ "Samsung" → ["S", "Sa", "Sam", "Samsu", "Samsung", ...]

검색어 자동완성에 활용 ("Sam" 입력 시 "Samsung" 매칭).

---

## 7. asciifolding — diacritic 제거

```
filter: ["asciifolding"]
```

→ "café" → "cafe", "Müller" → "Muller".

EU/US 데이터에서 악센트 부호 정규화에 필수.

---

## 8. 한 컬럼 → 여러 분석기 (multi-field)

```json
"trademark_name_kor": {
    "type": "text",
    "analyzer": "kodex_analyzer",
    "fields": {
        "raw":    {"type": "keyword"},
        "ngram":  {"type": "text", "analyzer": "edge_ngram_analyzer"},
        "jamo":   {"type": "text", "analyzer": "mark_jamo_analyzer"},
        "kodex_v2": {"type": "text", "analyzer": "mark_kodex_v2_analyzer"}
    }
}
```

검색 시 용도별로 필드 명시:
```python
# 정확 매칭
{"term": {"trademark_name_kor.raw": "삼성"}}

# 코덱스 (자음 검색)
{"match": {"trademark_name_kor.kodex_v2": "ㅅㅅ"}}

# 자모 검색
{"match": {"trademark_name_kor.jamo": "ㅅㅏㅁㅅㅓㅇ"}}

# 자동완성
{"match": {"trademark_name_kor.ngram": "삼"}}
```

→ 한 색인으로 5가지 검색 시나리오 커버.

비용: 인덱스 크기 × N. 디스크 사용량 증가 트레이드오프.

---

## 9. 응용 포인트

- 텍스트 검색 인덱스는 항상 **분석기 설계 우선**. 매핑은 그 다음.
- 색인 throughput 이 중요한 시기는 `refresh_interval=-1` 또는 30s, 끝나면 `_refresh`.
- 한국어는 Nori + 품사 stoptags 가 표준 출발점.
- 발음 유사도 (영문) 는 phonetic 플러그인의 double_metaphone.
- 한 컬럼을 여러 용도로 쓰려면 multi-field. 비용/효용 trade-off.
- 도메인 특수 정규화(상표명 코덱스 등) 가 필요하면 자체 플러그인 개발 가능.
