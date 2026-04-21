---
# =============================================================================
# API CONTRACT — frozen before code. Handlers match these schemas exactly.
# Rule: every endpoint has a gateway route entry + permission + error taxonomy.
# =============================================================================
slice_id: <from 00>                           # [REQUIRED]
journey_id: <from 02>                         # [REQUIRED]
status: draft                                 # draft | ready
services_touched:                             # [REQUIRED] from your monorepo's services
  - <service-name>-svc
gateway_routes_added: <n>                     # [REQUIRED] must match rows in §4
openapi_fragment_path: <docs/slices/<slice-id>/openapi.yaml>   # [REQUIRED] emit this file too
---

# API Contract: {{slice_name}}

## 1. Endpoint Inventory  [REQUIRED]

| endpoint_id | Method | Route | Service | Permission | Idempotent | UI screens (03) |
|---|---|---|---|---|---|---|
| ENDPOINT-001 | POST | `/api/v1/…` | `<svc>` | `<perm>` | yes | UI-001 |
| ENDPOINT-002 | GET | `/api/v1/…` | `<svc>` | `<perm>` | yes | UI-001 |

---

## 2. Per-Endpoint Spec  [REQUIRED — repeat for each endpoint]

### 2.1 Endpoint: `ENDPOINT-001`

#### Header
| Field | Value |
|---|---|
| Service | `<svc>-svc` |
| Method + route | `POST /api/v1/…` |
| Permission (from 06) | `<perm-code>` |
| Authentication | bearer-jwt |
| Tenant header | `X-Org-ID` \| `X-Tenant-Id` |
| Rate limit | `<n>/minute per user` |
| Idempotency-Key header | required \| optional \| n/a |
| p50 latency SLO | `<ms>` |
| p99 latency SLO | `<ms>` |
| Max request body | `<KB>` |
| Max response body | `<KB>` |

#### 2.2 Request Schema  [REQUIRED — JSON Schema]

```yaml
# request body schema
type: object
required: [field_a, field_b]
properties:
  field_a:
    type: string
    minLength: 1
    maxLength: 200
  field_b:
    type: integer
    minimum: 0
    maximum: 1000
  field_c:
    type: string
    format: uuid
    description: references <table>.<id>
additionalProperties: false
```

Zod schema (paste-ready for handler):

```typescript
import { z } from 'zod';

export const RequestSchema = z.object({
  field_a: z.string().min(1).max(200),
  field_b: z.number().int().min(0).max(1000),
  field_c: z.string().uuid(),
}).strict();

export type RequestBody = z.infer<typeof RequestSchema>;
```

#### 2.3 Response — Success

Status: `200 OK` or `201 Created` or `204 No Content`

```yaml
type: object
required: [data]
properties:
  data:
    type: object
    required: [id, created_at]
    properties:
      id: { type: string, format: uuid }
      field_a: { type: string }
      created_at: { type: string, format: date-time }
      updated_at: { type: string, format: date-time }
```

Example body:

```json
{
  "data": {
    "id": "01HXX…",
    "field_a": "…",
    "created_at": "2026-04-15T10:00:00Z",
    "updated_at": "2026-04-15T10:00:00Z"
  }
}
```

For list endpoints: `{ "data": [...], "meta": { "total": n, "page": 1, "limit": 50 } }` per CLAUDE.md envelope rule.

#### 2.4 Response — Errors  [REQUIRED — enumerate all]

