# /slice-build — Implement a Completed Slice

You are executing **Phase 6** of the Slice Builder workflow: turning the 11 slice docs + supporting artifacts into working code, tests, and observability for a slice whose `PROGRESS.md` is `status: completed`. Phases 1-5 (design + validation) produced the contract; `/slice-build` enforces it in code.

> **Prerequisite.** The slice MUST have completed `/slice` (and any `/slice-resume` passes). If `PROGRESS.md` is `draft | in-progress | abandoned`, stop and tell the user to run `/slice` or `/slice-resume` first. Do not build from a slice whose contract is still shifting.
>
> **Sub-agent pattern.** Same as `/slice`: the main thread orchestrates; all file writes (migrations, handlers, components, tests) go through spawned `Agent` calls. The main thread reads only what it needs to orchestrate (PROGRESS, 00, 02, 04, 05, BUILD-PROGRESS, CLAUDE). Agents re-read whatever slice doc(s) they need for their scoped task.
>
> **Checkpoint-aware.** Every sub-phase writes `BUILD-PROGRESS.md` so the build survives any number of `/clear` cycles. Re-entering `/slice-build <SLICE_ID>` with an in-progress build auto-resumes from `current_subphase`. No separate `/slice-build-resume` skill — this one is re-entry-safe.

## Efficiency Principle

Do what's necessary. Skip what's not. Never compromise quality.

- **Don't re-read** files that haven't changed since the last read. `BUILD-PROGRESS.md` (current sub-phase, verdicts, files written) and `PROGRESS.md` are authoritative for state — trust them unless there's concrete reason to doubt.
- **Don't re-run** Step 2.5 verifiers, sub-phase agents, or reconcile passes whose output is still current. If the working tree is unchanged since the last verified run, the verdict is unchanged. Re-dispatching "to be thorough" burns tokens and wall time.
- **Don't regenerate** sub-phase work marked `[x]` in `BUILD-PROGRESS.md`. Resume at `current_subphase`, not the first unchecked line.
- **Don't route to `/slice-resume`** for a mechanical patch when a surgical in-place fix exists. Contract re-validation across 11 docs is expensive; reserve it for actual design changes. A 5-edit fix + targeted re-verification beats a full skill re-run.
- **Don't add** scope the task didn't ask for. No defensive re-validation, no "while we're here" cleanups, no extra agent dispatches.
- **Continue a live agent via `SendMessage`** when its prior context (DB setup, reads, migration knowledge) is still relevant. Fresh `Agent()` throws away warm context.
- **Fast is a feature.** Unnecessary re-work burns context, tokens, and the user's time.

When a skill's default flow would re-do work already verified, override it with the surgical path and document the deviation in `BUILD-PROGRESS.md § Session History`. **Quality gates stay in place** — the deviation is about _scope_, not _standards_. A surgical patch still runs Step 2.5 verification on the patched artifacts; it just doesn't re-verify everything else.

## Step 0 — Load the contract

Main thread reads ONLY these (keep context for orchestration):

1. `docs/slices/<SLICE_ID>/PROGRESS.md` — verify `status: completed`. Capture `validator_verdict` (expect PASS or WARN) AND `slice_stage:` frontmatter override (if present).
2. `docs/slices/<SLICE_ID>/00-SLICE-INDEX.md` — cross-reference matrix, DoD checklist, Blockers (D1..Dn), Contradictions (C1..Cn), Risks, Rollout plan.
3. `docs/slices/<SLICE_ID>/02-JOURNEY.md` — golden path + §3a Resolution Loop (authoritative list of state-changing writes).
4. `docs/slices/<SLICE_ID>/04-API-CONTRACT.md` — endpoint IDs, Zod schemas, permission codes, error taxonomy.
5. `docs/slices/<SLICE_ID>/05-DATA-MODEL.md` — tables, state machines, migration file names.
6. `docs/slices/<SLICE_ID>/BUILD-PROGRESS.md` if it exists (resume path).
7. `CLAUDE.md`, `.claude/CLAUDE.md` — conventions (ESM `.js`, `org_id` tenancy, raw `pg`, Zod, fail-closed gateway, `.meta.md` files) AND the `PROJECT_STAGE:` declaration.

## Step 0.25 — Load PROJECT_STAGE

Resolve the active stage: `slice_stage:` from PROGRESS.md frontmatter (if present) overrides the project-level `PROJECT_STAGE:` in CLAUDE.md. Read `.claude/project-stages.md` to map stage name → six knob values: `migration_mutability`, `migration_reversibility`, `schema_mirror_sync`, `data_migration_strategy`, `rollout_safety`, `observability_floor`.

