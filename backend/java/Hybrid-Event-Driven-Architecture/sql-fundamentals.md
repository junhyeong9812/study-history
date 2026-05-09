# SQL 기본기 — INTERVAL, JOIN, UNION 외 자주 쓰는 패턴들

> 이 프로젝트의 `OutboxCleaner`, `OutboxMigrationTest` 등에 등장하는 SQL 표현을 풀어 정리한 학습 노트.
> ANSI 표준과 PostgreSQL 고유 기능을 함께 다룬다.

---

## 0. 출발점 — `INTERVAL` 표현

```sql
DELETE FROM outbox
WHERE status = 'PUBLISHED'
  AND published_at < NOW() - INTERVAL '7 days'
```

이 한 줄에 두 가지 SQL 도구가 있다:

| 도구 | 역할 |
|------|------|
| `NOW()` | 현재 시각 (`CURRENT_TIMESTAMP`와 거의 같음) |
| `INTERVAL '7 days'` | 7일이라는 **시간 간격(duration)** 값 |

`NOW() - INTERVAL '7 days'`는 **"지금으로부터 7일 전"** 의 timestamp를 계산. 거기보다 이전(`<`)인 행만 DELETE.

---

## 1. `INTERVAL` — 시간 간격 표현

### 1.1 문법

```sql
INTERVAL '<숫자> <단위>'
```

```sql
INTERVAL '7 days'
INTERVAL '1 hour'
INTERVAL '30 minutes'
INTERVAL '2 weeks'
INTERVAL '1 year 6 months'      -- 복합
INTERVAL '1 day 3 hours 15 minutes'
INTERVAL '-5 days'              -- 음수 가능
```

지원 단위: `microseconds`, `milliseconds`, `second`, `minute`, `hour`, `day`, `week`, `month`, `year`, `decade`, `century`, `millennium`.

### 1.2 시각 산술

```sql
NOW() + INTERVAL '1 hour'                 -- 한 시간 후
NOW() - INTERVAL '7 days'                  -- 일주일 전
'2026-01-01'::date + INTERVAL '1 month'    -- 2026-02-01
created_at + INTERVAL '24 hours'           -- 컬럼에서도 동작
```

### 1.3 사이 시각 비교

```sql
WHERE published_at < NOW() - INTERVAL '7 days'   -- 7일 이전
WHERE created_at > NOW() - INTERVAL '1 hour'     -- 최근 1시간 이내
WHERE event_time BETWEEN NOW() - INTERVAL '1 day'
                     AND NOW()                    -- 어제부터 지금까지
```

### 1.4 PostgreSQL 고유 — 데이터베이스 마다 다름

`INTERVAL` 자체는 ANSI 표준이지만 문법이 DB별로 미묘하게 다르다.

| DB | 7일 전 표현 |
|----|-----------|
| PostgreSQL | `NOW() - INTERVAL '7 days'` |
| MySQL | `NOW() - INTERVAL 7 DAY` (따옴표/단수형) |
| Oracle | `SYSDATE - 7` (숫자만 가능) 또는 `INTERVAL '7' DAY` |
| SQL Server | `DATEADD(day, -7, GETDATE())` |
| H2 | `DATEADD('DAY', -7, NOW())` |

→ 이 프로젝트는 PostgreSQL 전용 문법을 그대로 쓴다 (Testcontainers로 진짜 PostgreSQL을 검증). H2 같은 임베디드 DB로 테스트했다면 이 SQL이 다르게 해석돼 운영 사고가 났을 것.

---

## 2. 시각 함수들

### 2.1 현재 시각

```sql
NOW()                   -- timestamp with time zone
CURRENT_TIMESTAMP       -- NOW()와 동일
CURRENT_DATE            -- 오늘 날짜만
CURRENT_TIME            -- 현재 시각만
LOCALTIMESTAMP          -- timestamp without time zone
```

`NOW()`는 **트랜잭션 시작 시각**을 반환 (한 트랜잭션 안에선 항상 같은 값). 진짜 현재 시각이 필요하면 `clock_timestamp()`.

### 2.2 시각 추출

