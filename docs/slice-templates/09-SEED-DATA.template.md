---
# =============================================================================
# SEED DATA — literal rows making the slice runnable end-to-end.
# Rule: named recognizable entities; assertion queries return expected state.
# =============================================================================
slice_id: <from 00>                           # [REQUIRED]
status: draft                                 # draft | ready
seed_script_path: packages/database/src/cli/seed-slices/<slice>.ts   # [REQUIRED]
seed_cli_flag: --slice=<slice-id>             # [REQUIRED] e.g. --slice=bookmark-v1
tenant_id: <uuid>                             # [REQUIRED] fixed UUID so dev can log in repeatedly
persona_login:                                # [REQUIRED]
  email: alex.chen@acme-slice-demo.test
  password_env: SEED_USER_PASSWORD           # read from env; default "demo123"
row_count_target: <n>                         # total rows seeded for this slice
---

# Seed Data: {{slice_name}}

## 1. How to Seed  [REQUIRED]

```bash
# Reset everything and seed this slice
pnpm db:seed:pg --reset --slice=<slice-id>

# Seed on top of existing (no reset)
pnpm db:seed:pg --slice=<slice-id>

# Verify
psql $DATABASE_URL -f docs/slices/<slice-id>/assertions.sql
```

## 2. Tenant + User Setup  [REQUIRED]

The persona from 01 must be loggable. Fixed UUIDs so re-seeding doesn't change login credentials.

```sql
-- Organization (the persona's company)
INSERT INTO public.organizations (id, slug, name, industry, created_at)
VALUES
  ('11111111-1111-1111-1111-111111111111', 'acme-slice-demo', 'Acme (slice demo)', 'SaaS', now())
ON CONFLICT (id) DO NOTHING;

-- The persona user
INSERT INTO public.users (id, org_id, email, password_hash, first_name, last_name, role_id, created_at)
VALUES
  ('22222222-2222-2222-2222-222222222222',
   '11111111-1111-1111-1111-111111111111',
   'alex.chen@acme-slice-demo.test',
   crypt(current_setting('app.seed_password'), gen_salt('bf')),
   'Alex', 'Chen',
   (SELECT id FROM public.roles WHERE code='product_manager'),
   now())
ON CONFLICT (id) DO NOTHING;
```

**Developer login:** `alex.chen@acme-slice-demo.test` / `demo123` (or `$SEED_USER_PASSWORD`).

## 3. Collaborator Users  [REQUIRED IF journey touches other personas]

| Email | Role | Purpose |
|---|---|---|
| manager@acme-slice-demo.test | product_lead | Alex's manager (for approval paths) |
| teammate@acme-slice-demo.test | product_manager | Alex's peer |

## 4. Entity Rows — Journey State Coverage  [REQUIRED]

For every state referenced in 02-JOURNEY's decision branches and atypical paths, at least one seed row.

```sql
-- Example: Features in each lifecycle state (from journey 02)
INSERT INTO product.features (id, org_id, name, status, priority, created_at)
VALUES
  ('01HAAA...', '<org-id>', 'AI-Powered Search', 'PROPOSED',    'p1', now() - interval '30 days'),
  ('01HAAB...', '<org-id>', 'Bulk Export',       'PLANNED',     'p2', now() - interval '14 days'),
  ('01HAAC...', '<org-id>', 'SSO Integration',   'IN_PROGRESS', 'p0', now() - interval '7 days'),
  ('01HAAD...', '<org-id>', 'Dark Mode',         'IN_QA',       'p2', now() - interval '3 days'),
  ('01HAAE...', '<org-id>', 'Better Tooltips',   'SHIPPED',     'p3', now() - interval '60 days');
```

Names MUST be recognizable — not "Feature 1", "Feature 2". Use real-product-sounding names.

## 5. Related Entities  [REQUIRED]

Everything the journey touches, seeded with realistic relationships.

```sql
-- Customers (if journey references them)
INSERT INTO crm.customers (id, org_id, name, arr_cents, tier, ...)
VALUES
  ('01HCUS...', '<org-id>', 'Wayne Industries',  1200000000, 'enterprise', ...),
  ('01HCUS...', '<org-id>', 'Stark Industries',   480000000, 'growth',     ...),
  ('01HCUS...', '<org-id>', 'Pied Piper',          60000000, 'startup',    ...);

-- Deals, contacts, etc. — per your journey
```