Unknown stage name, missing declaration, or absent knob → stop and ask (never guess). `last reviewed` date > 90 days → one-line advisory, no block.

Emit this banner before Step 1:

```
Project stage: <name> (ordinal <n>; last reviewed <date>)
  migration_mutability:     <value>
  migration_reversibility:  <value>
  schema_mirror_sync:       <value>
  data_migration_strategy:  <value>
  rollout_safety:           <value>
  observability_floor:      <value>
```

Gates annotated `[conditional: knob=value]` in Step 2, Step 2.5, and sub-phases 6.1/6.3/6.4 are evaluated against this profile. Non-matching gates print `⊘ skipped (stage=<name>, <knob>=<actual>)` inline and do not block. `[absolute]` gates always run.

Do NOT read on main thread — agents handle their own loads:
- Slice docs 01, 03, 06, 07, 08, 09, 10
- `openapi.yaml`, `assertions.sql`, `features/*.feature`
- `agent_docs/*` (agent-specific)

## Step 1 — Parse SLICE_ID

Arguments after `/slice-build` are SLICE_ID.

- **Explicit SLICE_ID given** → use it; stop with error if dir missing.
- **Empty args** → glob `docs/slices/*/PROGRESS.md`, filter to `status: completed`. Auto-select if exactly one; `AskUserQuestion` if multiple; stop with "nothing to build" if zero.
- **Slice `status: in-progress`** → stop. Tell user to run `/slice-resume <SLICE_ID>` first.
- **Slice `status: abandoned`** → stop. Slice was abandoned; user must rerun `/slice` with narrower scope.
- **Slice `status: draft`** → stop. Slice never finished Phase 5.
- **No PROGRESS.md** → stop. Not a slice.

Announce SLICE_ID + fresh-or-resume + validator verdict.

## Step 2 — Preflight gates (fail-closed)

The slice contract is only trustworthy if Phase 5 closed cleanly AND supporting artifacts actually exist. Block the build if any gate fails. Preflight is stricter than Phase 5 validator — WARN is acceptable for design, but open blockers are not acceptable to build.

| Gate | Check | On fail |
|---|---|---|
| Validator | `PROGRESS.md validator_verdict` in `{PASS, WARN}` | Stop: "Validator never reached PASS/WARN. Run `/slice-resume` to finish Phase 5." |
| Blocker ledgers (shape) | Every ledger section in `00-SLICE-INDEX.md` (any H2/H3 whose heading matches "Blocker", "Contradiction", "Gap", "Open Issue", "Pre-Build TODO", or "Risk (blocking)") has well-formed items: each row has an ID, a `Status` ∈ {`OPEN`, `RESOLVED`, `WAIVED`, `DEFERRED`}, and a `Resolution:` pointer for anything not `OPEN`. Zero items in `OPEN`. | Stop; list `OPEN` items and any items missing Status/Resolution. |
| Blocker resolution verification | Step 2.5 — every non-`OPEN` item's linked artifact(s) verified live (file exists, migration round-trips, test passes, grep hit, etc.). **Markers are never trusted without running the verifier.** | Stop; print the item ID + the specific verifier that failed + one-line fix pointer. |
| Migrations exist `[absolute]` | Every migration cited in `05-DATA-MODEL.md` exists under `packages/database/postgresql/migrations/` | Stop, list missing files. |
| `.down.sql` exists `[conditional: migration_reversibility!=not-required]` | Each forward migration has a paired `.down.sql` | Stop under `recommended` / `required-tested`; print `⊘ skipped (migration_reversibility=not-required)` otherwise. |
| Docker init synced | `infrastructure/docker/postgres/init/02-schema.sql` contains the new tables from 05 (grep table names) | Stop, cite `CLAUDE.md § Common Mistakes` schema-sync rule. |
| Seed module | `packages/database/src/cli/seed-slices/<slug>.ts` exists AND `packages/database/src/cli/seed-postgres.ts` wires it behind `--slice=<slug>` | Stop. |
| Gateway routes | Every endpoint ID in 04 has a row in `apps/api-gateway/src/data/route-permissions.data.ts` (fail-closed — missing = 403) | Stop, list missing rows. |
| OpenAPI fragment | `docs/slices/<SLICE_ID>/openapi.yaml` exists AND parses as YAML | Stop. |
| Gherkin features | `docs/slices/<SLICE_ID>/features/*.feature` ≥ 1 file AND non-empty | Stop. |
| Assertions | `docs/slices/<SLICE_ID>/assertions.sql` exists AND non-empty | Stop. |
| Git clean | `git status --porcelain` empty OR only slice-doc-adjacent files | Warn user; require confirmation to proceed on dirty tree. |

