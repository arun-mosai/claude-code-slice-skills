---
# =============================================================================
# TEST PLAN — turn "feels done" into "demonstrably done".
# Rule: Gherkin cites seed row IDs; every layer has explicit coverage.
# =============================================================================
slice_id: <from 00>                           # [REQUIRED]
status: draft                                 # draft | ready
coverage_targets:
  unit_pct: 80                                # % of new handlers/services
  integration: "all service boundaries + DB roundtrips"
  e2e_scenarios: <n>                          # count matching §1
  contract: "all inter-service calls"
test_files_added:
  unit: []
  integration: []
  e2e: []
  contract: []
---

# Test Plan: {{slice_name}}

## 1. Gherkin Scenarios  [REQUIRED — min N scenarios]

Must cover: every golden path step, every decision branch, every atypical path, every failure mode. `Given`s cite seed row IDs from 09, `When`s cite interactions from 03, `Then`s cite observable outcomes from 02.

Path: `docs/slices/<slice-id>/features/*.feature` — Cucumber-compatible.

### Scenario: Golden Path

```gherkin
Feature: {{journey_name}}

Background:
  Given the slice <slice-id> is seeded
  And the user "alex.chen@acme-slice-demo.test" is logged in

Scenario: STEP-001 through STEP-00N (golden path)
  Given feature "SSO Integration" exists in status "IN_PROGRESS"   # from 09 §4
  When the user clicks the bookmark icon on "SSO Integration"       # UI-001 interaction from 03
  Then a row is inserted into product.bookmarks                     # 02 success outcome
  And event "product.bookmark.created.v1" is published              # 07 event catalog
  And metric "product.bookmarks.created.total" increments by 1
  And the bookmark appears in the sidebar within 500ms
```

### Scenario: Atypical-1
```gherkin
Scenario: <atypical path name from 02>
  Given …
  When …
  Then …
```

### Scenario: Failure-Mode FAIL-001
```gherkin
Scenario: downstream service is down
  Given feature service is unavailable
  When the user clicks bookmark
  Then an error toast shows "<copy from 03 copy deck>"
  And no partial state is left in the DB
  And metric "product.bookmarks.error_rate" increments
  And alert "<slice>_error_rate_high" does NOT fire (single failure below threshold)
```

**Required scenario count:** |golden-path| + |atypical-paths| + |failure-modes| + |cross-tenant-authz-tests|.

## 2. Unit Tests  [REQUIRED]

Co-located in `__tests__/` per CLAUDE.md rule 10. Pattern: `<file>.test.ts`.

| File under test | Test file | What's mocked | Key cases |
|---|---|---|---|
| `apps/services/<svc>/handlers/<file>.ts` | `apps/services/<svc>/handlers/__tests__/<file>.test.ts` | DB + event bus | validation, authz, tenancy |
| `apps/services/<svc>/services/<file>.ts` | `…/__tests__/<file>.test.ts` | DB | business rules, state transitions |
| `apps/web/src/stores/<file>.ts` | `…/<file>.test.ts` | API client | state transitions, optimistic updates |

Run: `pnpm test -- <glob>`.

## 3. Integration Tests  [REQUIRED]

Per `agent_docs/testing-patterns.md` — use `getTestApp()` factory with Supertest against real DB (test schema).

| Boundary | Test file | Notes |
|---|---|---|
| HTTP + DB | `apps/services/<svc>/__tests__/integration/<slice>.test.ts` | creates real rows, reads them back, verifies tenancy |
| Event publish/subscribe | `apps/services/<svc>/__tests__/integration/<slice>-events.test.ts` | publish → consumer side effect |
| Gateway permission | `apps/api-gateway/__tests__/<slice>.test.ts` | fail-closed: unregistered route = 403 |

Pattern:
```typescript
import { getTestApp } from '../../../testing/test-app.js';

describe('<slice> integration', () => {
  let app: Express;
  let cleanup: () => Promise<void>;
  beforeAll(async () => { ({ app, cleanup } = await getTestApp('<svc>-svc')); });
  afterAll(async () => cleanup());

  it('creates and lists bookmarks, scoped to tenant', async () => {
    // … real DB
  });
});
```

## 4. Contract Tests  [REQUIRED IF inter-service calls]

Pact consumer-driven. Per `agent_docs/testing-patterns.md`.

| Consumer | Provider | Contract file | Interactions |
|---|---|---|---|
| `<svc-A>-svc` | `<svc-B>-svc` | `contracts/<A>-<B>.json` | |

Run: `pnpm test:contracts`.

## 5. E2E Playwright Tests  [REQUIRED]

Path: `apps/web/e2e/<slice>/<scenario>.e2e.ts`.

