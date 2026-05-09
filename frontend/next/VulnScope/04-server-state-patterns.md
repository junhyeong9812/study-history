# 04. Server State 패턴

> 이 문서가 다루는 것: server state ≠ client state, stale-while-revalidate, queryKey 위계, observer 구독 — TanStack Query 의 본질적 통찰.

---

## §0. server state 가 client state 와 다른 이유

### 0.1 client state
```typescript
const [isOpen, setIsOpen] = useState(false);    // 모달 open
const [filter, setFilter] = useState("all");     // UI filter
```

**특징**:
- 컴포넌트 안에서만 의미.
- 서버 모름.
- truth = client.

### 0.2 server state
```typescript
const scans = await fetch("/api/scans").then(r => r.json());
```

**특징**:
- truth 가 **다른 곳 (서버)** 에 있음.
- **stale 가능** — 우리가 fetch 한 후 서버에서 변경됐을 수 있음.
- **shared** — 다른 사용자도 변경 가능.
- async — 로딩/에러 상태 필요.
- expensive — 매번 fetch 하면 cost.

### 0.3 server state 의 어려운 질문들
- **언제 refetch?** — 사용자 focus 돌아왔을 때? 매 페이지 navigate? 5초마다?
- **여러 컴포넌트가 같은 데이터 — 중복 fetch?**
- **mutation 후 cache 무효화 — 어디까지?**
- **optimistic update — 실패 시 rollback?**

**naive (useState + useEffect)**:
```typescript
const [scans, setScans] = useState<Scan[]>([]);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);

useEffect(() => {
    fetch("/api/scans")
        .then(r => r.json())
        .then(d => setScans(d))
        .catch(setError)
        .finally(() => setLoading(false));
}, []);
```

위 모든 질문 매번 직접 처리. boilerplate 폭증.

**TanStack Query (React Query)**:
```typescript
const { data: scans, isLoading, isError } = useQuery({
    queryKey: ["scans"],
    queryFn: () => fetch("/api/scans").then(r => r.json()),
});
```

→ 위 모든 처리 자동.

이게 **server state 라이브러리가 등장한 이유** 입니다.

---

## §1. Stale-While-Revalidate (SWR) 패턴

### 1.1 핵심 아이디어
```
1. cache 에 데이터 있음 → 즉시 표시 (stale 일 수 있음)
2. background 에서 fetch
3. 응답 도착 → cache 업데이트 → UI re-render
```

→ 사용자는 **즉시 UI 보고**, 백그라운드에서 신선화.

### 1.2 비교
| 패턴 | 동작 |
|---|---|
| **fetch every time** | 매번 loading → 응답 → 표시. 매번 spinner. |
| **cache first, no refresh** | 한번 fetch 후 stale 그대로. 데이터 안 신선. |
| **SWR** | cache 즉시 표시 + 백그라운드 refresh. **둘 다 충족**. |

### 1.3 staleTime 의 의미
```typescript
new QueryClient({
    defaultOptions: {
        queries: {
            staleTime: 30_000,    // 30초 동안 fresh
        },
    },
});
```

- `staleTime` 동안: cache 가 fresh. refetch 안 함.
- 그 후: stale. 다음 mount/focus 시 자동 refetch.

VulnScope 의 30초:
- 같은 페이지 빠른 navigate (`/scans → /findings/X → back`) 시 refetch 안 함.
- 30초 후 다시 mount 면 refetch.

### 1.4 cacheTime (gcTime in v5)
- query 가 unmount 된 후 cache 보관 시간.
- 그 시간 안에 다시 mount 되면 cache 즉시 사용.
- 그 시간 후 GC.

### 1.5 직접 구현한다면
```typescript
type Cache = Map<string, { data: any; timestamp: number }>;
const cache: Cache = new Map();

function useFetch<T>(key: string, fn: () => Promise<T>, staleMs = 30_000) {
    const cached = cache.get(key);
    const isStale = !cached || (Date.now() - cached.timestamp) > staleMs;

    const [data, setData] = useState<T | undefined>(cached?.data);

    useEffect(() => {
        if (isStale) {
            fn().then(d => {
                cache.set(key, { data: d, timestamp: Date.now() });
                setData(d);
            });
        }
    }, [key]);

    return data;
}
```

원리만. 실제 라이브러리는 더 복잡 (subscriber tracking, focus refetch, retry, error handling).

---

## §2. queryKey 위계 — cache 식별 + invalidation

### 2.1 queryKey 가 무엇인가
```typescript
useQuery({
    queryKey: ["scan", scanId],
    queryFn: () => getScan(scanId),
});
```

`["scan", scanId]` = cache key. 같은 key 면 같은 데이터 공유 (cache hit).

다른 컴포넌트가 같은 key 사용 시 데이터 공유. 한 번 fetch, 모두 활용.