**Override path:** if the user insists on building with specific unresolved items (e.g. an infra blocker that doesn't touch this slice's code), the override is **per-item**, not blanket — see Step 2.5.5. Bare `--force-blockers` is rejected.

## Step 2.5 — Blocker Resolution Verification (generic, fail-closed)

Step 2 confirmed ledgers have zero `OPEN` items. This step verifies every non-`OPEN` item actually landed in code/infra — the status marker alone is never sufficient. This is generic across any blocker taxonomy (`D*`, `C*`, `GAP-*`, `BL-*`, `RISK-*`, or free-form slugs); it keys on the ledger row's **Status + Resolution pointer**, not on the ID prefix.

### 2.5.1 — Discover ledgers

Scan `00-SLICE-INDEX.md` for H2/H3 headings (case-insensitive) matching any of:

- `blocker` / `blockers`
- `contradiction` / `contradictions`
- `gap` / `gaps`
- `open issue` / `open issues`
- `pre-build` / `pre-build todo`
- `known issue` / `known issues`
- `risk (blocking)` / `blocking risk`

Optional frontmatter override (authoritative if present):

```yaml
blocker_sections: ["§8 Blockers", "§9 Contradictions Ledger", "§11 Pre-Build Gaps"]
```

Each discovered section must expose a table or bullet list where every row has: a unique **ID** (any format), a **Status** field ∈ {`OPEN`, `RESOLVED`, `WAIVED`, `DEFERRED`}, and a **Resolution** pointer (for non-`OPEN` rows). Rows missing Status or Resolution fail Step 2's shape gate — this step only runs against well-formed ledgers.

### 2.5.2 — Per-item verification contract

| Status | Required verification |
|---|---|
| `RESOLVED` | Every artifact cited in the `Resolution:` pointer must exist AND pass its type-specific verifier (2.5.3). All tokens in the pointer are AND-conjoined — any failure fails the item. |
| `WAIVED` | Pointer must cite a waiver-rationale artifact (ADR, commit SHA, `§Risks` cross-link, or an explicitly written justification paragraph). Artifact must exist and be readable. Item is logged to `BUILD-PROGRESS.md skipped_gates` with the rationale. |
| `DEFERRED` | Pointer must cite a concrete follow-up tracker (ticket ID, roadmap item, next-slice ID, or a TODO line in code). Tracker must exist. Item is logged to `BUILD-PROGRESS.md deferred_blockers` and surfaced in the completion summary. |
| `OPEN` | Should never reach this step — caught by Step 2 shape gate. If seen here, hard stop. |

### 2.5.3 — Pointer token types and verifiers

A `Resolution:` pointer is free-form prose; the verifier extracts recognizable tokens and runs the matching check. A pointer with zero recognizable tokens fails by default (prose-only resolution is insufficient — demand a concrete artifact).

| Token pattern | Verifier | Fail condition |
|---|---|---|
| `packages/**/migrations/<nnn>_*.sql` | File exists; paired `.down.sql` exists; if the SQL contains `CREATE INDEX CONCURRENTLY`, no `BEGIN;`/`COMMIT;` wrapper is present. Round-trip apply/down/re-apply on an ephemeral DB if one is reachable. | Any missing file, malformed transaction wrapper, or non-reversible round-trip. |
| `packages/database/postgresql/schema.sql` **and** `infrastructure/docker/postgres/init/02-schema.sql` (dual-sync) | Both files grep-match the cited table/column name. | Either file missing the symbol — cite `CLAUDE.md § Common Mistakes` schema-sync rule. |
| `apps/**/*.ts(:L<line>)?` or `packages/**/*.ts(:L<line>)?` | File exists; grep for the cited identifier (function, symbol, or quoted string). Line number is a hint only — identifier must exist anywhere in file. | Identifier not found. |
| `apps/api-gateway/src/config/route-permissions.data.ts` (with endpoint path/method) | Matching endpoint row exists with the expected permission string. | Missing row or mismatched permission. |
| `packages/shared-types/src/permissions.ts` (with `module:resource:action` string) | Permission string appears in both the registry AND at least one role preset. | Missing from either. |
| `docs/runbooks/*.md` | File exists; non-empty; has at least one `## ` section. | Missing / empty / malformed. |
| `infrastructure/**/dashboards/*.json` or Grafana dashboard path | `jq empty <file>` passes. | Invalid JSON. |
| Test pointer: `<path>::<test name>` or `pnpm test -- -t "<name>"` | Run the scoped test; must pass. | Non-zero exit. |
| `packages/database/src/cli/seed-slices/*.ts` | File exists AND `seed-postgres.ts` wires `--slice=<slug>` dispatch. | Missing file or missing CLI branch. |
| Prose-only pointer (no code path / no file) | Always fails — demand a concrete artifact. | N/A |

