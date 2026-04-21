---
# =============================================================================
# SECURITY & TENANCY — fail-closed invariants, permission matrix, authz tests.
# Rule: every endpoint in the matrix; every table has a tenancy invariant line.
# =============================================================================
slice_id: <from 00>                           # [REQUIRED]
status: draft                                 # draft | ready
pii_classification: none | low | medium | high   # [REQUIRED]
compliance_tags:                              # [REQUIRED — empty list ok]
  - SOC2                                      # SOC2 | GDPR | HIPAA | PCI
encryption:
  at_rest: postgres-tde                       # what applies
  in_transit: tls-1.3
audit_log_writes: <n>                         # count of actions that write audit log
secrets_added: []                             # new env vars
---

# Security & Tenancy: {{slice_name}}

## 1. Permission Matrix  [REQUIRED — fenced YAML]

Every `endpoint_id` from 04 × every role that should have access → allow | deny | conditional.

```yaml
permissions:
  - endpoint_id: ENDPOINT-001
    route: POST /api/v1/…
    permissions_required:
      - <perm-code>
    role_matrix:
      admin: allow
      manager: allow
      ic: conditional   # see condition below
      read_only: deny
      external: deny
    conditions:
      ic: "user must be assigned to the <entity> OR be the creator"
  - endpoint_id: ENDPOINT-002
    …
```

Cross-reference to 04 — every endpoint must appear here.

## 2. Route-Permissions Registration  [REQUIRED — paste-ready]

To add to `apps/api-gateway/src/data/route-permissions.data.ts`. Gateway is fail-closed — missing entry returns 403.

```typescript
// Slice: {{slice_id}}
export const <slice>Routes: RoutePermission[] = [
  { method: 'POST', path: '/api/v1/…', permission: '<perm-code>', service: '<svc>-svc' },
  { method: 'GET',  path: '/api/v1/…', permission: '<perm-code>', service: '<svc>-svc' },
];
```

## 3. Tenancy Invariants  [REQUIRED — per new table]

Every table from 05-DATA-MODEL. For each: the literal query filter that prevents cross-tenant reads, and the code path that enforces it.

| Table | Tenant column | Enforcement file:line | Test that proves it |
|---|---|---|---|
| `<schema>.<table_a>` | `org_id` | `apps/services/<svc>/src/handlers/<file>.ts:42` | `SLICE-xxx-AUTHZ-001` |
| `<schema>.<table_b>` | `org_id` | … | … |

**Invariant test pattern** (paste into service tests):

```typescript
it('cannot read <table_a> from another tenant', async () => {
  const tenantA = await seed.org();
  const tenantB = await seed.org();
  await db.query(`INSERT INTO <table_a> (org_id, …) VALUES ($1, …)`, [tenantA.id]);
  const res = await api.as(tenantB.admin).get('/api/v1/…');
  expect(res.body.data).toEqual([]);     // must not leak rows from tenantA
});
```

## 4. PII Inventory  [REQUIRED]

| Column | Classification | Encryption at rest | Redaction in logs | Consent basis |
|---|---|---|---|---|
| `users.email` | medium (PII) | yes (TDE) | hash/mask | legit interest |
| `<table>.<col>` | | | | |

Fields that MUST NOT appear in logs: list them. Log redactor path: `packages/observability/src/redactors.ts`.

## 5. Rate Limiting & Abuse Surface  [REQUIRED]

| Endpoint | Limit | Per | Reason |
|---|---|---|---|
| ENDPOINT-001 | 60 req/min | user | writes; prevent spam |
| ENDPOINT-002 | 300 req/min | user | reads; normal browsing |
| `<expensive AI endpoint>` | 10 req/min | org | cost cap |

Abuse scenarios considered:
- [ ] Credential stuffing (if login-related)
- [ ] Enumeration (list endpoints: opaque cursors, not sequential IDs)
- [ ] Data exfiltration (log/alert on abnormal list-size + export)
- [ ] CSRF (state-changing endpoints require `X-CSRF-Token` or same-site cookie + Origin check)
- [ ] IDOR (every entity lookup filtered by `org_id`)

## 6. Authz Test Cases  [REQUIRED — min 5]

Gherkin-style. Will be auto-imported into 10-TEST-PLAN.

```gherkin
Scenario: cross-tenant read rejected
  Given tenant-A has <entity> "X"
  And tenant-B has user "bob" with admin role
  When bob queries GET /api/v1/<entity>/X
  Then the response is 404 NOT_FOUND (no leak of existence)

Scenario: role escalation attempt rejected
  Given user "alice" has role "ic"
  When alice calls POST /api/v1/<admin-endpoint>
  Then the response is 403 FORBIDDEN

Scenario: unauthenticated probe rejected
  When an unauthenticated request hits GET /api/v1/…
  Then the response is 401 UNAUTHORIZED

Scenario: expired JWT rejected
  Given user "alice" has an expired token
  When alice calls any endpoint
  Then the response is 401 TOKEN_EXPIRED

Scenario: tenant header spoofing rejected
  Given user "alice" is in tenant-A
  When alice sends X-Org-ID: <tenant-B-id>
  Then the response is 403 TENANT_MISMATCH
```

## 7. Secrets & Env Vars  [REQUIRED IF ANY NEW]

| Env var | Purpose | Required | Example value (fake) | Rotation policy |
|---|---|---|---|---|
| | | | | |

Secret-loading path: `packages/utils/src/config.ts` (validates at boot, crashes if missing).

## 8. Audit Log  [REQUIRED]

| Action | Audit event name | Actor | Target | Fields logged | Retention |
|---|---|---|---|---|---|
| create `<entity>` | `<svc>.<entity>.created` | userId | entityId | diff | 7y (SOC2) |
| update `<entity>` | `<svc>.<entity>.updated` | userId | entityId | before/after diff | 7y |
| delete `<entity>` | `<svc>.<entity>.deleted` | userId | entityId | last state | 7y |

Audit table: `public.audit_logs` (per existing codebase convention).

## 9. Threat Model  [REQUIRED — STRIDE lite]

| Threat | Vector | Mitigation in this slice | Residual risk |
|---|---|---|---|
| **S**poofing | stolen JWT | short TTL + refresh rotation | accept |
| **T**ampering | request body manipulation | Zod validation + DB constraints | accept |
| **R**epudiation | "I didn't do that" | audit log § 8 | accept |
| **I**nformation disclosure | cross-tenant leak | tenancy invariants § 3 | test covers |
| **D**enial of service | request flood | rate limits § 5 | accept |
| **E**scalation of privilege | IDOR | fail-closed + authz tests § 6 | test covers |

## 10. Compliance Callouts  [REQUIRED IF compliance_tags NON-EMPTY]

- **SOC2**: access logs retained 7y, change management via PR review, least-privilege via RBAC
- **GDPR**: data subject export (cite endpoint), deletion (cite soft-delete + purge), consent (cite field)
- **HIPAA**: n/a for this slice | applies because `<reason>` → list BAA-covered paths

## 11. Validator Will Fail If …

- Any endpoint from 04 missing from the permission matrix
- Any table from 05 missing a tenancy invariant row
- Fewer than 5 authz test cases
- PII classification not set (or set to "none" with PII-looking columns like `email`, `phone`, `address`)
- Audit log table has write actions but `audit_log_writes: 0` in frontmatter
- No rate limit specified for any endpoint from 04
- `permissions_required` empty for any endpoint