```sql
EXTRACT(YEAR FROM created_at)        -- 2026
EXTRACT(EPOCH FROM created_at)       -- Unix timestamp (초)
DATE_TRUNC('day', created_at)        -- 일 단위로 절단 → 자정 시각
DATE_TRUNC('hour', created_at)       -- 시간 단위
TO_CHAR(created_at, 'YYYY-MM-DD')    -- 문자열 포맷
```

### 2.3 시각 차이 계산

```sql
NOW() - created_at                   -- INTERVAL 반환
EXTRACT(EPOCH FROM (NOW() - created_at))  -- 초 단위 차이
AGE(NOW(), created_at)               -- 사람이 읽기 좋은 형태
```

---

## 3. JOIN 종류

### 3.1 데이터 준비 (예시)

```sql
orders                payments
─────────────         ─────────────
id  customer_id       id  order_id  status
1   100               10  1         COMPLETED
2   100               11  3         FAILED
3   200
4   300                              -- order 4의 payment 없음
                                     -- payment 11의 order는 있지만 다른 상태
```

### 3.2 INNER JOIN — 양쪽 다 매칭되는 행만

```sql
SELECT o.id, p.status
FROM orders o
INNER JOIN payments p ON o.id = p.order_id;

-- 결과:
-- 1, COMPLETED
-- 3, FAILED
```

`order_id`가 매칭되는 쌍만. order 2, 4는 payment가 없으니 빠진다.

### 3.3 LEFT JOIN — 왼쪽 전부 + 오른쪽 매칭

```sql
SELECT o.id, p.status
FROM orders o
LEFT JOIN payments p ON o.id = p.order_id;

-- 결과:
-- 1, COMPLETED
-- 2, NULL          (payment 없음)
-- 3, FAILED
-- 4, NULL          (payment 없음)
```

**왼쪽 테이블의 모든 행**이 결과에 포함. 매칭되는 오른쪽 행이 없으면 NULL.

### 3.4 RIGHT JOIN — 오른쪽 전부

```sql
SELECT o.id, p.status
FROM orders o
RIGHT JOIN payments p ON o.id = p.order_id;
```

LEFT의 거울. 보통은 LEFT만 쓰고 RIGHT는 잘 안 씀 (테이블 순서를 바꾸면 LEFT로 표현 가능).

### 3.5 FULL OUTER JOIN — 양쪽 전부

```sql
SELECT o.id, p.status
FROM orders o
FULL OUTER JOIN payments p ON o.id = p.order_id;

-- 결과:
-- 1, COMPLETED
-- 2, NULL
-- 3, FAILED
-- 4, NULL
-- (payment 11이 매칭 안 되는 경우 이 행도 추가될 것)
```

매칭 여부 무관하게 양쪽 모든 행. 사용 빈도는 낮지만 데이터 차이 분석에 유용.

### 3.6 CROSS JOIN — 카테시안 곱

```sql
SELECT o.id, p.id FROM orders o CROSS JOIN payments p;
-- orders 4행 × payments 2행 = 8행
```

조건 없이 모든 조합. 보통은 의도된 게 아니면 실수.

### 3.7 SELF JOIN — 같은 테이블끼리

```sql
SELECT a.id AS order_id, b.id AS sibling_id
FROM orders a
JOIN orders b ON a.customer_id = b.customer_id AND a.id <> b.id;
-- 같은 고객의 다른 주문
```

테이블 별칭(`a`, `b`)으로 자기 자신을 두 번 쓴 것.

### 3.8 JOIN 종류 요약

```
        A ∩ B           A ∪ B (교집합 외엔 NULL로)        A only
        INNER           LEFT                              LEFT + WHERE B IS NULL
         ┌─┐                ┌────┐                       ┌────┐
         │█│              ┌─┤████│                     ┌─┤████│
         └─┘              │█│████│                     │█│    │
                          └─┴────┘                     └─┴────┘

        FULL                                            B only
        ┌──────┐                                        ┌──┴──┐
      ┌─┤██████│                                          │████│
      │█│██████│                                          │████│
      └─┴──────┘                                          └────┘
```

---

## 4. UNION / INTERSECT / EXCEPT — 결과 집합 연산

### 4.1 UNION — 합집합 (중복 제거)

