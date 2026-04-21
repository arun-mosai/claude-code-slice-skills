---
slice_id: SLICE-bookmark-v1
status: ready
seed_script_path: packages/database/src/cli/seed-slices/bookmark-v1.ts
seed_cli_flag: --slice=bookmark-v1
tenant_id: 11111111-1111-1111-1111-111111111111
persona_login:
  email: alex.chen@flowroom-slice-demo.test
  password_env: SEED_USER_PASSWORD
row_count_target: 14
---

# Seed Data: Feature Bookmarks

## 1. How to Seed

```bash
SEED_USER_PASSWORD=demo123 pnpm db:seed:pg --reset --slice=bookmark-v1
psql $DATABASE_URL -f docs/slices/00-example-bookmark-v1/assertions.sql
```

## 2. Tenant + User Setup

```sql
INSERT INTO public.organizations (id, slug, name, industry, created_at)
VALUES ('11111111-1111-1111-1111-111111111111', 'flowroom-slice-demo',
        'Flowroom (slice demo)', 'SaaS Productivity', now())
ON CONFLICT (id) DO NOTHING;

INSERT INTO public.users (id, org_id, email, password_hash, first_name, last_name, role_id, created_at)
VALUES ('22222222-2222-2222-2222-222222222222',
        '11111111-1111-1111-1111-111111111111',
        'alex.chen@flowroom-slice-demo.test',
        crypt(coalesce(current_setting('app.seed_password', true), 'demo123'), gen_salt('bf')),
        'Alex', 'Chen',
        (SELECT id FROM public.roles WHERE code='product_manager'),
        now() - interval '14 months')
ON CONFLICT (id) DO NOTHING;
```

Login: `alex.chen@flowroom-slice-demo.test` / `demo123`.

## 3. Collaborator Users

| Email | Role | Purpose |
|---|---|---|
| sarah.okafor@flowroom-slice-demo.test | product_director | Alex's manager (for Atypical-2 share-via-URL) |
| jamie@flowroom-slice-demo.test | engineering_lead | Collaborator; sees different subset of features |

```sql
INSERT INTO public.users (id, org_id, email, password_hash, first_name, last_name, role_id, created_at)
VALUES
  ('33333333-3333-3333-3333-333333333333', '11111111-1111-1111-1111-111111111111',
   'sarah.okafor@flowroom-slice-demo.test',
   crypt('demo123', gen_salt('bf')), 'Sarah', 'Okafor',
   (SELECT id FROM public.roles WHERE code='product_director'), now() - interval '3 years'),
  ('44444444-4444-4444-4444-444444444444', '11111111-1111-1111-1111-111111111111',
   'jamie@flowroom-slice-demo.test',
   crypt('demo123', gen_salt('bf')), 'Jamie', 'Rivera',
   (SELECT id FROM public.roles WHERE code='engineering_lead'), now() - interval '2 years')
ON CONFLICT (id) DO NOTHING;
```

## 4. Entity Rows — Journey State Coverage

Five features spanning every lifecycle state Alex will encounter. Recognizable names.

```sql
INSERT INTO product.features (id, org_id, name, description, status, priority, owner_user_id, created_at, updated_at)
VALUES
  ('aaaaaaa1-0000-0000-0000-000000000001', '11111111-1111-1111-1111-111111111111',
   'SSO Integration', 'SAML + OIDC support for enterprise customers',
   'IN_PROGRESS', 'p0', '22222222-2222-2222-2222-222222222222',
   now() - interval '7 days', now() - interval '2 days'),

  ('aaaaaaa1-0000-0000-0000-000000000002', '11111111-1111-1111-1111-111111111111',
   'Bulk Export to CSV', 'Export workspace data for enterprise compliance',
   'PLANNED', 'p2', '22222222-2222-2222-2222-222222222222',
   now() - interval '14 days', now() - interval '14 days'),

  ('aaaaaaa1-0000-0000-0000-000000000003', '11111111-1111-1111-1111-111111111111',
   'Presence Indicators', 'Live avatars showing who is viewing a workspace',
   'PROPOSED', 'p1', '22222222-2222-2222-2222-222222222222',
   now() - interval '30 days', now() - interval '30 days'),

  ('aaaaaaa1-0000-0000-0000-000000000004', '11111111-1111-1111-1111-111111111111',
   'Dark Mode', 'Full dark theme across product',
   'IN_QA', 'p2', '44444444-4444-4444-4444-444444444444',
   now() - interval '45 days', now() - interval '3 days'),

  ('aaaaaaa1-0000-0000-0000-000000000005', '11111111-1111-1111-1111-111111111111',
   'Better Onboarding Tooltips', 'Contextual hints for first-time users',
   'SHIPPED', 'p3', '22222222-2222-2222-2222-222222222222',
   now() - interval '90 days', now() - interval '60 days')
ON CONFLICT (id) DO NOTHING;
```

## 5. Related Entities

No customer/deal entities needed for this slice — bookmarks are feature-scoped.

## 6. Edge-Case & Failure-Mode Rows

| Entity | State | Triggers failure mode | Notes |
|---|---|---|---|
| Feature "Hidden Feature" | restricted to `product_director` role only | FAIL-001 (permission-denied path for Alex) | Alex sees a 403 if she tries to bookmark it |
| Pre-existing bookmark | Alex already bookmarked "Presence Indicators" | BRANCH-001 (toggle off path) | Start state includes 1 bookmark |

