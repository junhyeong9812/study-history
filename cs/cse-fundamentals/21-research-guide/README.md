# Ch.21 — 연구 가이드 (Research Guide)

> *"좋은 연구는 좋은 질문에서 시작된다."*

> **상태: ✅ 본문 작성 완료**

---

## Part A: Google Scholar 논문 검색법

### 1. Google Scholar란

```
scholar.google.com — 학술 논문 전용 검색 엔진.
일반 Google과 다른 점: 학술 논문, 특허, 학위 논문, 기술 보고서만 검색.

장점: 무료, 방대한 범위, 인용 추적, 관련 논문 추천
한계: 품질 필터링이 약함 (비학술 자료도 섞일 수 있음)
     → Scopus, Web of Science보다 범위는 넓지만 정밀도는 낮을 수 있음
```

### 2. 기본 검색 전략

```
🔴 핵심 원칙: "구체적으로, 단계적으로"

  나쁜 검색: "machine learning"
    → 수백만 결과. 원하는 것을 찾기 어려움.

  좋은 검색: "transformer attention mechanism image classification"
    → 수천 결과. 관련성 높은 것이 상위에.

  더 좋은 검색: "Vision Transformer ViT image classification 2023"
    → 수백 결과. 최신 관련 논문 집중.
```

### 3. 고급 검색 연산자

```
🔴 [외워야 할 연산자들]

  "exact phrase"      — 정확한 구문 검색
    "attention is all you need" → 이 제목의 논문만

  author:이름         — 특정 저자의 논문
    author:"Yoshua Bengio" → Bengio의 논문만

  intitle:키워드      — 제목에 키워드 포함
    intitle:transformer → 제목에 transformer가 있는 논문

  source:저널/학회     — 특정 학술지/학회의 논문
    source:"NeurIPS" → NeurIPS에 게재된 논문

  연도 제한           — 좌측 사이드바에서 연도 범위 설정
    "Since 2023" → 최신 논문만

  OR                  — 두 키워드 중 하나라도 포함
    "object detection" OR "object recognition"

  -키워드             — 특정 키워드 제외
    transformer -electrical -power → 전기 변압기 제외
```

### 4. 인용 추적 — 가장 강력한 기능

```
🔴 [핵심 전략] 인용 관계를 따라가며 문헌을 확장한다.

  (1) "Cited by" (인용됨):
    이 논문을 인용한 후속 논문 목록.
    → "이 논문 이후에 무엇이 발전했는가?"
    → 새로운/최신 연구를 찾는 최고의 방법!

  (2) "Related articles" (관련 논문):
    Scholar가 자동으로 추천하는 유사 논문.
    → 비슷한 주제의 다른 접근법 발견.

  (3) "References" (참고문헌):
    이 논문이 인용한 논문들.
    → "이 연구의 기반이 된 것은 무엇인가?"
    → 분야의 핵심 논문(seminal paper)을 찾는 방법.

  탐색 전략 — 눈덩이 효과:
    핵심 논문 하나를 찾으면:
      ↓ Cited by: 후속 발전
      ↓ References: 기반 연구
      ↓ Related: 유사 연구
    → 이 세 방향으로 확장하면 분야를 빠르게 파악할 수 있다.
```

### 5. 알림과 프로필

```
🔵 이메일 알림: 특정 검색어의 새 논문이 나오면 자동 알림.
   → 최신 연구를 놓치지 않는 최고의 방법.
   → 검색 결과 좌측의 봉투 아이콘 클릭.

🔵 Google Scholar 프로필: 연구자의 논문, 인용 수, h-index를 한눈에.
   → 특정 연구자를 "Follow"하면 새 논문 알림.
```

### 6. 효과적인 검색 워크플로우

```
  Step 1: 키워드 2~3개로 넓게 검색 → 분야 감 잡기
  Step 2: 인용 수가 높은 핵심 논문(survey/review) 찾기
  Step 3: 핵심 논문의 References로 기반 연구 파악
  Step 4: 핵심 논문의 Cited by로 최신 발전 파악
  Step 5: intitle:, author:, 연도 제한으로 좁히기
  Step 6: 관련 키워드를 추가/변경하며 반복
```

---

## Part B: 연구 키워드 선정 전략

### 7. 왜 키워드가 중요한가

```
논문을 찾을 수 있는가 없는가는 키워드에 달려 있다.
같은 개념을 다른 분야에서 다른 용어로 부를 수 있다.

  "이상 탐지" = anomaly detection = outlier detection = novelty detection
  → 하나의 키워드로만 검색하면 관련 연구의 일부만 찾게 된다!
```

### 8. 좋은 키워드 선정법

```
🔴 (1) 동의어와 관련어를 나열한다:
  자동완성: deep learning → deep neural network, representation learning
  시소러스: ACM Computing Classification System (CCS)
  
  (2) 넓은 것 → 좁은 것으로 계층화:
  machine learning → supervised learning → classification → image classification
  → convolutional neural network → ResNet
  
  (3) 핵심 논문의 키워드를 차용:
  좋은 논문의 keywords 섹션 → 해당 분야에서 통용되는 정확한 용어
  
  (4) 방법론 + 응용 도메인을 조합:
  "graph neural network" + "drug discovery"
  "transformer" + "time series forecasting"
```