**Note on the project's conventions:** the actual gateway permissions file is `apps/api-gateway/src/config/route-permissions.data.ts` (not `src/data/…`). If future conventions shift these paths, update this table — never paper over with path fallbacks in the verifier.

### 2.5.4 — Execution model

Spawn **one** verification `Agent` with the slice dir + discovered ledger sections as input. The agent:

1. Parses every ledger section per 2.5.1 into a flat list of rows `{id, status, resolution_pointer}`.
2. Runs all verifiers per 2.5.3 — in parallel where tool calls are independent.
3. Returns a structured report:
   ```
   | ID | Status | Pointer | Verifier | Verdict | Evidence |
   |----|--------|---------|----------|---------|----------|
   ```
   Verdict ∈ {`PASS`, `FAIL`, `WAIVED`, `DEFERRED`}.

Main thread behavior:

- **0 FAILs** → append the report to `BUILD-PROGRESS.md § Blocker Verification Report` with a timestamp, then proceed to Step 3.
- **≥1 FAIL** → stop the build. Print the failing rows + the specific verifier output + a one-line fix pointer. Offer three paths:
  1. **Contract change** — route the fix back through `/slice-resume` (the Resolution pointer was wrong in the design).
  2. **Artifact gap** — if the gap is purely a missing artifact (e.g. a runbook file that should exist per the pointer but doesn't), fix it in a side commit on the build branch (not the slice docs), then rerun `/slice-build`.
  3. **Per-item override** — go to 2.5.5 if the user decides to ship anyway.

### 2.5.5 — `--force-blockers` (per-item, not blanket)

Bare `--force-blockers` is rejected. The flag takes an explicit comma-separated list of item IDs:

```
/slice-build <SLICE_ID> --force-blockers=<id1>,<id2>,...
```

Rules:

- Only the listed IDs skip verification.
- Each skipped item is copied verbatim into `BUILD-PROGRESS.md skipped_gates` together with the verifier's failure output and an optional justification the user must provide at the prompt.
- All unlisted ledger items still gate — one skipped ID does not open the floodgates.
- Skipped items are surfaced prominently in the Step 6 completion summary and in the suggested PR description so the risk carries forward.
- Listing an ID that isn't in any ledger section is an error — prevents typos from silently bypassing gates.

## Step 3 — Create or load BUILD-PROGRESS.md

File: `docs/slices/<SLICE_ID>/BUILD-PROGRESS.md`. Schema:

```yaml
---
slice_id: <SLICE_ID>
status: in-progress | completed | abandoned
started_at: <ISO>
last_updated: <ISO>
completed_at: <ISO or empty>
current_phase: 6
current_subphase: "6.1-data" | "6.2-api" | "6.3-events" | "6.4-ai" | "6.5-ui" | "6.6-tests" | "6.7-verify" | "completed"
session_count: <n>
force_blockers_ids: []     # per-item override list — populated by --force-blockers=<id,...>
skipped_gates: []          # one entry per force_blockers_ids item: {id, verifier_failure, justification}
deferred_blockers: []      # one entry per DEFERRED ledger row: {id, follow_up_tracker}
blocker_verification_ran_at: <ISO or empty>
blocker_verification_verdict: "" | "PASS" | "PASS_WITH_OVERRIDES" | "FAIL"
branch: <branch name>
commit_count: <n>
head_sha: <sha>
---

# BUILD-PROGRESS — <SLICE_ID>

## Session History
- <ISO> session 1: started at 6.1-data
- <ISO> session 2: resumed at 6.3-events

## Sub-phase Checklist
- [ ] 6.1 Data Foundation
  - [ ] migrations forward
  - [ ] migrations down verified
  - [ ] seed slice runs clean
  - [ ] assertions.sql passes
- [ ] 6.2 API Layer (N endpoints)
- [ ] 6.3 Events & Observability
- [ ] 6.4 AI Surface               # mark [n/a] if 08 status: n/a
- [ ] 6.5 UI Layer (M screens)
- [ ] 6.6 Test Harness Wiring
- [ ] 6.7 Verification

## Blocker Verification Report
<populated by Step 2.5; one row per ledger item; rewritten on every resume>

| Ledger ID | Status | Pointer (abridged) | Verifier | Verdict | Evidence / Failure |
|-----------|--------|---------------------|----------|---------|---------------------|
|           |        |                     |          |         |                     |

Verdict key:
- `PASS` — pointer artifacts verified live
- `WAIVED` — rationale artifact present; logged in skipped_gates
- `DEFERRED` — follow-up tracker present; logged in deferred_blockers
- `FAIL` — verifier failed; must be `--force-blockers=<id>` overridden or contract re-opened via `/slice-resume`

## Files Written
<path>              <sub-phase>       <commit-sha>

## Stop Reason
<populated only when an agent halts mid-phase or preflight fails>
```

- **Fresh build**: write the file with all checkboxes empty. Record the current git branch (result of `git branch --show-current`) in the `branch:` frontmatter — do NOT claim a branch that isn't cut. If the user wants a dedicated `build/<SLICE_ID>` branch, they'll say so; default is to operate on the branch the session is already on and branch-at-commit-time during 6.7 step 8.
- **Resume**: increment `session_count`, append session-history line, jump to `current_subphase`. Do NOT regenerate work already marked `[x]`.

## Step 4 — Phase 6 execution

Each sub-phase dispatches work via `Agent` calls. **Main thread never writes implementation code.** Use the same sub-agent prompt discipline as `/slice`: give the agent exact paths, locked decisions, and the acceptance criterion.

### 6.1 — Data Foundation (sequential, single agent)

Spawn **one** Agent (backend-architect or general-purpose) with scope:

1. Read `05-DATA-MODEL.md`, the migration files, `assertions.sql`, and the seed module.
2. Apply migrations forward: `pnpm db:migrate:pg`.
3. `[conditional: migration_reversibility=required-tested]` Verify `.down.sql` reversibility: apply → rollback → re-apply on throwaway DB (or tx-wrapped). Under `not-required` / `recommended`, print `⊘ skipped (migration_reversibility=<value>)` and proceed.
4. Run `pnpm db:seed:pg --reset --slice=<SLICE_ID>`.
5. Run `psql $DATABASE_URL -f docs/slices/<SLICE_ID>/assertions.sql` — every assertion returns the expected count.
6. **Do NOT commit.** Stage changes only if needed for your own diff reasoning. Per the cycle-commit rule (see /slice SKILL.md), the working tree accumulates until end-of-cycle.
7. Update BUILD-PROGRESS.md 6.1 checkboxes (no commit SHA — there is no commit mid-cycle).

**Failure:** assertion fails → agent reports which one + DB state. Main thread routes to `/debug-loop` or pauses for user. No auto-retry.

### 6.2 — API Layer (parallel per endpoint + reconcile)

Spawn **N parallel Agents** in ONE message — one per endpoint in `04-API-CONTRACT.md`. Each agent's scope is a single endpoint:

- Handler file under the owner service (follow `apps/services/<svc>/src/handlers/` convention)
- Zod validation inline, `org_id` injected via `getRequiredOrgId(req)`
- Route registration in `routes.ts`
- Permission row in `apps/api-gateway/src/data/route-permissions.data.ts` — **append only, with `// <EP-ID>` marker comment, do NOT reorder other rows**
- Unit tests (`*.test.ts`): happy path + validation + permission denied + tenancy isolation
- Integration tests via `getTestApp()` factory: golden path + every error-code row from 04 § Error Taxonomy
- Co-located `.meta.md` per `.claude/CLAUDE.md` rules
- Appends to `openapi.yaml` fragment if inline docs drift

After all N agents return, spawn a **reconcile Agent**:
- Dedupe / sort `route-permissions.data.ts` rows
- Run `pnpm lint --fix`
- Run `pnpm test:spec:api` and `pnpm test:spec:perm`
- **Do NOT commit the reconcile.** Leave the unified diff on the working tree.

Main thread: block on test failures → `/debug-loop`.


### 6.3 — Events & Observability (parallel, 2-3 agents)

Spawn parallel Agents in ONE message:

- **Events agent** — for each domain event in 02-JOURNEY + 07-OBSERVABILITY: publisher call in handlers (or audit writer), subscriber in downstream services, registration in shared-types event catalog.
- **Metrics/alerts agent** — scope scales with `observability_floor`: `logs-only` = structured log events per error code only (skip metrics/dashboard/alerts agents); `metrics+runbooks` = + Prometheus metrics (cardinality-safe) + per-error-code runbook pointers; `slo+alerts+dashboards` = + Grafana dashboard JSON at the exact path cited in 07 + alerts wired with runbook links; `full-audit-trail` = + audit-table retention enforcement.
- **Audit writer agent** — if the slice writes to a domain audit table (per 06 + 05), implement the writer and wire into every state-changing handler (reference Resolution Loop `audit_event` fields in 02 §3a).

Reconcile pass (main thread): `pnpm test:spec`, then grep for orphaned metric names (in code but not in dashboard JSON) and orphaned log codes.

**No commits.** Each agent ends with `git status` clean-ish on unrelated paths and its own paths staged-in-memory / on working tree only.

### 6.4 — AI Surface (conditional)

**If** `08-AI-SURFACE.md status: ready` — spawn one Agent with scope:
- Prompt templates committed with version tags
- Eval harness + dataset from 08
- Cost tracking + fallback path from 08
- Feature flag wiring (prompt behind flag)

**If** `08 status: n/a` — mark BUILD-PROGRESS 6.4 `[n/a]` and skip. Note in BUILD-PROGRESS.md stop-reason or session history: "AI surface: n/a per slice contract".

### 6.5 — UI Layer (parallel per screen + reconcile)

Spawn **M parallel Agents** in ONE message — one per screen in `03-UI-SPEC.md`. Each agent's scope is a single screen:

- Component tree under `apps/web/src/` matching existing conventions
- Zustand slice or TanStack Query hook per data dependency
- API client wrapper using the response envelope (`{ data }` / `{ data, meta }`)
- Tests: component + hook + MSW-mocked integration
- Storybook stories covering the 7 canonical states (initial / loading / empty / populated / error / partial-stale / permission-denied) as enumerated in 03 §2.x state matrices
- Copy deck strings per 03 (i18n bundle if project uses one, else inline constants)
- Motion tokens per 03 §motion using `src/lib/motion.ts` conventions if present
- Router wiring — **append only with `// <SCR-ID>` marker, do NOT reorder**
- Co-located `.meta.md` files

After all M agents return, spawn a **UI reconcile Agent**:
- Dedupe / order router entries
- Run `pnpm lint --fix`
- `pnpm --filter web test`
- `pnpm --filter web typecheck`
- **Do NOT commit.** Leave the UI reconcile diff on the working tree.

Then invoke the `/screenshot-test` skill for visual verification against 03's copy deck + state matrices.


### 6.6 — Test Harness Wiring (single agent, sequential)

Spawn one Agent:
- Bind each Gherkin step in `features/*.feature` to Playwright step definitions under `apps/web/tests/e2e/<SLICE_ID>/`
- Scaffold chaos scenarios from 10 § Chaos into existing chaos harness (or `scripts/chaos/`)
- Axe-core a11y sweep hooked into Playwright per 10 § A11y
- Tag slice-specific tests with `@<SLICE_ID>` so `pnpm test:e2e --grep @<SLICE_ID>` isolates this slice
- **Do NOT commit.** Leave the test-harness diff on the working tree.

Main thread: run `pnpm test:e2e --grep @<SLICE_ID>`. Block on failure → `/debug-loop`.

### 6.7 — Verification (sequential, goal-backward)

Final gate. Nothing marks completed until all seven steps pass.

1. **Full test sweep** (main thread): `pnpm test` + `pnpm test:spec` + `pnpm test:e2e --grep @<SLICE_ID>` + `pnpm typecheck` + `pnpm lint`. All green.
2. **Assertions replay** (main thread): re-run `assertions.sql` post-E2E to confirm seed state remains consistent.
3. **Golden-path smoke** (agent): spawn an agent that reads 02 § Golden Path + §3a Resolution Loop and walks each step via curl/playwright with real seed data. Returns PASS/FAIL per step.
4. **Screenshot review** (user-mediated): invoke `/screenshot-test`; user signs off on visuals.
5. **Fresh-context code review** (agent): spawn a fresh `code-reviewer` Agent (no session history) with the slice docs as the spec + the full diff. Reviewer returns spec-compliance verdict per Definition of Done.
6. **Acceptance criteria audit** (main thread): open `00-SLICE-INDEX.md § DoD`. Flip `[ ]` → `[x]` only for items now demonstrably true.
7. **Meta-file audit** (main thread): every new/modified source file has a current `.meta.md` per `.claude/CLAUDE.md`. Grep for orphans.

Any step 1-5 fails → `/debug-loop` on the specific failure. Do not mark built until all 7 pass.

8. **End-of-cycle commit prompt (user-gated).** Only after steps 1-7 pass AND user types "commit" / "ship it": create the commit(s) on `build/<SLICE_ID>` branch. Default: one commit per sub-phase is fine if each sub-phase's diff is logically coherent; otherwise one atomic commit. Do NOT push. Wait for explicit "push" or "open PR" before `git push` / `gh pr create`. This step is NEVER auto-advanced.

## Step 5 — Phase Markers

After each sub-phase boundary, emit a single-line marker (`**Sub-phase 6.N complete. Proceeding to 6.N+1.**`) and advance immediately to the next sub-phase. Do NOT pause for user input, do NOT emit checkpoint blocks, do NOT suggest `/clear`. Context management is Claude Code's responsibility via auto-compact — not this skill's.

The `/slice → /slice-resume → /slice-build` cycle is one seamless continuous operation. All user decisions are gathered upfront in `/slice` Phase 1 (Batch A + Batch B intake). If `/slice-build` encounters ambiguity mid-cycle, that is a slice-contract defect — route to `/slice-resume` to reopen the design, never pause the main thread for a mid-cycle decision.

BUILD-PROGRESS.md is the durable checkpoint. If auto-compact fires or the session crashes, the user re-invokes `/slice-build <SLICE_ID>` and the skill resumes at `current_subphase` with no interaction required.

## Step 6 — Completion

When 6.7 all-pass:

1. Set `BUILD-PROGRESS.md status: completed`, `completed_at: <ISO>`, `current_subphase: completed`. **Do NOT commit BUILD-PROGRESS.md yet — it rides the final commit with the rest of the cycle.**
2. Update `PROGRESS.md` Session History with one line: `<ISO> session <n>: /slice-build completed → <branch>@<head-sha>`. Do NOT change `PROGRESS.md status` — slice **design** stays `completed`; the build is a separate artifact.
3. Emit completion summary:

```
SLICE <SLICE_ID> — BUILT

Branch:           <branch> (pre-commit — ask user to approve the end-of-cycle commit)
Uncommitted files: <n> (staged on working tree)
Total diffs:      <n> (one per sub-phase, mentally grouped for the commit-message draft)
Files written:    <n>
Tests:            <unit> unit • <spec> spec • <e2e> e2e  (all green)
Assertions:       <n>/<n> pass
Meta files:       <new> new, <updated> updated
Context used:     <approx %>
Elapsed:          <h>h <m>m (session count: <n>)

Rollout (from 00 § Rollout):
  1. <flag name> → dogfooding cohort
  2. <flag name> → 10% → 50% → 100% over <duration>
  3. Soak dashboard: <path from 07>

Observability:
  Grafana:      <dashboard path from 07>
  Alerts wired: <count> (runbooks: <path>)
  Events:       <count> domain events published

Skipped gates (if --force-blockers was used):
  - <list, or "none">

Next steps:
  1. Open PR against main; paste BUILD-PROGRESS.md as PR description
  2. Stakeholder demo per 00 § Demo script
  3. Soak per rollout plan above
  4. Close corresponding tickets / update roadmap
```

## Failure Protocol

| Situation | Response |
|---|---|
| Preflight gate fails | Stop. Report the specific gate + one-line fix pointer. Do NOT write BUILD-PROGRESS.md. |
| Migration fails forward | Rollback via `.down.sql`. Stop. Report DB state snapshot. No re-run. |
| Migration forward OK but `.down.sql` broken | STOP — reversibility is a hard gate. Agent rewrites `.down.sql` before build continues. |
| Agent stuck in sub-phase | Read agent return, log to BUILD-PROGRESS Stop Reason, pause for user (retry / skip / escalate). |
| Test failure in 6.1-6.6 | `/debug-loop` on the failing test. No commits exist to push — just iterate on the working tree until green. |
| Test failure in 6.7 (regression) | Stop. Indicates cross-phase interaction the slice didn't anticipate. Surface to user; may require re-opening the contract via `/slice-resume`. |
| User wants to build with open blockers | Require explicit `--force-blockers`. Log in BUILD-PROGRESS `force_blockers: true` + `skipped_gates: [...]`. Surface in final summary. |
| Slice contract wrong (doc says X, code can't do X) | Stop. Do NOT edit slice docs. Require user to re-open design via `/slice-resume`, then re-run `/slice-build`. |

## Never

- **Never** commit ANY file during `/slice`, `/slice-resume`, or `/slice-build` sub-phases. The cycle commits atomically only after 6.7 verification + user approval. See the absolute rule in `.claude/skills/slice/SKILL.md`.
- **Never** run `git commit --no-verify`. Pre-commit hooks enforce the same quality bars this skill depends on.
- **Never** use `--no-gpg-sign`, `--no-edit`, or bypass author-email checks. Commit author and trailer policy follow your project's `CLAUDE.md`.
- **Never** write implementation code from the main thread. Always spawn an Agent.
- **Never** edit slice docs (00-10) during `/slice-build`. Those are the frozen contract. If a doc is wrong, stop and tell the user to re-open via `/slice-resume`.
- **Never** mark a sub-phase `[x]` until its success criteria pass (tests green, assertions pass, reconcile clean).
- **Never** force-push. Never `reset --hard`. Never delete branches.
- **Never** violate the "one agent writes a given file" invariant in parallel fan-outs. Use append-with-marker + reconcile agent, not N concurrent writers on the same file.
- **Never** skip preflight gates silently. Either pass them or require explicit per-item `--force-blockers=<id,...>` (bare `--force-blockers` is rejected).
- **Never** trust a `RESOLVED` / `WAIVED` / `DEFERRED` ledger marker without running the Step 2.5 verifier against its pointer. Design-time annotations rot; only re-verification proves the artifact is still in the tree.
- **Never** hardcode a blocker-ID prefix (e.g. `D*`, `C*`) anywhere in the skill or a sub-agent prompt. Ledger discovery is heading-based + Status/Resolution-shape-based so it generalizes to any slice's taxonomy.
- **Never** regenerate work already marked `[x]` on resume — respect BUILD-PROGRESS as the source of truth.

## Budgets (hard limits)

- **Parallel agent fan-out:** ≤ 8 agents in one message (engine limit)
- **Test-fix loops per sub-phase:** ≤ 3 iterations before escalating to user

Context and wall-clock are NOT budgets this skill tracks. Auto-compact handles context; BUILD-PROGRESS.md makes any session resumable; the cycle runs continuously until 6.7 PASS.

## Re-entry rules (resume semantics)

On `/slice-build <SLICE_ID>` with an existing `BUILD-PROGRESS.md`:

- `status: in-progress` → resume at `current_subphase`. Increment `session_count`. Append to Session History. Re-run Step 2 preflight (blockers may have reopened). Do NOT re-dispatch completed sub-phases.
- `status: completed` → stop. Report: "Slice already built at <branch>@<head-sha> on <completed_at>. Nothing to do." Offer: (a) diff viewer pointer, (b) delete BUILD-PROGRESS.md and rebuild from scratch (requires explicit confirmation).
- `status: abandoned` → stop. Report abandonment reason from Stop Reason. Offer to rerun with fresh BUILD-PROGRESS.md (explicit confirmation).

## Reference

- Prior skills: `.claude/skills/slice/SKILL.md`, `.claude/skills/slice-resume/SKILL.md`
- Debug helper: `.claude/skills/debug-loop/SKILL.md` (invoke on test failures)
- Visual review: `.claude/skills/screenshot-test/SKILL.md` (invoke in 6.5 + 6.7 step 4)
- Code review: `.claude/skills/review/SKILL.md` (invoke in 6.7 step 5)
- Conventions: `CLAUDE.md`, `.claude/CLAUDE.md`
- Reference filled slice (design): `docs/slices/00-example-bookmark-v1/`
- Blocker-ledger shape: every slice's `00-SLICE-INDEX.md` should carry at least one ledger section whose rows have `{ID, Status ∈ {OPEN|RESOLVED|WAIVED|DEFERRED}, Resolution: <concrete pointer>}`. Slices whose ledgers use different ID prefixes (`D*`, `C*`, `GAP-*`, `BL-*`, `RISK-*`, etc.) all pass through the same Step 2.5 verifier — the skill never hardcodes a taxonomy.

ARGUMENTS: <SLICE_ID> [--force-blockers=<id1>,<id2>,...]
