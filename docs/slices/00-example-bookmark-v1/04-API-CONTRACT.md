---
slice_id: SLICE-bookmark-v1
journey_id: JOURNEY-bookmark-feature-v1
status: ready
services_touched:
  - product-svc
gateway_routes_added: 3
openapi_fragment_path: docs/slices/00-example-bookmark-v1/openapi.yaml
---

# API Contract: Feature Bookmarks

## 1. Endpoint Inventory

| endpoint_id | Method | Route | Service | Permission | Idempotent | UI screens |
|---|---|---|---|---|---|---|
| ENDPOINT-001 | POST   | `/api/v1/bookmarks`                | product-svc | `product:bookmark:write` | yes | UI-001 |
| ENDPOINT-002 | GET    | `/api/v1/bookmarks`                | product-svc | `product:bookmark:read`  | yes | UI-002 |
| ENDPOINT-003 | DELETE | `/api/v1/bookmarks/:featureId`     | product-svc | `product:bookmark:write` | yes | UI-001, UI-002 |

---

## 2. Per-Endpoint Spec

### 2.1 ENDPOINT-001 — Create bookmark

| Field | Value |
|---|---|
| Service | product-svc |
| Route | `POST /api/v1/bookmarks` |
| Permission | `product:bookmark:write` |
| Authentication | bearer-jwt |
| Tenant header | `X-Org-ID` |
| Rate limit | 60 req/min per user |
| Idempotency-Key header | optional (constraint makes create idempotent anyway) |
| p50 / p99 latency SLO | 50ms / 150ms |
| Max request body | 256 bytes |
| Max response body | 2 KB |

**Request schema:**
```yaml
type: object
required: [feature_id]
properties:
  feature_id:
    type: string
    format: uuid
additionalProperties: false
```

```typescript
import { z } from 'zod';
export const CreateBookmarkSchema = z.object({
  feature_id: z.string().uuid(),
}).strict();
export type CreateBookmarkBody = z.infer<typeof CreateBookmarkSchema>;
```

**Success response** — `201 Created`:
```json
{
  "data": {
    "id": "bbbbbbb1-0000-0000-0000-000000000042",
    "feature_id": "aaaaaaa1-0000-0000-0000-000000000001",
    "user_id": "22222222-2222-2222-2222-222222222222",
    "created_at": "2026-04-17T14:03:12.441Z"
  }
}
```

If already bookmarked (idempotent path): `200 OK` with existing row.

**Error responses:**

| Code | HTTP | Condition | Machine | Human UI copy |
|---|---|---|---|---|
| `VALIDATION_ERROR` | 400 | body missing `feature_id` or not uuid | field errors | "Something's off — refresh and try again" |
| `UNAUTHORIZED` | 401 | no/bad JWT | — | redirect to /login |
| `FORBIDDEN` | 403 | role lacks `product:bookmark:write` | permission denied | "You don't have access to bookmark features" |
| `NOT_FOUND` | 404 | feature_id doesn't exist or not visible to user | — | "That feature no longer exists" |
| `BOOKMARK_LIMIT` | 422 | user has ≥ 50 bookmarks (BRANCH-002) | "Limit reached" | "You've hit 50 bookmarks — remove one to add another" |
| `RATE_LIMITED` | 429 | > 60/min | Retry-After | "Slow down — try again in a moment" |
| `INTERNAL_ERROR` | 500 | unhandled | trace_id | "Something went wrong. We've logged it." |

**Side effects:**
| Effect | Details |
|---|---|
| DB write | INSERT into `product.bookmarks` with ON CONFLICT DO NOTHING |
| Event published | `product.bookmark.created.v1` on `product` stream |
| Audit log | `product.bookmark.created` (actor: user_id, target: feature_id) |

**Tenancy contract:**
- `org_id` source: `getRequiredOrgId(req)` from your shared utils package (reads `X-Org-ID`, validates against JWT claims)
- Applied in: WHERE clause on feature-visibility check + inserted into new row
- Missing header → 400 `TENANT_MISSING`
- Mismatch with JWT → 403 `TENANT_MISMATCH`