```sql
SELECT id, customer_id FROM orders WHERE status = 'CONFIRMED'
UNION
SELECT id, customer_id FROM orders WHERE customer_id IN (100, 200);
-- 두 결과의 합. 같은 행은 한 번만.
```

### 4.2 UNION ALL — 합집합 (중복 유지)

```sql
SELECT id FROM orders
UNION ALL
SELECT id FROM payments;
-- 더 빠름 (중복 검사 안 함). 보통 UNION보다 ALL이 의도에 맞음.
```

`UNION`이 중복 제거하느라 정렬·해시 작업을 함. **중복이 없다는 걸 알면 항상 `UNION ALL`** 이 정답.

### 4.3 INTERSECT — 교집합

```sql
SELECT customer_id FROM orders
INTERSECT
SELECT customer_id FROM premium_customers;
-- 두 쿼리 결과 모두에 등장하는 customer_id만
```

### 4.4 EXCEPT (Oracle은 MINUS)

```sql
SELECT customer_id FROM orders
EXCEPT
SELECT customer_id FROM blocked_customers;
-- 첫 결과에서 두 번째 결과를 뺀 차집합
```

### 4.5 JOIN vs UNION

자주 헷갈리는 둘:

| | JOIN | UNION |
|--|------|-------|
| 결합 방식 | **수평** (열을 늘림) | **수직** (행을 합침) |
| 컬럼 수 | 양쪽 합 | 같아야 함 |
| 컬럼 타입 | 무관 | 같아야 함 (호환 가능) |
| 용도 | 관련된 데이터를 함께 표현 | 같은 종류의 데이터를 합침 |

---

## 5. 서브쿼리와 CTE

### 5.1 서브쿼리 (Subquery)

```sql
-- 평균보다 큰 주문 금액
SELECT * FROM orders
WHERE amount > (SELECT AVG(amount) FROM orders);

-- 결제 완료된 주문만
SELECT * FROM orders
WHERE id IN (SELECT order_id FROM payments WHERE status = 'COMPLETED');
```

### 5.2 CTE (Common Table Expression, `WITH`)

복잡한 쿼리를 단계로 쪼개 가독성 ↑.

```sql
WITH pending_outbox AS (
    SELECT * FROM outbox WHERE status = 'PENDING'
),
old_pending AS (
    SELECT * FROM pending_outbox
    WHERE created_at < NOW() - INTERVAL '1 hour'
)
SELECT count(*) FROM old_pending;
```

서브쿼리를 중첩하면 가독성 폭망. CTE는 **이름 있는 단계**라 추론 가능.

### 5.3 재귀 CTE — 트리/계층 순회

```sql
WITH RECURSIVE org_tree AS (
    SELECT id, name, manager_id FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.name, e.manager_id
    FROM employees e
    JOIN org_tree t ON e.manager_id = t.id
)
SELECT * FROM org_tree;
```

조직도, 카테고리 트리, 그래프 순회에 강력.

---

## 6. WINDOW 함수 — 그룹 안에서 순위/누적

### 6.1 기본

```sql
SELECT
    id,
    customer_id,
    amount,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
FROM orders;
-- 각 고객의 주문을 시간 역순으로 번호 매김
```

### 6.2 자주 쓰는 함수

| 함수 | 의미 |
|------|------|
| `ROW_NUMBER()` | 1, 2, 3, ... (동률 무시) |
| `RANK()` | 1, 2, 2, 4, ... (동률 처리) |
| `DENSE_RANK()` | 1, 2, 2, 3, ... (동률 후 연속) |
| `LAG(col)` | 이전 행의 값 |
| `LEAD(col)` | 다음 행의 값 |
| `SUM(col) OVER (ORDER BY ...)` | 누적 합 |
| `AVG(col) OVER (...)` | 이동 평균 |

### 6.3 활용 예

```sql
-- 각 고객의 최신 주문만 가져오기
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
    FROM orders
) t WHERE rn = 1;
```

---

## 7. PostgreSQL 고유 기능 (이 프로젝트 사용)

### 7.1 JSONB