```sql
-- Role-restricted feature (triggers FAIL-001 when Alex tries to bookmark it)
INSERT INTO product.features (id, org_id, name, description, status, priority, visibility_scope, created_at)
VALUES ('aaaaaaa1-0000-0000-0000-000000000099',
        '11111111-1111-1111-1111-111111111111',
        'Confidential: Acquisition Roadmap', 'Board-restricted M&A item',
        'PROPOSED', 'p0', 'director_only',
        now() - interval '2 days')
ON CONFLICT (id) DO NOTHING;

-- Pre-existing bookmark for BRANCH-001
-- (table schema is defined in 05-DATA-MODEL.md but shown here for demo)
INSERT INTO product.bookmarks (id, org_id, user_id, feature_id, created_at)
VALUES ('bbbbbbb1-0000-0000-0000-000000000001',
        '11111111-1111-1111-1111-111111111111',
        '22222222-2222-2222-2222-222222222222',
        'aaaaaaa1-0000-0000-0000-000000000003',
        now() - interval '1 day')
ON CONFLICT (id) DO NOTHING;
```

## 7. Time Relativity

All timestamps use `now() - interval '<n> days'` — re-seedable without drift.

## 8. Assertion Queries

Path: `docs/slices/00-example-bookmark-v1/assertions.sql`

```sql
-- Assertion 1: persona loggable
SELECT count(*) FILTER (WHERE email='alex.chen@flowroom-slice-demo.test')  = 1  AS persona_present,
       count(*) FILTER (WHERE email='sarah.okafor@flowroom-slice-demo.test') = 1 AS director_present
FROM public.users
WHERE org_id = '11111111-1111-1111-1111-111111111111';

-- Assertion 2: 5 lifecycle-state features + 1 restricted
SELECT status, count(*) AS n
FROM product.features
WHERE org_id = '11111111-1111-1111-1111-111111111111'
GROUP BY status
ORDER BY status;
-- Expected: IN_PROGRESS=1, IN_QA=1, PLANNED=1, PROPOSED=2, SHIPPED=1

-- Assertion 3: pre-existing bookmark for BRANCH-001
SELECT count(*) = 1 AS precondition_ok
FROM product.bookmarks
WHERE user_id = '22222222-2222-2222-2222-222222222222'
  AND feature_id = 'aaaaaaa1-0000-0000-0000-000000000003';

-- Assertion 4: no stale fixture bookmarks leaked from other slices
SELECT count(*) AS foreign_bookmarks
FROM product.bookmarks
WHERE org_id <> '11111111-1111-1111-1111-111111111111';
-- Expected: unchanged from baseline
```

## 9. TypeScript Seed Plan

```typescript
// packages/database/src/cli/seed-slices/bookmark-v1.ts
import type { TenantSeedPlan } from '../seed-postgres-plan.js';

export const bookmarkV1SeedPlan: TenantSeedPlan = {
  slice_id: 'bookmark-v1',
  organization: {
    id: '11111111-1111-1111-1111-111111111111',
    slug: 'flowroom-slice-demo',
    name: 'Flowroom (slice demo)',
  },
  users: [
    { id: '22222222-2222-2222-2222-222222222222', email: 'alex.chen@flowroom-slice-demo.test',   role: 'product_manager'  },
    { id: '33333333-3333-3333-3333-333333333333', email: 'sarah.okafor@flowroom-slice-demo.test', role: 'product_director'},
    { id: '44444444-4444-4444-4444-444444444444', email: 'jamie@flowroom-slice-demo.test',        role: 'engineering_lead'},
  ],
  features: [
    { id: 'aaaaaaa1-0000-0000-0000-000000000001', name: 'SSO Integration',           status: 'IN_PROGRESS' },
    { id: 'aaaaaaa1-0000-0000-0000-000000000002', name: 'Bulk Export to CSV',         status: 'PLANNED'     },
    { id: 'aaaaaaa1-0000-0000-0000-000000000003', name: 'Presence Indicators',        status: 'PROPOSED'    },
    { id: 'aaaaaaa1-0000-0000-0000-000000000004', name: 'Dark Mode',                  status: 'IN_QA'       },
    { id: 'aaaaaaa1-0000-0000-0000-000000000005', name: 'Better Onboarding Tooltips', status: 'SHIPPED'     },
    { id: 'aaaaaaa1-0000-0000-0000-000000000099', name: 'Confidential: Acquisition Roadmap', status: 'PROPOSED', visibility_scope: 'director_only' },
  ],
  bookmarks: [
    { user_id: '22222222-2222-2222-2222-222222222222',
      feature_id: 'aaaaaaa1-0000-0000-0000-000000000003',
      created_offset_days: -1 },
  ],
};
```

Register in `seed-postgres.ts`:

```typescript
import { bookmarkV1SeedPlan } from './seed-slices/bookmark-v1.js';
SLICE_PLANS['bookmark-v1'] = bookmarkV1SeedPlan;
```

## 10. Expected Post-Seed State Summary

| Entity | Count | Notes |
|---|---|---|
| Organizations | 1 | Flowroom slice demo |
| Users | 3 | Alex + Sarah + Jamie |
| Features | 6 | 5 visible + 1 director-only |
| Bookmarks | 1 | Alex already bookmarked "Presence Indicators" |

**Post-seed verification:** log in as Alex, go to `/features`, see 5 rows (director-only filtered out), open sidebar — 1 bookmark visible.

## 11. Cleanup

```bash
pnpm db:seed:pg --cleanup --slice=bookmark-v1
```

Deletes in FK order: bookmarks → features → users → organizations (only ones with the fixed demo UUIDs).