| Code | HTTP | Condition | Machine message | Human UI copy (maps to 03 copy deck) |
|---|---|---|---|---|
| `VALIDATION_ERROR` | 400 | schema violation | field-level errors | "Please fix the highlighted fields" |
| `UNAUTHORIZED` | 401 | no/invalid JWT | "Unauthorized" | redirect to login |
| `FORBIDDEN` | 403 | permission denied | "Insufficient permissions for <perm>" | "You don't have access" |
| `NOT_FOUND` | 404 | entity doesn't exist or not in tenant | "Not found" | "That <thing> doesn't exist" |
| `CONFLICT` | 409 | optimistic-lock / duplicate | "Resource was modified; refresh" | "Someone else updated this. Reload?" |
| `RATE_LIMITED` | 429 | exceeded quota | Retry-After header | "Slow down — try again in a moment" |
| `INTERNAL_ERROR` | 500 | unhandled | trace_id | "Something went wrong. We've logged it." |
| `DEPENDENCY_UNAVAILABLE` | 503 | downstream svc or AI provider down | "Service temporarily unavailable" | "Temporarily unavailable. Retry later." |

All errors use standard envelope:
```json
{ "error": { "code": "CODE", "message": "…", "details": { … }, "trace_id": "…" } }
```

#### 2.5 Side Effects  [REQUIRED]

| Effect | Details |
|---|---|
| DB writes | `INSERT INTO <table> (…)`, `UPDATE <table> SET … WHERE org_id=$1 AND id=$2` |
| Events published (to 07) | `<event.name.v1>` with payload `<…>` |
| External calls | `<system>`, `<endpoint>`, timeout `<ms>`, retry policy |
| Cache invalidation | `<cache-key patterns>` |
| Audit log entry | yes (action: `<verb>`, actor: userId, target: entityId) |

#### 2.6 Tenancy Contract  [REQUIRED]

- `org_id` source: `req.headers['x-org-id']` via your tenant-resolver helper (e.g. `getRequiredOrgId()` from a shared utils package)
- Applied to every query: yes (cite handler line after code written)
- If header missing: 400 `TENANT_MISSING`
- If header doesn't match JWT: 403 `TENANT_MISMATCH`
- Cross-tenant access prevention: `WHERE org_id = $orgId` on every SELECT/UPDATE/DELETE

#### 2.7 Pagination (list endpoints only)  [REQUIRED IF LIST]

- Strategy: cursor | offset | keyset
- Max `limit`: `<n>`
- Default `limit`: `<n>`
- Cursor format: `base64(created_at || id)` or similar
- Response includes: `meta.next_cursor` or `meta.total`

#### 2.8 Gateway Route Entry  [REQUIRED — paste-ready]

To add to `apps/api-gateway/src/data/route-permissions.data.ts`:

```typescript
{
  method: 'POST',
  path: '/api/v1/…',
  permission: '<perm-code>',
  service: '<svc>-svc',
  rateLimit: { window: '1m', max: <n> },
}
```

Gateway fail-closed rule (CLAUDE.md §Critical Rules): absent = 403.

---

## 3. Cross-Service Event Producers/Consumers  [REQUIRED IF ANY]

| Event name | Producer endpoint | Consumer services | Schema | Retry policy |
|---|---|---|---|---|
| `<event.name.v1>` | ENDPOINT-001 | `<svc>-svc` subscriber | see 07-OBS §event catalog | 3× exp-backoff |

---

## 4. Gateway Routes Summary Table  [REQUIRED]

All added routes in one table (matches `route-permissions.data.ts` additions):

| Method | Path | Permission | Rate limit | Service |
|---|---|---|---|---|
| | | | | |

---

## 5. OpenAPI Fragment  [REQUIRED]

Emit a literal OpenAPI 3.1 file at `docs/slices/<slice-id>/openapi.yaml`. Must validate via `npx @redocly/cli lint`.

## 6. Breaking-Change Analysis  [REQUIRED]

If this slice modifies existing endpoints:

| Endpoint | Change | Breaking? | Migration plan |
|---|---|---|---|
| | | | |

## 7. Validator Will Fail If …

- Any endpoint missing request or response schema
- Any endpoint missing rate limit or permission
- OpenAPI fragment doesn't lint clean
- Gateway route entries not paste-ready (missing fields)
- Error taxonomy has fewer than 5 codes
- No tenancy contract section
- List endpoint missing pagination section
- Event produced by endpoint not registered in 07-OBSERVABILITY event catalog