```sql
-- 컬럼 타입
payload JSONB NOT NULL

-- 값 추출
SELECT payload->>'orderId' FROM outbox;        -- 텍스트로
SELECT payload->'orderId' FROM outbox;         -- jsonb로
SELECT payload#>>'{user,name}' FROM outbox;    -- 깊은 경로

-- 조건
WHERE payload @> '{"orderId": 42}'             -- 부분 일치
WHERE payload ? 'urgentFlag'                    -- 키 존재
```

### 7.2 부분 인덱스 (Partial Index)

```sql
-- 이 프로젝트가 쓰는 outbox PENDING 인덱스
CREATE INDEX idx_outbox_pending
    ON outbox(created_at)
    WHERE status = 'PENDING';
```

PENDING 행만 인덱스에 들어 있으니, 발행 완료된 수백만 행이 있어도 PENDING이 적으면 인덱스가 작음 → 빠른 카운트·조회.

### 7.3 `FOR UPDATE SKIP LOCKED` (Phase 3)

```sql
SELECT * FROM outbox
WHERE status = 'PENDING'
ORDER BY created_at
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

여러 Relay가 동시에 같은 outbox를 폴링해도 **이미 잠긴 행은 건너뛰고** 다음 행을 잡는다. 멀티 인스턴스 작업 큐의 표준 패턴.

### 7.4 `RETURNING`

```sql
-- INSERT/UPDATE/DELETE의 결과를 즉시 받기
INSERT INTO orders (...) VALUES (...) RETURNING id;
UPDATE outbox SET status = 'PUBLISHED' WHERE id = 42 RETURNING *;
DELETE FROM outbox WHERE published_at < NOW() - INTERVAL '7 days' RETURNING id;
```

JPA의 `INSERT 후 ID 받기`도 내부적으로 이걸 사용.

### 7.5 Upsert — `ON CONFLICT`

```sql
INSERT INTO inbox (message_id, event_type, payload) VALUES (?, ?, ?)
ON CONFLICT (message_id) DO NOTHING;
-- 중복이면 그냥 무시 (멱등성)

INSERT INTO settings (key, value) VALUES ('foo', 'bar')
ON CONFLICT (key) DO UPDATE SET value = EXCLUDED.value;
-- 있으면 업데이트
```

이 프로젝트의 InboxConsumer 멱등성에 적용 가능 (현재는 애플리케이션 레벨 `existsByMessageId` 체크로 처리).

---

## 8. 자주 쓰는 패턴 모음

### 8.1 그룹별 카운트

```sql
SELECT status, COUNT(*) FROM outbox GROUP BY status;
```

### 8.2 그룹별 최댓값을 가진 행

```sql
-- 각 고객의 가장 최근 주문
SELECT DISTINCT ON (customer_id) *
FROM orders
ORDER BY customer_id, created_at DESC;
-- (PostgreSQL 고유 — 다른 DB는 WINDOW 함수로)
```

### 8.3 EXISTS / NOT EXISTS

```sql
-- 결제가 있는 주문
SELECT * FROM orders o
WHERE EXISTS (SELECT 1 FROM payments p WHERE p.order_id = o.id);

-- 결제가 없는 주문
SELECT * FROM orders o
WHERE NOT EXISTS (SELECT 1 FROM payments p WHERE p.order_id = o.id);
```

`IN`보다 보통 빠름 (서브쿼리 결과를 다 만들지 않고 첫 매칭만 확인).

### 8.4 CASE WHEN

```sql
SELECT
    id,
    CASE
        WHEN amount > 10000 THEN 'HIGH'
        WHEN amount > 1000 THEN 'MID'
        ELSE 'LOW'
    END AS tier