Locators MUST match copy from 03 copy deck exactly (so changing copy breaks the test loudly, not silently).

```typescript
import { test, expect } from '@playwright/test';

test.beforeEach(async ({ page }) => {
  await page.goto('/login');
  await page.fill('input[name="email"]', 'alex.chen@acme-slice-demo.test');
  await page.fill('input[name="password"]', 'demo123');
  await page.click('button:has-text("Sign in")');
});

test('golden path — bookmark a feature', async ({ page }) => {
  await page.goto('/features');
  const row = page.getByRole('row', { name: /SSO Integration/ });
  await row.getByRole('button', { name: 'Bookmark' }).click();
  await expect(page.getByText('Bookmarked')).toBeVisible();    // toast from copy deck
  await expect(page.getByTestId('bookmark-sidebar')).toContainText('SSO Integration');
});
```

Run: `pnpm test:e2e -- <slice>`.

## 6. Chaos & Failure Tests  [REQUIRED — min 3]

Deliberately break things; verify user-visible behavior matches 02 §6 failure modes and 03 state matrix.

| Break | How to induce | Expected UX | Test file |
|---|---|---|---|
| DB connection lost | Kill postgres mid-request | `state: error` with retry button | `chaos/<slice>-db.e2e.ts` |
| Downstream svc 503 | Mock service returns 503 | Toast with recovery copy from 03 | `chaos/<slice>-downstream.e2e.ts` |
| AI provider down (if AI feature) | Mock Anthropic 503 | Fallback UI per 08 §2.1.7 | `chaos/<slice>-ai.e2e.ts` |
| Tenancy break (malicious) | Send `X-Org-ID` for different tenant | 403 TENANT_MISMATCH | from 06 §6 authz tests |

## 7. Performance Tests  [REQUIRED]

Match SLOs from 04.

| Scenario | Load | Target p99 | Tool |
|---|---|---|---|
| List endpoint | 100 RPS × 10 tenants | from 04 SLO | k6 / artillery |
| Write endpoint | 50 RPS × 10 tenants | from 04 SLO | k6 |

Script path: `perf/<slice>.k6.js`. Run on seeded test tenant. CI runs nightly (not per-PR).

## 8. Accessibility Tests  [REQUIRED]

- `axe-core` run on every screen in 03 — **0 violations** threshold
- Keyboard-only golden-path completion — verified in CI via Playwright without mouse
- Screen-reader smoke check — record NVDA/VoiceOver transcript once per release

Playwright a11y snippet:
```typescript
import AxeBuilder from '@axe-core/playwright';

test('a11y — <screen>', async ({ page }) => {
  await page.goto('/…');
  const results = await new AxeBuilder({ page }).analyze();
  expect(results.violations).toEqual([]);
});
```

## 9. Regression Tests  [REQUIRED IF modifying existing screens/endpoints]

List existing tests that MUST still pass. Run them before merge:

| Existing test | Why it could break | Verified green on branch |
|---|---|---|
| | | |

## 10. Coverage Report  [REQUIRED]

Run `pnpm test:spec:coverage` and paste the summary. Slice sub-path coverage must meet targets in frontmatter.

## 11. CI Integration  [REQUIRED]

- [ ] Unit + spec tests: run on every PR
- [ ] Integration tests: run on every PR
- [ ] E2E tests for this slice: run on every PR (tagged `@<slice-id>`)
- [ ] Chaos tests: run nightly
- [ ] Perf tests: run nightly
- [ ] a11y tests: run on every PR

## 12. Manual Test Script  [REQUIRED]

10-step walkthrough a human runs before calling the slice done. Paste in release notes.

1. `pnpm db:seed:pg --reset --slice=<slice-id>`
2. Log in as `alex.chen@acme-slice-demo.test`
3. Navigate to `<route>`
4. Complete golden path (see 02 §3)
5. Verify `<observable outcome>` from 02 §2
6. Trigger atypical path — verify behavior
7. Open Grafana `<dashboard URL>` — verify metrics incremented
8. Check audit log: `SELECT * FROM audit_logs WHERE actor_user_id = '<alex-id>' ORDER BY created_at DESC LIMIT 5`
9. Test permission denied: log out, log in as read-only user, verify lock states from 03 state matrix
10. Cleanup: `pnpm db:seed:pg --cleanup --slice=<slice-id>`

## 13. Validator Will Fail If …

- Scenario count < (golden path + atypical paths + failure modes + authz tests)
- Any Gherkin scenario references a seed row ID not in 09
- Any Gherkin scenario references an endpoint_id not in 04
- E2E test locators don't match 03 copy deck
- No chaos test for each failure mode in 02
- No a11y test included
- Coverage below targets in frontmatter
- Manual test script missing