### 2.2 위계적 key
```typescript
useQuery({ queryKey: ["scan", scanId] });               // ① scan 본문
useQuery({ queryKey: ["scan", scanId, "summary"] });    // ② summary
useQuery({ queryKey: ["scan", scanId, "findings"] });   // ③ findings
```

`["scan", scanId, ...]` 가 prefix. 모두 "이 scan 의 sub-resource".

### 2.3 invalidation 의 강력함
```typescript
// scan trigger 후 모든 scan-related cache invalidate
queryClient.invalidateQueries({ queryKey: ["scan"] });
```

→ `["scan", X]`, `["scan", X, "findings"]`, `["scan", Y]`, ... **모두 invalidate**.

prefix matching. 위계가 자동 의미.

### 2.4 정밀 invalidate
```typescript
// scan X 의 findings 만
queryClient.invalidateQueries({ queryKey: ["scan", scanId, "findings"] });

// 모든 scan
queryClient.invalidateQueries({ queryKey: ["scan"] });

// 모든 query
queryClient.invalidateQueries();
```

### 2.5 VulnScope 의 위계
```typescript
useScanResult(scanId) {
    useQuery({ queryKey: ["scan", scanId] });
    useQuery({ queryKey: ["scan", scanId, "summary"] });
    useQuery({ queryKey: ["scan", scanId, "findings"] });
}

useFindingDetail(findingId) {
    useQuery({ queryKey: ["finding", findingId] });
}

useScans({ page, size }) {
    useQuery({ queryKey: ["scans", { page, size }] });    // opts 도 key 일부
}

useTargets() {
    useQuery({ queryKey: ["targets"] });
}

useProfiles() {
    useQuery({ queryKey: ["scan-profiles"] });
}
```

### 2.6 핵심 통찰
**queryKey 가 hierarchy 표현 + invalidation 단위**. 잘 설계하면 mutation 후 정확한 cache invalidate.

### 2.7 직접 구현한다면
prefix matching:
```typescript
function invalidate(prefix: any[]) {
    for (const [key, _] of cache.entries()) {
        const keyArray = JSON.parse(key);
        if (matchPrefix(keyArray, prefix)) {
            // refetch 또는 stale 마크
        }
    }
}

function matchPrefix(a: any[], prefix: any[]): boolean {
    if (a.length < prefix.length) return false;
    return prefix.every((p, i) => deepEqual(a[i], p));
}
```

---

## §3. Observer 구독 (multiple subscribers, single fetch)

### 3.1 같은 query 여러 컴포넌트
```typescript
function ComponentA() {
    const { data } = useQuery({ queryKey: ["scans"], queryFn: ... });
    // ...
}

function ComponentB() {
    const { data } = useQuery({ queryKey: ["scans"], queryFn: ... });
    // ...
}
```

**naive**: 두 번 fetch.

**TanStack Query**: 같은 key → 같은 query observer → **1번 fetch, 두 컴포넌트 모두 같은 data**.

### 3.2 동작 원리
1. ComponentA mount → query observer 등록 → fetch 시작.
2. ComponentB mount → 같은 key → 기존 observer 에 subscribe.
3. fetch 응답 → 모든 observer 의 setState 트리거 → A, B 모두 re-render.
4. ComponentA unmount → observer 1개 → cleanup 카운트.
5. ComponentB unmount → observer 0 → cacheTime 후 GC.

### 3.3 핵심 통찰
**Observer pattern + reference counting**. 한 fetch, 여러 subscriber.

### 3.4 직접 구현한다면
```typescript
class QueryStore {
    private cache = new Map<string, any>();
    private observers = new Map<string, Set<() => void>>();
    private inflight = new Map<string, Promise<any>>();

    subscribe(key: string, callback: () => void) {
        const set = this.observers.get(key) ?? new Set();
        set.add(callback);
        this.observers.set(key, set);
        return () => set.delete(callback);
    }

    async fetch(key: string, fn: () => Promise<any>) {
        if (this.inflight.has(key)) return this.inflight.get(key);
        const promise = fn().then(data => {
            this.cache.set(key, data);
            this.observers.get(key)?.forEach(cb => cb());
            this.inflight.delete(key);
            return data;
        });
        this.inflight.set(key, promise);
        return promise;
    }
}
```

원리. RHF 가 더 정밀 (focus refetch, retry, dedupe, etc).

---

## §4. Mutation + invalidation

### 4.1 useMutation
```typescript
const mutation = useMutation({
    mutationFn: (input) => createTarget(input),
    onSuccess: () => {
        queryClient.invalidateQueries({ queryKey: ["targets"] });   // refetch
    },
});

// 호출
mutation.mutate(input);
```

**아이디어**:
- write 작업은 useMutation.
- onSuccess 에서 관련 query invalidate → 자동 refetch.