FROM orders;
```

### 8.5 COALESCE — NULL 기본값

```sql
SELECT COALESCE(deleted_at, '9999-12-31') FROM users;
-- deleted_at이 NULL이면 미래 시각으로 대체
```

---

## 9. 자주 만나는 함정

### 9.1 NULL 비교는 `=` 아닌 `IS`

```sql
WHERE status = NULL          -- ❌ 항상 false
WHERE status IS NULL          -- ✅
WHERE status IS NOT NULL      -- ✅
```

### 9.2 `IN` 안에 NULL이 섞이면 NOT IN이 모두 false

```sql
WHERE id NOT IN (1, 2, NULL)  -- 항상 false (3-valued logic)
WHERE id NOT IN (1, 2) AND id IS NOT NULL  -- 안전
```

### 9.3 `COUNT(*)` vs `COUNT(column)` vs `COUNT(DISTINCT column)`

```sql
COUNT(*)              -- 모든 행 (NULL 포함)
COUNT(column)         -- column이 NULL이 아닌 행
COUNT(DISTINCT col)   -- 고유 값 수
```

### 9.4 GROUP BY는 SELECT의 모든 비집계 컬럼을 포함해야 함

```sql
-- ❌ 표준 위반 (MySQL은 허용하지만 위험)
SELECT customer_id, name, COUNT(*) FROM orders GROUP BY customer_id;

-- ✅
SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id;
SELECT customer_id, name, COUNT(*) FROM orders GROUP BY customer_id, name;
```

### 9.5 시각의 시간대 (timezone)

```sql
-- timestamp without time zone (TZ 정보 없음)
created_at TIMESTAMP DEFAULT NOW()

-- timestamp with time zone (UTC로 저장 + TZ 변환)
created_at TIMESTAMPTZ DEFAULT NOW()
```

운영에선 `TIMESTAMPTZ` 권장. 이 프로젝트는 단순화 위해 `TIMESTAMP` 사용.

---

## 10. 이 프로젝트에서의 SQL 예시 모음

### 10.1 OutboxCleaner — INTERVAL 기반 일괄 삭제

```sql
DELETE FROM outbox
WHERE status = 'PUBLISHED'
  AND published_at < NOW() - INTERVAL '7 days';
```

### 10.2 OutboxRepository — 부분 인덱스 활용 카운트

```sql
SELECT COUNT(*) FROM outbox WHERE status = 'PENDING';
-- 부분 인덱스 idx_outbox_pending 덕분에 빠름
```

### 10.3 OutboxMigrationTest — information_schema 메타 조회

```sql
SELECT column_name FROM information_schema.columns
WHERE table_name = 'outbox';

SELECT COUNT(*) FROM pg_indexes
WHERE tablename = 'outbox' AND indexname = 'idx_outbox_pending';
```

### 10.4 Phase 3 멀티 Relay — SKIP LOCKED

```sql
SELECT * FROM outbox
WHERE status = 'PENDING'
ORDER BY created_at
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

---

## 11. 한 줄 요약

> **`INTERVAL`은 시간 간격, `JOIN`은 수평 결합, `UNION`은 수직 결합.**
> 이 셋과 `EXISTS`, `WINDOW`, `WITH`만 익숙해지면 데이터 작업의 90%는 끝난다.
> PostgreSQL 고유 기능(JSONB, 부분 인덱스, SKIP LOCKED, ON CONFLICT)은 운영 시스템의 강력한 무기 — 이 프로젝트가 일관되게 활용한다.

---

## 12. 함께 보면 좋은 자료

- 공식: [PostgreSQL Documentation](https://www.postgresql.org/docs/current/index.html)
- `docs/study/jdbc-template.md` — Spring에서 위 SQL을 호출하는 도구
- `docs/study/flyway-migrations.md` — DDL을 안전하게 진화시키기
- `docs/phase-2-tdd.md` Step 1, 4 — outbox/inbox 스키마 정의
- `docs/phase-3-tdd.md` Step 3, 7 — OutboxCleaner의 INTERVAL, SKIP LOCKED

---

## 13. 학습 순서 추천

```
1. SELECT + WHERE + ORDER BY + LIMIT (기초 조회)
   ↓
2. JOIN (INNER, LEFT 위주)
   ↓
3. GROUP BY + 집계 함수 (COUNT, SUM, AVG)
   ↓
4. UNION ALL, IN, EXISTS
   ↓
5. INTERVAL과 시각 함수
   ↓
6. CTE (WITH) — 가독성 도약
   ↓
7. WINDOW 함수 — "그룹 안에서 순위·누적"
   ↓
8. DB 고유 기능 (JSONB, 부분 인덱스, SKIP LOCKED 등)
```

각 단계가 다음 단계의 가독성을 만든다. 1~5만 알아도 일상 80%, 6~7까지 가면 거의 모든 분석 쿼리 가능.