**Gateway route entry:**
```typescript
{
  method: 'POST',
  path: '/api/v1/bookmarks',
  permission: 'product:bookmark:write',
  service: 'product-svc',
  rateLimit: { window: '1m', max: 60 },
}
```

### 2.2 ENDPOINT-002 — List bookmarks (sidebar load)

| Field | Value |
|---|---|
| Route | `GET /api/v1/bookmarks` |
| Permission | `product:bookmark:read` |
| Rate limit | 300 req/min per user |
| p50 / p99 | 30ms / 100ms |

**Query params:**
```yaml
type: object
properties:
  limit:  { type: integer, minimum: 1, maximum: 50, default: 50 }
  cursor: { type: string, description: "opaque cursor from previous response meta.next_cursor" }
```

**Success** — `200 OK`:
```json
{
  "data": [
    {
      "id": "bbbbbbb1-...",
      "feature_id": "aaaaaaa1-0000-0000-0000-000000000001",
      "feature_name": "SSO Integration",
      "feature_status": "IN_PROGRESS",
      "created_at": "2026-04-17T14:03:12.441Z"
    }
  ],
  "meta": { "total": 7, "limit": 50, "next_cursor": null }
}
```

**Errors:** `UNAUTHORIZED`, `FORBIDDEN`, `INTERNAL_ERROR` (same envelope).

**Pagination:** cursor-based on `(created_at, id)` descending. Max `limit=50`. Most PMs have < 50 bookmarks so first page covers all in practice.

**Gateway route entry:**
```typescript
{
  method: 'GET',
  path: '/api/v1/bookmarks',
  permission: 'product:bookmark:read',
  service: 'product-svc',
  rateLimit: { window: '1m', max: 300 },
}
```

### 2.3 ENDPOINT-003 — Delete bookmark (unbookmark)

| Field | Value |
|---|---|
| Route | `DELETE /api/v1/bookmarks/:featureId` |
| Permission | `product:bookmark:write` |
| Rate limit | 60 req/min per user |
| p50 / p99 | 40ms / 120ms |

**Path param:** `featureId` (uuid).

**Success** — `204 No Content`. Idempotent — deleting an already-absent bookmark still returns 204.

**Errors:** `UNAUTHORIZED`, `FORBIDDEN`, `RATE_LIMITED`, `INTERNAL_ERROR`.

**Side effects:**
- DB: DELETE by `(org_id, user_id, feature_id)`
- Event: `product.bookmark.deleted.v1`
- Audit log: `product.bookmark.deleted`

**Gateway route entry:**
```typescript
{
  method: 'DELETE',
  path: '/api/v1/bookmarks/:featureId',
  permission: 'product:bookmark:write',
  service: 'product-svc',
  rateLimit: { window: '1m', max: 60 },
}
```

---

## 3. Cross-Service Event Producers/Consumers

| Event | Producer | Consumers | Retry |
|---|---|---|---|
| `product.bookmark.created.v1` | ENDPOINT-001 | analytics-svc (telemetry), learning-svc (future: recommender) | 3× exp-backoff, DLQ |
| `product.bookmark.deleted.v1` | ENDPOINT-003 | analytics-svc | 3× exp-backoff, DLQ |

Full schema in `07-OBSERVABILITY.md §6`.

## 4. Gateway Routes Summary

```typescript
// Append to apps/api-gateway/src/data/route-permissions.data.ts
export const bookmarkRoutes: RoutePermission[] = [
  { method: 'POST',   path: '/api/v1/bookmarks',            permission: 'product:bookmark:write', service: 'product-svc', rateLimit: { window: '1m', max: 60  } },
  { method: 'GET',    path: '/api/v1/bookmarks',            permission: 'product:bookmark:read',  service: 'product-svc', rateLimit: { window: '1m', max: 300 } },
  { method: 'DELETE', path: '/api/v1/bookmarks/:featureId', permission: 'product:bookmark:write', service: 'product-svc', rateLimit: { window: '1m', max: 60  } },
];
```

## 5. OpenAPI Fragment

See `./openapi.yaml` in this slice dir.

## 6. Breaking-Change Analysis

This slice adds new endpoints only. No existing endpoint modified. Zero breaking changes.
