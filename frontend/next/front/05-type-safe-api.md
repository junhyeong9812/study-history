# 05. Type-safe API 패턴

> 이 문서가 다루는 것: OpenAPI codegen, path-based 타입, 백엔드 SSOT — `openapi-fetch + openapi-typescript` 의 본질적 통찰.

---

## §0. fetch 의 type safety 문제

### 0.1 naive fetch
```typescript
const res = await fetch("/api/scans/123");
const data = await res.json();    // any!
console.log(data.status);          // 에러 안 잡힘. data.staus 도 통과
```

**문제**:
- 응답 타입이 `any`. compile time 검증 0.
- 백엔드가 `status` → `state` 로 rename → frontend silent break (런타임).
- IDE 자동완성 X.

### 0.2 수동 타입 선언
```typescript
type Scan = { id: string; status: string; ... };
const data: Scan = await res.json() as Scan;
```

**문제**:
- 타입 직접 작성 = 백엔드와 sync 수동.
- 백엔드 schema 변경 시 frontend 안 update → runtime 에서만 발견.
- type 만 맞춤 (실제 검증 X).

### 0.3 codegen
```typescript
// 자동 생성 (OpenAPI spec → TypeScript)
import { paths } from "./generated";
const data = await client.GET("/scans/{id}", { params: { path: { id: "123" } } });
// data 타입 자동 추론 from OpenAPI
```

→ **백엔드 spec 이 SSOT (Single Source of Truth)**. frontend 타입 자동.

---

## §1. OpenAPI 가 무엇이고 왜 SSOT 인가?

### 1.1 OpenAPI = REST API 명세 표준
**구조**:
- `paths`: endpoint URL + method + parameters + body + response.
- `components.schemas`: 재사용 타입 (Scan, Finding, ...).
- `tags`, `security`, `servers` 등 메타.

**format**: YAML 또는 JSON.

```yaml
# docs/api/openapi.yaml (VulnScope)
paths:
  /scans/{scanId}:
    get:
      parameters:
        - in: path
          name: scanId
          required: true
          schema:
            type: string
            format: uuid
      responses:
        "200":
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ScanView"
components:
  schemas:
    ScanView:
      type: object
      properties:
        id: { type: string, format: uuid }
        status: { type: string, enum: [QUEUED, RUNNING, DONE, FAILED, CANCELED] }
        ...
```

### 1.2 SSOT 의 의미
**Single Source of Truth** — 한 곳의 정의가 모두에게 진실.

VulnScope:
- `docs/api/openapi.yaml` 이 SSOT.
- 백엔드: 이 spec 기반 controller 구현 (또는 코드 → spec 자동 생성).
- frontend: 이 spec → TypeScript 타입 codegen.

→ spec 변경 시 양쪽 자동 (codegen).

### 1.3 변경 흐름
```
1. 백엔드 개발자 → openapi.yaml 수정 (예: ScanView 에 newField 추가)
2. frontend 가 npm run openapi:generate
3. → src/shared/api/generated.ts 자동 생성
4. → 사용 코드에서 newField 자동 보임
5. → 누락 시 compile error (타입 불일치)
```

→ **silent break 0**.

---

## §2. openapi-typescript — codegen 의 통찰

### 2.1 사용
```bash
npm run openapi:generate
# 내부적으로:
# openapi-typescript ../docs/api/openapi.yaml --output src/shared/api/generated.ts
```

### 2.2 생성된 코드 (예시)
```typescript
// generated.ts
export interface paths {
    "/scans": {
        get: {
            parameters: { query?: { ... } };
            responses: {
                200: {
                    content: {
                        "application/json": {
                            data: components["schemas"]["ScanView"][];
                            meta: { ... };
                        };
                    };
                };
            };
        };
        post: {
            requestBody: {
                content: { "application/json": { ... } };
            };
            responses: { 201: { ... } };
        };
    };
    "/scans/{scanId}": {
        get: { parameters: { path: { scanId: string } }; ... };
    };
    // ... 모든 endpoint
}

export interface components {
    schemas: {
        ScanView: {
            id: string;
            status: "QUEUED" | "RUNNING" | "DONE" | "FAILED" | "CANCELED";
            ...
        };
        FindingView: { ... };
        // ...
    };
}
```

### 2.3 핵심 통찰
**OpenAPI 의 모든 path 가 TypeScript 인덱스 타입으로**.

```typescript
type GetScanResponse = paths["/scans/{scanId}"]["get"]["responses"][200]["content"]["application/json"];
// = ScanView
```

→ 길지만 정확. 라이브러리가 이걸 wrap.

### 2.4 직접 구현한다면
1. OpenAPI spec parse (YAML → JSON).
2. 각 path/method/response 를 TypeScript interface 로 변환.
3. schema → type alias.
4. file 출력.

`openapi-typescript` 가 이걸 자동. 정밀 (nullable, oneOf, allOf 등 OpenAPI 모든 feature).

---

## §3. openapi-fetch — typed client wrapper

### 3.1 사용
```typescript
import createClient from "openapi-fetch";
import type { paths } from "./generated";

export const client = createClient<paths>({
    baseUrl: "/api",
    credentials: "include",
});
```

### 3.2 타입 안전 호출
```typescript
const { data, error, response } = await client.GET("/scans/{scanId}", {
    params: { path: { scanId: "123" } },
});
//        ↑ "/scans/{scanId}" 자동완성
//        ↑ params.path.scanId 자동완성
//        ↑ data 타입 자동 추론 (= ScanView)
```

POST:
```typescript
const { data, error } = await client.POST("/scans", {
    body: {
        targetId: "uuid",
        profileId: "uuid",
    },
    //   ↑ body 의 모든 필드 자동 추론
});
```

