# Porting the Slice Skills to Your Stack

The skills ship with an opinionated default stack. The phase structure, validator logic, resolution loop, atomic-commit rule, and blocker-verification pattern are **stack-neutral**. The specific file paths, conventions, and commands are the parts that need adapting.

This doc lists every stack assumption, where it lives in the skill files, and what you'd change to port to a different ecosystem.

---

## Default stack assumptions

| Assumption | Where it appears | What it means |
|---|---|---|
| TypeScript + ESM with `.js` import extensions | `slice-build/SKILL.md`, `SLICE-BUILDER-PROMPT.md` | `import { x } from './y.js'` |
| `pnpm` + monorepo layout | Test/lint/typecheck commands throughout `slice-build` | `pnpm test`, `pnpm turbo build --filter='*'` |
| `apps/services/<name>-svc/` for backend services | `slice-build/SKILL.md §6.2`, `04-API-CONTRACT.template.md` | Where new handlers land |
| `apps/web/src/` for React frontend | `slice-build/SKILL.md §6.5` | Where new screens land |
| `apps/api-gateway/src/config/route-permissions.data.ts` | `slice-build/SKILL.md` preflight, §6.2 | Fail-closed gateway permissions file |
| Raw `pg` (no ORM), snake_case DB, `org_id` tenancy column | `05-DATA-MODEL.template.md`, validator-data | Tenancy column + DDL conventions |
| Zod for validation, inline at top of handler | `04-API-CONTRACT.template.md`, `slice-build/SKILL.md §6.2` | Validation library + location |
| Vitest for unit/spec, Playwright for E2E | `slice-build/SKILL.md §6.6`, `10-TEST-PLAN.template.md` | Test framework |
| Co-located `.meta.md` files next to every source file | `slice-build/SKILL.md §6.7` step 7 | Your project's metadata convention |
| `packages/database/postgresql/migrations/` | `slice-build/SKILL.md §6.1`, preflight | Migration location |
| `infrastructure/docker/postgres/init/02-schema.sql` | `slice-build/SKILL.md` preflight (schema mirror) | Docker-init mirror of schema |

---

## How to adapt

### Option 1: edit the skills in place

The skills are plain Markdown. If you're porting to a different stack permanently, just **open the three `SKILL.md` files + the 11 `.template.md` files** and replace paths / commands / conventions wholesale. There's nothing dynamic — you're editing a prompt.

### Option 2: override via `CLAUDE.md`

The skills defer to your project's `CLAUDE.md` for most conventions. Override there and the skills pick it up. Minimum `CLAUDE.md` content to make the skills behave correctly:

```markdown
**PROJECT_STAGE:** <stage-name> (last reviewed YYYY-MM-DD)

## Tech Stack
<one paragraph: languages, frameworks, test runner, DB, frontend>

## Essential Commands
<the exact commands for: test, lint, typecheck, db migrate, db seed, start-dev>

## Architecture
<service layout — where handlers live, where UI lives, where migrations live>

## Critical Rules
<tenancy column name, validation library, import extension convention,
permission-registration file path, any banned patterns>
```

When the skills reference "CLAUDE.md conventions," they mean this. If a convention isn't in your `CLAUDE.md`, sub-agents will use the ones in the skill's default language, which may not match your stack.

### Option 3: fork per stack

If you're maintaining multiple projects on different stacks, fork this repo per stack and keep them in sync manually for skill-structure changes. A stack-per-branch scheme works too.

---

## Specific porting notes

### Python (Django, FastAPI, Flask)

- Replace `pnpm` commands with `uv` / `poetry` / `pip` equivalents.
- Replace Zod with pydantic / marshmallow; the "validation at the top of the handler" rule maps cleanly.
- Raw `pg` → SQLAlchemy Core or `psycopg` raw. The ORM-vs-raw question is stylistic; the skills' constraints (explicit `WHERE org_id = $1`) work with either.
- Migration patterns: Alembic works fine. The `.down.sql` convention becomes Alembic's `downgrade()`.
- Testing: pytest + httpx replaces Vitest + Supertest. Playwright still works.

### Go

- No ESM/TS story. Drop all import-extension rules.
- `go test ./...` replaces `pnpm test`. `golangci-lint run` replaces `pnpm lint`.
- Validation at handler top is still the rule — use `go-playground/validator` or hand-rolled.
- Migrations via `goose` / `migrate`. `.down.sql` paired migrations are natural here.
- Testing: standard library + `testcontainers-go` for integration.

### Rails

- Different story — Rails' conventions (ActiveRecord, RSpec, migrations) are tight enough that you may want to genuinely rewrite the templates rather than paste over them.
- The resolution-loop reasoning, atomic-cycle-commit rule, and project-stage calibration **still apply**. They're Rails-agnostic.

### Next.js / React Router frameworks

- `apps/web/src/` maps to `app/` or `src/app/` in Next.js.
- The 7-state coverage per screen (loading / empty / populated / error / etc.) is framework-agnostic.
- Server components + RSC change how `03-UI-SPEC.md` decomposes data fetching — the template may need a column for "server-rendered vs client-fetched."

---

## What NOT to change

These are the load-bearing ideas. Changing them turns the skills into something else:

- **The 11-doc taxonomy.** Persona / Journey / UI / API / Data / Security / Observability / AI / Seed / Tests / Index. Adding an `n+1`th doc or dropping one breaks the validator orchestrator.
- **The phase structure.** Intake → Foundation (serial) → Contracts (parallel + reconcile) → Supporting (parallel) → Index + Validate → Build. This is the spine.
- **Atomic cycle commit.** Mid-cycle commits break the reviewer's mental model and create bisection confusion.
- **Blocker verification via Status + Resolution.** Don't hardcode a blocker-ID prefix; the shape-based discovery is what generalizes.
- **Sub-agent-only writes.** The main thread orchestrates; sub-agents absorb template reads. This is what makes the skills survive `/clear`.

---

## Getting help

If you hit an awkward edge — say, the ID reconciliation step in Phase 3 doesn't match your framework's routing conventions — open an issue. The skills are a work in progress and have a lot of embedded opinions that deserve to be questioned.
