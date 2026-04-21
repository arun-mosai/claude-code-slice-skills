# Example Slice: `bookmark-v1`

**This is a reference implementation of a filled slice.** Trivial on purpose — "Alex (PM) bookmarks a Feature" is small enough to read in one sitting but exercises every template. Copy this structure (not the domain) when filling your own slice.

## Files in this slice

| File | Status | Notes |
|------|--------|-------|
| `00-SLICE-INDEX.md` | full | tie-together |
| `01-PERSONA.md` | full | gold-standard fidelity example |
| `02-JOURNEY.md` | full | golden path + 3 branches + 3 failures |
| `03-UI-SPEC.md` | stub | **finish before implementing** |
| `04-API-CONTRACT.md` | stub | finish before implementing |
| `05-DATA-MODEL.md` | stub | finish before implementing |
| `06-SECURITY-TENANCY.md` | stub | finish before merge |
| `07-OBSERVABILITY.md` | stub | finish before merge |
| `08-AI-SURFACE.md` | `n/a` | no AI in this slice — example of `n/a` mode |
| `09-SEED-DATA.md` | full | shows literal rows + assertion SQL |
| `10-TEST-PLAN.md` | stub | finish before merge |

The three FULL files (01, 02, 09) show the fidelity required. The STUB files show the minimum scaffold.

## How to verify this slice works (once stubs are completed)

```bash
# 1. Validate the slice
# Spawn the orchestrator agent with this dir path — all 11 templates must PASS.

# 2. Seed it
pnpm db:seed:pg --reset --slice=bookmark-v1

# 3. Assert seed succeeded
psql $DATABASE_URL -f docs/slices/00-example-bookmark-v1/assertions.sql

# 4. Run the slice's tests
pnpm test -- bookmark-v1
pnpm test:e2e -- --grep bookmark-v1

# 5. Log in as the persona
# email: alex.chen@acme-slice-demo.test
# password: demo123
# navigate to /features, click bookmark icon on "SSO Integration"
```