### 3.3 핵심 통찰
**path string 이 타입의 key**. `paths["/scans/{scanId}"]["get"]` 인덱스 타입 자동.

→ runtime overhead 0. 컴파일 타임에 타입 검증.

### 3.4 fetch 와 차이
- 표준 fetch: URL string + body 직접 작성. 타입 0.
- openapi-fetch: 같은 fetch, but path/body/response 모두 타입.

크기:
- openapi-fetch: ~5KB (gzipped).
- axios: ~15KB.

### 3.5 VulnScope 의 활용
```typescript
// features/scan/api.ts
import { client } from "@/shared/api/client";
import type { components } from "@/shared/api/generated";

export type ScanView = components["schemas"]["ScanView"];

export async function triggerScan(targetId: string, profileId?: string): Promise<ScanView> {
    const { data, error, response } = await client.POST("/scans", {
        body: { targetId, profileId: profileId ?? STANDARD_PROFILE_ID },
    });
    if (error) throw new Error(`스캔 시작 실패 (${response.status})`);
    return data!;
}
```

- `components["schemas"]["ScanView"]` — schema type 직접 추출.
- `client.POST` — typed.
- error 분기 후 `data!` (non-null assertion).

### 3.6 직접 구현한다면
```typescript
type Paths = { "/scans/{scanId}": { get: { ... } } };

class TypedClient<P> {
    async GET<K extends keyof P>(path: K, opts: P[K]['get']['parameters']) {
        // path 의 {scanId} 를 opts.path.scanId 로 replace
        // fetch 호출
    }
}
```

원리. openapi-fetch 가 정밀 (params 분리, body 처리, error type, header).

---

## §4. 백엔드 spec 생성 vs 수동

### 4.1 두 방향
1. **spec-first**: openapi.yaml 직접 작성 → 백엔드 + frontend 모두 spec 따름.
2. **code-first**: 백엔드 코드에 어노테이션 → spec 자동 생성.

### 4.2 VulnScope 의 선택
**spec-first**. `docs/api/openapi.yaml` 직접 작성. 백엔드/frontend 둘 다 reference.

### 4.3 trade-off
| | spec-first | code-first |
|---|---|---|
| 생성 노력 | spec 직접 작성 (시간) | 백엔드 코드 → 자동 |
| sync 보장 | 매뉴얼 (백엔드 변경 시 spec 갱신) | 자동 |
| 백엔드 의존 | spec 파일 있으면 OK | 백엔드 빌드 필수 |
| design-first | YES (spec 부터 설계) | 코드 먼저 → spec 따라감 |

VulnScope 는 디자인 우선 → spec-first.

### 4.4 code-first 후보 (Spring)
- `springdoc-openapi` — controller annotation → spec 자동.
- 백엔드 빌드 시 `/v3/api-docs` endpoint 가 spec serve.

VulnScope 는 spec-first 의 명시성 + design 단계 가치 우선.

---

## §5. 함정 + 실전 팁

### 5.1 envelope 처리
백엔드의 `{data, meta}` envelope 응답:
```typescript
const { data } = await client.GET("/scans");
//    ^ data = { data: ScanView[], meta: {...} }
const scans = data?.data ?? [];   // unwrap
```

→ 한 번 더 unwrap.

VulnScope 의 `listScans` 는 raw fetch 사용 — typed client 의 envelope 매칭이 어려운 케이스에서 fallback.

### 5.2 path parameter 자동 substitution
```typescript
client.GET("/scans/{scanId}", { params: { path: { scanId: "123" } } });
// → 실제 URL: /scans/123
```

→ 직접 string concat 안 함.

### 5.3 query parameter
```typescript
client.GET("/scans", {
    params: { query: { page: 0, size: 20, status: "DONE" } },
});
// → /scans?page=0&size=20&status=DONE
```

### 5.4 type narrowing
```typescript
const { data, error } = await client.GET("/scans/{id}", ...);
if (error) {
    // data 는 undefined
    return;
}
// 여기부터 data 는 non-null
console.log(data.status);
```

→ TypeScript 의 type narrowing 활용.

### 5.5 enum 자동 추론
```typescript
const status: "QUEUED" | "RUNNING" | "DONE" | "FAILED" | "CANCELED" = data.status;
```

→ 백엔드 enum 이 union type 으로. 잘못된 값 compile error.

---

## §6. 패턴 매트릭스

| 문제 | 패턴 |
|---|---|
| 백엔드-frontend 타입 sync | OpenAPI spec SSOT |
| spec → TypeScript | openapi-typescript codegen |
| typed fetch | openapi-fetch with `<paths>` |
| path/query/body 자동완성 | path string 인덱스 타입 |
| envelope 처리 | data.data unwrap |
| error 분기 | type narrowing (if error) |

---

## §7. 학습 포인트

1. **fetch 는 any** — 수동 type 도 sync 수동.
2. **OpenAPI = SSOT** — 한 곳 변경, 모두 반영.
3. **codegen 으로 sync 자동** — silent break 0.
4. **path string 이 타입의 key** — `paths["/scans/{id}"]["get"]`.
5. **runtime overhead 0** — 컴파일 타임 타입.
6. **path/query/body 자동 substitution** — string concat 0.
7. **type narrowing** — error 분기 후 data non-null.
8. **enum 자동 union** — 잘못된 값 compile error.
9. **spec-first vs code-first** — design 우선 → spec-first.
10. **envelope 처리** — 한 번 더 unwrap.

### 추가 참고
- OpenAPI Spec: https://swagger.io/specification/
- openapi-fetch: https://openapi-ts.dev/openapi-fetch/
- openapi-typescript: https://openapi-ts.dev/
- spec-first vs code-first 토론: https://nordicapis.com/api-design-first-vs-code-first/