## 6. Edge-Case & Failure-Mode Rows  [REQUIRED]

One row per failure mode in 02-JOURNEY §6. Lets manual walkthrough + tests exercise these paths.

| Entity | State | Triggers failure mode | Notes |
|---|---|---|---|
| Feature "Broken import" | PROPOSED with 0 requirements | FAIL-001 (cannot advance without reqs) | UI shows blocking banner |
| Customer "Stale Corp" | last activity > 180d | FAIL-002 (churn signal) | Health score red |
| Subscription "Expiring Inc" | renewal_date < 30d | Atypical-1 (renewal path) | Shows renewal banner |

## 7. Time Relativity  [REQUIRED]

Use `now()` offsets so data stays fresh on every reseed.

- Never hardcode `'2025-04-15T…'` timestamps
- Use: `now() - interval '7 days'`, `now() + interval '30 days'`
- Derive other timestamps: `created_at + interval '2 hours'` for `updated_at`

## 8. Assertion Queries  [REQUIRED — fenced SQL]

Path: `docs/slices/<slice-id>/assertions.sql`

Must all return expected rows on a freshly-seeded DB.

```sql
-- Assertion 1: persona user exists
SELECT count(*) AS expected_1 FROM public.users
WHERE email = 'alex.chen@acme-slice-demo.test';

-- Assertion 2: 5 features across lifecycle states
SELECT status, count(*)
FROM product.features
WHERE org_id = '<org-id>'
GROUP BY status
ORDER BY status;
-- Expected: PROPOSED=1, PLANNED=1, IN_PROGRESS=1, IN_QA=1, SHIPPED=1

-- Assertion 3: at least one row per failure mode
SELECT name, status FROM product.features
WHERE org_id = '<org-id>' AND name IN ('Broken import', …);
```

Helper: `packages/database/src/cli/assert.ts` runs these and fails loudly if anything diverges.

## 9. TypeScript Seed Plan  [REQUIRED]

Extends `TenantSeedPlan` from `packages/database/src/cli/seed-postgres-plan.ts`.

```typescript
// packages/database/src/cli/seed-slices/<slice>.ts
import type { TenantSeedPlan } from '../seed-postgres-plan.js';

export const <slice>SeedPlan: TenantSeedPlan = {
  slice_id: '<slice-id>',
  organization: {
    id: '11111111-1111-1111-1111-111111111111',
    slug: 'acme-slice-demo',
    name: 'Acme (slice demo)',
  },
  users: [
    { id: '2222…', email: 'alex.chen@…', role: 'product_manager' },
    { id: '3333…', email: 'manager@…',   role: 'product_lead' },
  ],
  features: [ /* from §4 */ ],
  customers: [ /* from §5 */ ],
  // …
};
```

Register in `seed-postgres.ts` so `--slice=<slice-id>` picks it up.

## 10. Expected Post-Seed State Summary  [REQUIRED]

| Entity type | Count | Notes |
|---|---|---|
| Organizations | 1 | Acme slice demo |
| Users | 3 | Alex + manager + teammate |
| Features | 5 | one per lifecycle state |
| Customers | 3 | enterprise + growth + startup |
| … | | |

**Dev can log in, navigate to `{{first_screen_route}}`, and see `{{expected_visible_rows}}`.**

## 11. Cleanup  [REQUIRED]

How to remove this slice's seed data without touching others:

```bash
pnpm db:seed:pg --cleanup --slice=<slice-id>
```

Implemented via: `DELETE FROM … WHERE org_id='<fixed-tenant-id>'` respecting FK order.

## 12. Validator Will Fail If …

- Persona from 01 not seeded with loggable credentials
- Any entity count = 0 where the journey needs it
- Fewer than 1 row per failure mode from 02
- Timestamps hardcoded (any ISO date string)
- Assertion queries return unexpected counts
- Entity names look like "Feature 1", "Customer A" (not recognizable)
- Seed re-run twice fails (not idempotent — missing ON CONFLICT DO NOTHING or similar)
- TypeScript seed plan not registered in `seed-postgres.ts`