---

## Part C: 탑티어 학회와 저널 가이드

### 9. CS에서 학회가 왜 중요한가

```
🔴 CS의 독특한 문화:
  대부분의 학문: 저널(journal)이 최고 권위
  CS: 학회(conference)가 최고 권위!

  왜:
    CS의 발전 속도가 빨라서, 저널의 긴 리뷰 과정(6~18개월)은 너무 느리다.
    학회는 6~9개월 주기로 최신 연구를 발표·공유.
    동료 연구자와 직접 토론하는 기회가 학회에만 있다.

  저널은 주로 학회 논문의 확장판(extended version)에 사용된다.
```

### 10. 분야별 탑티어 학회 목록

```
🔴 [외워야 할 핵심 학회들]

┌─────────────────┬──────────────────────────────────────────────┐
│ 분야              │ 탑티어 학회 (CORE A*/A 등급)                   │
├─────────────────┼──────────────────────────────────────────────┤
│ AI 전반           │ NeurIPS, ICML, ICLR, AAAI, IJCAI            │
│ 컴퓨터 비전       │ CVPR, ICCV, ECCV                             │
│ 자연어 처리       │ ACL, EMNLP, NAACL                            │
│ 데이터 마이닝     │ KDD, WSDM, WWW, ICDM                        │
│ 데이터베이스      │ SIGMOD, VLDB, ICDE                           │
│ 시스템           │ SOSP, OSDI, EuroSys                          │
│ 네트워크         │ SIGCOMM, NSDI, INFOCOM, MobiCom              │
│ 보안             │ IEEE S&P, CCS, USENIX Security, NDSS         │
│ 프로그래밍 언어   │ POPL, PLDI, OOPSLA, ICFP                     │
│ 알고리즘/이론     │ STOC, FOCS, SODA                             │
│ 컴퓨터 구조      │ ISCA, MICRO, ASPLOS, HPCA                    │
│ 소프트웨어 공학   │ ICSE, FSE, ASE                               │
│ HCI             │ CHI, UIST, CSCW                              │
│ 로보틱스         │ ICRA, IROS, RSS, CoRL                        │
│ 컴퓨터 그래픽    │ SIGGRAPH, SIGGRAPH Asia, Eurographics         │
└─────────────────┴──────────────────────────────────────────────┘

참고:
  CORE Ranking (core.edu.au): A* > A > B > C 등급
  CCF (중국 컴퓨터 학회): A, B, C 등급
  CSRankings.org: 교수/대학 기준 학회 분류
```

### 11. 학회 랭킹을 보는 법

```
🔴 학회의 권위를 판단하는 기준:

  (1) 수락률 (Acceptance Rate):
    탑티어: 15~25% (4~7편 중 1편만 수락)
    NeurIPS 2023: ~26%, CVPR 2023: ~25.8%
    
  (2) h-index: 학회의 전체 인용 영향력
  
  (3) CORE 등급: A* (최상), A (우수), B (양호)
  
  (4) 구글 학술 메트릭: Scholar에서 학술지/학회별 h5-index 확인 가능

  주의: 숫자만으로 판단하지 마라.
    "이 학회에 어떤 연구자들이 발표하는가"가 가장 중요한 신호.
    분야의 리더들이 발표하는 학회 = 그 분야의 탑.
```

---

## Part D: 논문 읽기와 작성 기초

### 12. 논문의 구조 — 왜 이 순서인가

```
🔴 표준 구조 (IMRaD + α):

  Abstract:     논문 전체를 300단어 내외로 요약
  Introduction: 문제 정의, 동기, 기여(contribution)
  Related Work: 관련 연구 정리. "기존 연구와 우리 연구의 차이"
  Method:       제안 방법의 상세 설명
  Experiments:  실험 설정, 결과, 비교, 분석
  Discussion:   한계, 향후 연구 방향
  Conclusion:   핵심 기여 재요약
  References:   인용 목록

왜 이 순서인가:
  Abstract: "읽을 가치가 있는가?" 판단 → 가장 먼저
  Introduction: "무슨 문제를 왜 풀었는가?" → 동기 부여
  Related Work: "남들은 어떻게 했는가?" → 맥락 설정
  Method: "우리는 어떻게 했는가?" → 핵심 기여
  Experiments: "정말 작동하는가?" → 증거 제시
  → 독자의 질문 순서대로 배열된 것!
```

### 13. 논문 읽는 법 — "세 번 읽기" 전략