### 4.2 VulnScope 의 mutation
VulnScope 는 직접 호출 + manual invalidation 패턴 (간단):
```typescript
async function onSubmit(values) {
    const target = await createTarget(values);
    const scan = await triggerScan(target.id);
    router.push(...);
}
```

`useMutation` 안 씀 — 단순 흐름. `mutation.mutate` 보다 직접 호출이 명확.

useMutation 권장 케이스:
- mutation 의 loading/error state 가 컴포넌트 여러 곳에서 사용.
- onSuccess hook 으로 cache 자동 invalidate.

### 4.3 Optimistic Update
```typescript
const mutation = useMutation({
    mutationFn: updateTarget,
    onMutate: async (newData) => {
        await queryClient.cancelQueries(["targets"]);
        const previous = queryClient.getQueryData(["targets"]);
        queryClient.setQueryData(["targets"], (old) =>
            old.map(t => t.id === newData.id ? newData : t)
        );
        return { previous };           // rollback 용
    },
    onError: (err, newData, ctx) => {
        queryClient.setQueryData(["targets"], ctx.previous);    // rollback
    },
    onSettled: () => {
        queryClient.invalidateQueries(["targets"]);             // refetch
    },
});
```

**아이디어**: 응답 기다리지 않고 UI 즉시 업데이트. 실패 시 rollback.

→ UX 빠름. VulnScope 미사용 (v0.2 후보).

### 4.4 직접 구현한다면
```typescript
async function mutate(fn, onSuccess, onError) {
    try {
        const result = await fn();
        onSuccess?.(result);
        return result;
    } catch (err) {
        onError?.(err);
        throw err;
    }
}
```

원리. invalidate 는 §2 의 prefix matching.

---

## §5. SWR 의 다른 자동 동작

### 5.1 refetchOnWindowFocus
사용자가 다른 탭 갔다 돌아오면 자동 refetch (stale 일 때).

→ "다른 사용자가 데이터 변경 가능" 가정. 신선화 보장.

### 5.2 refetchOnReconnect
네트워크 끊겼다 복구 시 자동 refetch.

### 5.3 retry on error
```typescript
defaultOptions: { queries: { retry: 1 } }
```
→ 실패 시 1번 재시도. 깜빡임 대응.

### 5.4 핵심 통찰
**server state 의 자동 신선화 + 회복**. naive 면 매번 직접.

---

## §6. 직접 구현 vs 라이브러리

### 6.1 직접 구현의 한계
위 §1~§5 의 모든 동작 (cache, observer, invalidate, mutation, optimistic, focus refetch, retry) 직접 구현 = 수천 줄 + 버그.

### 6.2 라이브러리 후보
- **TanStack Query** (React Query) — VulnScope 선택.
- **SWR** (Vercel) — 비슷한 개념. 더 단순.
- **Apollo Client** — GraphQL 전용.
- **RTK Query** — Redux Toolkit 기반.

### 6.3 핵심 차이
- React Query: REST/GraphQL 무관. 강력한 cache/invalidate.
- SWR: 더 작고 hook-first. 기능 적음.
- Apollo: GraphQL 깊이 통합.
- RTK Query: Redux 사용 시 자연.

VulnScope = REST + 단순 cache → **React Query**.

---

## §7. 패턴 매트릭스

| 문제 | 패턴 |
|---|---|
| server vs client state | server state library |
| 즉시 UI + 신선화 | SWR (Stale-While-Revalidate) |
| cache 식별 + invalidation 단위 | queryKey 위계 |
| 중복 fetch 회피 | Observer (multiple subscribers, single fetch) |
| write + cache 갱신 | useMutation + invalidate |
| 즉시 UI 응답 | Optimistic Update |
| 자동 신선화 | refetchOnWindowFocus / Reconnect |

---

## §8. 학습 포인트

1. **server state ≠ client state** — truth 가 외부.
2. **SWR pattern** = cache 즉시 + background refresh.
3. **staleTime** = fresh 기간. cacheTime = unmount 후 보관.
4. **queryKey 위계** = invalidation 의 단위.
5. **Observer** = 같은 key, 1 fetch, N subscriber.
6. **useMutation + invalidate** = write 후 cache 갱신.
7. **Optimistic Update** = UI 먼저, 실패 시 rollback.
8. **refetchOnWindowFocus** = 자동 신선화.
9. **invalidateQueries(prefix)** = 위계적 무효화.
10. **직접 구현 = 천 줄 + 버그** — 라이브러리 권장.

### 추가 참고
- TanStack Query: https://tanstack.com/query/latest
- SWR: https://swr.vercel.app/
- Tanner Linsley, [Why You Want React Query](https://ui.dev/why-react-query)
- Tao of React Query