```
🔴 Keshav의 "Three-Pass Approach":

  1차 (5~10분): 큰 그림 파악
    Title, Abstract, Introduction의 마지막 문단(기여), 
    Section 제목들, Conclusion만 읽기.
    → "이 논문이 무엇에 관한 것인가?" 파악.
    → 더 읽을 가치가 있는가 판단.

  2차 (1시간): 내용 이해
    그림, 표, 다이어그램을 주의 깊게 봄.
    핵심 수식을 이해하려고 노력 (증명은 건너뜀).
    모르는 용어/인용을 표시.
    → 논문의 주장을 남에게 요약할 수 있는 수준.

  3차 (4~5시간, 필요시만): 완전 이해
    논문을 "다시 구현할 수 있을 정도로" 이해.
    모든 가정, 증명, 실험을 검증.
    → 리뷰어 수준의 이해.

  대부분의 논문은 1차로 충분하다.
  분야의 핵심 논문만 2~3차까지 읽으면 된다.
```

### 14. 레퍼런스 관리 도구

```
🔵 왜 관리 도구가 필요한가:
  수십~수백 편의 논문을 읽으면, 어떤 논문에서 뭘 봤는지 기억할 수 없다.
  인용 형식(APA, IEEE, ACM)을 매번 수동으로 맞추는 것은 고문이다.

  도구들:
    Zotero: 무료, 오픈소스. 브라우저 확장으로 원클릭 저장. 추천!
    Mendeley: 무료, PDF 리더 내장.
    Paperpile: Google Docs 연동이 좋음. 유료.
    BibTeX: LaTeX 사용 시 필수. 텍스트 기반 참고문헌 관리.
    
    기본 워크플로우:
    (1) 논문을 찾으면 → 도구에 저장 (메타데이터 자동 추출)
    (2) 태그/폴더로 분류
    (3) 메모/하이라이트 추가
    (4) 논문 작성 시 → 도구에서 인용 삽입 → 형식 자동 생성
```

---

## Part E: 연구 시작하기 — 실전 조언

### 15. 연구 주제를 어떻게 찾는가

```
  (1) Survey 논문에서 "Open Problems" / "Future Work" 섹션 읽기
      → 아직 풀리지 않은 문제가 명시되어 있다!
  
  (2) 최신 탑티어 학회 논문의 Limitation 섹션 읽기
      → "이 연구의 한계"가 곧 다음 연구의 기회.
  
  (3) 두 분야의 교차점
      "NLP + 의료", "그래프 + 추천", "강화학습 + 로봇"
      → 한 분야의 기법을 다른 분야에 적용 = 새 연구.
  
  (4) 기존 방법을 새로운 데이터/도메인에 적용
      → "이 방법이 한국어에서도 작동하는가?"
      → "이 방법이 위성 이미지에서도 작동하는가?"
  
  (5) 지도교수/선배와 대화
      → 가장 현실적이고 효과적인 방법!
```

### 16. 재현성 — 왜 코드를 공개하는가

```
🔵 현대 CS 연구의 트렌드: 코드 + 데이터 공개

  왜: 논문만으로는 재현이 어렵다. 세부 구현이 결과에 큰 영향.
  GitHub에 코드를 공개하면:
    (1) 다른 연구자가 검증할 수 있다 (신뢰성)
    (2) 후속 연구에서 기반으로 사용 가능 (영향력)
    (3) 인용이 늘어난다 (실용적 이점)
  
  Papers With Code (paperswithcode.com):
    논문 + 코드 + 벤치마크를 연결해주는 사이트.
    → "이 논문의 공식 구현은 어디에?"를 찾을 때 최고.
```

---

## 17. 정리 — 연구를 시작하는 체크리스트

```
  □ Google Scholar에서 분야의 핵심 키워드로 검색
  □ 인용 수 높은 survey/review 논문 1~2편 찾기 (분야 조감도)
  □ 해당 분야의 탑티어 학회 확인 (CORE A*/A)
  □ 핵심 논문 5~10편을 "세 번 읽기" 전략으로 읽기
  □ Cited by / References로 문헌 확장
  □ Zotero 등으로 레퍼런스 관리 시작
  □ 읽은 논문마다 1문단 요약 메모 작성
  □ Open Problems / Limitations에서 연구 주제 후보 도출
  □ 코드 구현 + 재현 실험으로 이해 심화
```

---

## 18. 더 깊이 공부하기 위한 자료

- **"How to Read a Paper"** (S. Keshav) — 3-pass 전략의 원전 (3페이지 논문)
- **"Writing a Great Research Paper"** (Simon Peyton Jones, YouTube) — 논문 작성 명강의
- **CSRankings.org** — 학교/교수별 탑 학회 게재 현황
- **Semantic Scholar** (semanticscholar.org) — AI 기반 논문 검색 (Google Scholar 대안)
- **Connected Papers** (connectedpapers.com) — 논문 관계를 그래프로 시각화

---

*이 챕터로 CSE Fundamentals 전체 교과서가 완성되었습니다.*
*처음부터 읽어도 좋고, 필요한 챕터만 골라 읽어도 좋습니다.*
*[메인 README로 돌아가기](../README.md)*
