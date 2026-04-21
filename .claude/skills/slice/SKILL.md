---
name: slice
description: >
  Expand a one-line feature description into a complete production-quality vertical slice —
  all 11 filled docs (persona, journey, UI, API, DB, security, observability, AI, seed, tests, index)
  plus migrations, seed TypeScript, Gherkin features, and OpenAPI fragment. Use when the user
  invokes /slice <description>, or says "build a full slice", "create a vertical slice",
  or wants a shippable persona→journey→UI→backend→DB→tests bundle from one sentence.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent
---

# /slice — One-Liner to Full Vertical Slice

You are executing the Slice Builder workflow defined in `docs/slice-templates/SLICE-BUILDER-PROMPT.md`. The user's arguments are the ONE_LINER.

> **Resumable by design.** Every phase writes a `PROGRESS.md` checkpoint to the slice directory.
> **Sub-agent driven.** Every slice doc is written by a spawned Agent, never inline on the main thread. Main thread orchestrates and summarizes; sub-agents absorb the heavy template/reference reads.
> **Continuous by default.** The skill runs all phases end-to-end in one session without pausing for user input between phases. After each phase it emits a short informational marker and proceeds immediately. Claude Code auto-compacts conversation history when context fills — routine phase-boundary resets are not the skill's job. PROGRESS.md + `/slice-resume` exist for real restart scenarios (intentional `/clear`, session crash, unrelated context swap), not for routine context management.

## Absolute Rule — Atomic Cycle Commit

**Applies to /slice, /slice-resume, and /slice-build equally.** The full cycle is one atomic unit of work. **Do NOT commit any files mid-cycle** — not per phase, not per sub-phase, not per endpoint, not per screen, not "for safety". Agents and the main thread stage nothing and commit nothing until all of these hold:

1. `/slice-build` Step 6.7 Verification passes all 7 steps.
2. `pnpm test` + `pnpm test:spec` + `pnpm test:e2e --grep @<SLICE_ID>` + `pnpm typecheck` + `pnpm lint` all green in one continuous run.
3. User explicitly says "commit" / "ship it" / equivalent.

**Rationale.** A slice is a vertical contract: migrations + API + UI + events + tests + observability interlock. Partial commit streams let reviewers see half-baked slices and create bisection confusion. One atomic commit (or a small, logically-grouped handful at cycle end) is easier to review and easier to revert.

**How state persists mid-cycle.** `PROGRESS.md` + `BUILD-PROGRESS.md` are the durable checkpoints that survive `/clear`. `git status` is the diff of "what the cycle has produced so far". Never use commits as checkpoints.

**No exceptions.** If a cycle aborts, complete it or `git stash` / `git reset` — never commit partial work.

## Efficiency Principle

Do what's necessary. Skip what's not. Never compromise quality.

- **Don't re-read** files that haven't changed since the last read. `PROGRESS.md`, `BUILD-PROGRESS.md`, and validator outputs are authoritative for state — trust them unless there's concrete reason to doubt.
- **Don't re-run** agents, validators, or sub-phases whose output is still current. If the working tree is unchanged, the verdict is unchanged. Re-dispatching "to be thorough" burns tokens and wall time.
- **Don't regenerate** work to fit a skill's default shape. Prefer surgical patches over full re-runs. A 5-edit fix beats a 20-agent regeneration.
- **Don't add** scope the task didn't ask for. No defensive re-validation, no "while we're here" cleanups.
- **Fast is a feature.** Unnecessary re-work burns context, tokens, and the user's time.

When a skill's default flow would re-do work already verified, override it with the surgical path and document the deviation in `PROGRESS.md` / `BUILD-PROGRESS.md`. **Quality gates stay in place** — the deviation is about _scope_, not _standards_.

## Execution Model (read before doing anything)

1. **Main thread** handles: Step 0 contract loading, Step 0.5 resume detection, Step 1 user Q&A (Phase 1), orchestration between phases, PROGRESS.md updates on phase boundaries, checkpoint emission.
2. **Sub-agents** (spawned via `Agent` tool) handle: all slice doc writes (01-10), OpenAPI, assertions.sql, Gherkin, migrations, seed TypeScript. Each agent reads templates + references itself and returns a ≤200-word summary.
3. **PROGRESS.md is the only durable state.** Every sub-agent's final step is to update PROGRESS.md. The main thread re-reads PROGRESS.md after each agent returns to verify the checkpoint landed.
4. **`/clear` is a user CLI command.** The main thread cannot invoke it. Instead, after each major phase the main thread prints a Context Checkpoint block the user can copy-paste: `/clear` then `/slice-resume <SLICE_ID>`.

## Step 0 — Load the contract

1. **Read** `docs/slice-templates/SLICE-BUILDER-PROMPT.md` in full. Source of truth for phases, quality bars, banned phrases, self-checks.
2. **Glob** `docs/slice-templates/*.template.md` — do NOT read all of them on the main thread. Sub-agents read the ones they need. Main thread just needs the list.
3. **Do NOT read** `docs/slices/00-example-bookmark-v1/*.md` on the main thread. Sub-agents read their corresponding reference file.
4. **Read** `CLAUDE.md` and `.claude/CLAUDE.md` for codebase conventions (ESM `.js` imports, `org_id` tenancy, fail-closed gateway, Zod, raw `pg`) AND the `PROJECT_STAGE:` declaration line.

Staying off the reference files keeps main-thread context small — agents will re-read what they need.

## Step 0.25 — Load PROJECT_STAGE

Read the `PROJECT_STAGE:` line from `CLAUDE.md` (top of file). Read `.claude/project-stages.md` and resolve the named stage to its six knob values.

**Stage resolution rules:**
- Stage name not found in `.claude/project-stages.md` → stop and ask the user to add the stage definition to that file. Do NOT guess.
- `PROJECT_STAGE:` line missing from CLAUDE.md → stop and ask the user to declare one. Do NOT default to `developing-mvp`.
- `last reviewed` date > 90 days stale → emit a one-line advisory ("Stage declaration last reviewed <date> — confirm still accurate.") but do not block.

Emit this banner once at session start (before Phase 1):

```
Project stage: <name> (ordinal <n>; last reviewed <date>)
  migration_mutability:     <value>
  migration_reversibility:  <value>
  schema_mirror_sync:       <value>
  data_migration_strategy:  <value>
  rollout_safety:           <value>
  observability_floor:      <value>
```

**How this affects /slice.** In Phase 4, Agent 4.2 (07-OBSERVABILITY.md) and Agent 4.3 (08-AI-SURFACE.md) consume the relevant knobs to calibrate their doc requirements. In Phase 5 the validator-orchestrator uses the observability_floor to set its threshold for dashboard/runbook completeness. Per-slice override via `slice_stage:` in `PROGRESS.md` frontmatter is honored.

## Step 0.5 — Check for in-progress slices

```bash
grep -lE "^status: in-progress$" docs/slices/*/PROGRESS.md 2>/dev/null
```

- **Paths returned:** announce them and ask via AskUserQuestion whether to resume (hand off to `/slice-resume <id>`) or start new.
- **Empty:** proceed to Step 1.

Never silently overwrite an in-progress slice.

## Step 1 — Parse user arguments (main thread)

Modes:
- `[AUTO] <one-liner>` → skip Phase 1 questions, flag every assumption
- `[SKELETON] <one-liner>` → fill only 00 + 01 + 02 + 09, others `status: draft`
- `<one-liner>` → interactive (default), ≤5 AskUserQuestion

If ONE_LINER empty or < 10 chars, ask for it. Don't invent.

Derive `SLICE_ID`: kebab-case, ≤40 chars, suffix `-v1`. Announce it.

## Step 2 — Create slice dir & initialize progress (main thread)

1. `mkdir -p docs/slices/<SLICE_ID>/`
2. Write initial `docs/slices/<SLICE_ID>/PROGRESS.md` (schema in § Progress Tracking Protocol). `current_phase: 1`, `next_action: "Phase 1 intake questions"`.
3. Proceed to phases.

## Phase 1 — Intake (main thread, user-facing)

Phase 1 asks in **two batches**. Both must be answered (or explicitly declined) before Phase 2 starts. The reason: Batch A classifies WHAT the slice is; Batch B uncovers the *resolution loop* — what the user DOES after consuming the surface. Without Batch B, journeys degenerate into "navigate + read + back-button" and ship view-only products that under-serve the persona's jobs.

### Batch A — Classification (≤5 questions, one AskUserQuestion call)
1. Persona archetype (Practitioner / Operator / Executive / Viewer / other)
2. Owner service (of 16)
3. AI involvement (none / read-side / generative / agentic)
4. Tenancy (`org_id` vs `tenant_id`)
5. New vs modifies existing UI

### Batch B — Resolution Loop (mandatory reasoning; ≤5 per-slice questions — **mandatory except in [AUTO]**)

Batch A said *what* the slice is. Batch B must uncover *how the persona reaches closure* on the job-to-be-done — what they do on the surface until they can stop thinking about it. Without Batch B, journeys ship as "navigate + read + back-button."

**Do NOT hand the user a fixed checklist.** Reason from three inputs, then synthesize 2–5 questions that are actually ambiguous *for this specific slice* — every option you present must be drawn from the codebase, not invented generically.

**Input 1 — Persona JTBDs.** What decision, hand-off, or state change does this persona need to make to consider themselves done? If 01-PERSONA.md is not yet written, infer from the one-liner + Batch A persona archetype.

**Input 2 — Product boundary (from `CLAUDE.md`).** The Module Boundaries table names which service owns what and — equally important — what each service must NOT touch. The slice's action surface stops at that boundary. If an action would cross it (e.g., a feature slice triggering a billing write), the action belongs to the downstream service's slice, not this one. Cite the boundary rule when you scope an action in or out.

**Input 3 — Natural closure shapes in this codebase** (reasoning aids, NOT a menu to read aloud):
- Does a **state machine** govern this domain? (Your CLAUDE.md should name the legal state transitions for each domain entity.) If yes, the legal transitions ARE the closure actions. Use those literal names.
- Is this an **approval/review** surface? Closure = accept / reject-with-reason / request-changes / comment.
- Is this an **authoring** surface? Closure = save-draft / publish / version / rollback.
- Is this an **intake/triage** surface? Closure = assign-owner / set-priority / route-to-queue.
- Is this a **monitoring/measurement** surface? Closure = acknowledge / escalate / mark-resolved / trigger-downstream-playbook.
- Is this a **reporting/sharing** surface? Closure = export / share-link / schedule-delivery / archive.

Pick the shape(s) that match this slice. Do NOT list shapes that don't apply.

**Four dimensions you must resolve for every slice** — phrase each as a per-slice question, not a generic one. Make an educated guess inline when you can.

| Dimension | What you must resolve | How to derive the options |
|---|---|---|
| Closure actions | Which concrete domain actions move this persona to "done"? | Read state machines + permissions + existing handler names in the owner service. Quote them literally (`FEATURE.PLANNED → IN_PROGRESS`, not `"change status"`). |
| Impact preview | How does the UI show consequences before commit? | Match action *severity*: idempotent writes → none + undo; irreversible state transition → confirm dialog with impact summary; multi-field edits → side-panel diff. |
| Cross-boundary routing | Which *other services* or *external integrations* must the action fan out to? | Inspect the 16-service module-boundary table. If the action emits a domain event another service subscribes to, list it; if it calls an existing integration (email, Slack, webhook), list it; if strictly local, say "none — audit log only." |
| Reversibility | What does undo look like? | Driven by whether the write is idempotent, state-machine-regressive (is the inverse transition legal?), or irreversible (audit-log-only). Don't promise an undo the state machine won't honor. |

**When you can confidently guess** (e.g., "this is an approval flow — `accept / reject / request-changes / comment` with a confirm dialog on reject, workflow-svc fan-out on accept, and undo-toast on reject only") — state the guess inline in the AskUserQuestion description and ask the user to confirm or redirect. That's higher signal than a blank menu.

**When you genuinely can't guess** (novel surface, no analog in the codebase) — ask the underlying reasoning question: *"What does 'done' mean for {{persona}} on this surface, and where does the workflow hand off?"*

**Record answers in `INTAKE.md` under `## Resolution Loop`** with this structure (reasoning lives here, not in downstream docs):

```
## Resolution Loop

### Persona's definition of "done"
<one paragraph: what state change or hand-off lets the persona close the tab and stop thinking>

### Product-boundary rationale
<one paragraph: citing CLAUDE.md § Module Boundaries — why certain actions are IN scope (this service owns them) and others are deferred (they cross into another service's territory)>

### Locked actions (each must surface in 02/03/04/05/06/10)
- ACTION-001: <domain-specific verb phrase using codebase nouns>
  - State transition: <literal machine transition, or "n/a — non-transitional write">
  - Impact preview: <shape tied to severity>
  - Downstream fan-out: <named services + events, or "none">
  - Reversibility: <mechanism tied to state-machine or audit>
- ACTION-002: ...

### Explicitly deferred (§9 Non-Goals must echo these)
- <action>: owned by slice `<id-vN>` because <boundary-citing reason>
```

Every locked action must surface across 02/03/04/05/06/10 per the coverage matrix in `02-JOURNEY.template.md §3a`. If zero actions are in scope, `02-JOURNEY.md §9 Non-Goals` MUST name the sibling slice that owns the resolution loop — otherwise the slice validator fails.

`[AUTO]` mode skips Batch B *questions* but MUST still perform the reasoning and record best-guess actions in `INTAKE.md`, flagged as assumptions. `[SKELETON]` runs Batch A only.

Write `docs/slices/<SLICE_ID>/INTAKE.md` (question/answer pairs from both batches, locked assumptions, Resolution Loop section, why-this-slice-matters paragraph, file manifest). Update `PROGRESS.md`: tick Phase 1, set `current_step: 2.1-persona`, append session history.

**Phase 1 complete. Proceed to Phase 2.**

## Phase 2 — Foundation (sub-agents, serial)

**Agent 2.1 — 01-PERSONA.md.** Spawn one `Agent` call with the template in § Sub-Agent Prompt Template. Serial (not parallel) because 2.2 references 2.1.

When it returns: verify `01-PERSONA.md` exists, re-read `PROGRESS.md` to confirm `current_step: 2.2-journey`. If not, fix manually.

**Agent 2.2 — 02-JOURNEY.md.** Spawn a second `Agent`. Prompt references the just-written persona file.

When it returns: verify `02-JOURNEY.md` exists, confirm `PROGRESS.md` advanced to `current_step: 3-contracts`.

**Phase 2 complete. Proceed to Phase 3.**

## Phase 3 — Contracts (sub-agents, parallel)

Spawn 3 `Agent` calls in a **single message** (true parallelism):
- Agent 3.1 — `03-UI-SPEC.md`
- Agent 3.2 — `04-API-CONTRACT.md` (also writes `openapi.yaml` fragment)
- Agent 3.3 — `05-DATA-MODEL.md` (also writes migration .sql + .down.sql)

Each agent uses placeholder IDs for cross-refs (e.g., `ENDPOINT-001`, `SCREEN-001`).

When all three return:
- **Spawn Agent 3.4 — ID Reconciliation.** Its job: read 03 + 04 + 05, unify ID references (every UI interaction cites a real endpoint ID; every endpoint references real tables). Rewrite the three docs with reconciled IDs. Update PROGRESS.md: tick 3.1–3.x, set `current_step: 4-supporting`.

**Phase 3 complete. Proceed to Phase 4.**

## Phase 4 — Supporting (sub-agents, parallel)

Spawn 5 `Agent` calls in a single message:
- Agent 4.1 — `06-SECURITY-TENANCY.md`
- Agent 4.2 — `07-OBSERVABILITY.md` — pass the loaded `observability_floor` knob in the prompt. `logs-only` = only structured-log events required (no dashboard/alerts/runbooks); `metrics+runbooks` = metrics catalog + per-error runbook pointers; `slo+alerts+dashboards` = full Grafana dashboard + alerts wired to runbooks; `full-audit-trail` = + audit-table retention policy section. Agent must not inflate requirements beyond the floor.
- Agent 4.3 — `08-AI-SURFACE.md` (set `status: n/a` with justification if no AI per INTAKE.md) — pass `rollout_safety`. `ship-direct` skips the feature-flag section entirely; `flagged` / `canary+flag` / `change-control` require the flag + rollout plan sections.
- Agent 4.4 — `09-SEED-DATA.md` (also writes `assertions.sql` + `packages/database/src/cli/seed-slices/<slug>.ts`) — pass `data_migration_strategy`. `reset-tolerant` seeds assume `--reset`; `backfill-tolerant` / `zero-downtime` seeds include backfill-compatible ordering.
- Agent 4.5 — `10-TEST-PLAN.md` (also writes `features/*.feature`)

When all five return: verify all files exist. Update PROGRESS.md: tick 4.1–4.5, set `current_step: 5.1-index`.

**Phase 4 complete. Proceed to Phase 5.**

## Phase 5 — Index & validation

**Agent 5.1 — 00-SLICE-INDEX.md.** Spawn an Agent that reads all 10 sibling docs and writes the index: pitch paragraph, status table, 15-item DoD checklist, 3 metrics from 07, rollback plan.

When it returns: update PROGRESS.md: tick 5.1, set `current_step: 5.2-validator`.

**Agent 5.2 — validator orchestrator.** Spawn `slice-validator-orchestrator` with the slice dir path. Expect PASS / WARN / FAIL.

- **PASS:** set `00-SLICE-INDEX.md` frontmatter `status: ready-to-build`. Update PROGRESS.md: `status: completed`, `completed_at`, `validator_verdict: PASS`. Emit final summary (Step 5).
- **WARN:** acceptable; log warnings in `00` §5, proceed to Step 5.
- **FAIL:** parse blockers. For each blocker, spawn a targeted fix-up agent ("Re-write 04-API-CONTRACT.md to fix: <blocker>"). Re-run validator. Max 3 loops. Increment `validator_loops` in PROGRESS.md after each loop.

If still FAIL after 3 loops: set PROGRESS.md `status: in-progress`, `validator_verdict: FAIL`, populate Stop Reason with stubborn blockers, and stop — report the blockers to the user. Do not fabricate PASS.

## Step 3 — Emit final summary (main thread)

On PASS:

```
SLICE <SLICE_ID> — COMPLETE

One-liner: "<original ONE_LINER>"

Files written:
  docs/slices/<SLICE_ID>/00-SLICE-INDEX.md           [ready]
  docs/slices/<SLICE_ID>/01-PERSONA.md               [ready]
  ... (11 total)
  docs/slices/<SLICE_ID>/PROGRESS.md                 [completed]
  docs/slices/<SLICE_ID>/openapi.yaml
  docs/slices/<SLICE_ID>/assertions.sql
  docs/slices/<SLICE_ID>/features/*.feature
  packages/database/src/cli/seed-slices/<slug>.ts
  packages/database/postgresql/migrations/<nnn>_<slug>.sql (+.down.sql)
  apps/api-gateway/src/data/route-permissions.data.ts (modified)

Assumptions flagged: <from Phase 1 or AUTO>
Validator verdict: PASS | WARN (<n> non-blocking)
Sessions used: <session_count from PROGRESS.md>

Next steps:
  1. Review 01-PERSONA.md and 02-JOURNEY.md.
  2. Seed: pnpm db:seed:pg --reset --slice=<SLICE_ID>
  3. Verify: psql $DATABASE_URL -f docs/slices/<SLICE_ID>/assertions.sql
  4. Implement: hand to a fresh session with "Implement the slice at docs/slices/<SLICE_ID>/."
```

## Phase Markers

After each phase, emit a one-line marker (`**Phase N complete. Proceed to Phase N+1.**`). Do not emit a long `/clear` recommendation block — Claude Code auto-compacts conversation history when context fills. The PROGRESS.md + `/slice-resume` path exists for restart scenarios (intentional `/clear`, session crash, unrelated context swap), not routine phase transitions.

On an **early stop** (validator FAIL after 3 loops, unrecoverable sub-agent BLOCKED, user-requested pause), emit a single block describing what landed, what didn't, and the resume command:

```
SLICE <SLICE_ID> — PAUSED
Last checkpoint: docs/slices/<SLICE_ID>/PROGRESS.md
Reason:          <one-line cause>
Resume with:     /slice-resume <SLICE_ID>
```

## Sub-Agent Prompt Template

Every Agent spawn for a slice doc uses this template. Fill the bracketed fields.

```
You are writing ONE slice document as part of a vertical-slice build. Main thread is orchestrating and will only see your final ≤200-word summary — do all your own reading.

# Target file
docs/slices/<SLICE_ID>/<NN>-<DOC-NAME>.md

# Read these first (in this order)
1. docs/slices/<SLICE_ID>/PROGRESS.md       — current state, sibling doc checklist
2. docs/slices/<SLICE_ID>/INTAKE.md         — Phase 1 decisions (LOCKED — do not re-decide)
3. docs/slice-templates/<NN>-<slug>.template.md  — structure you must match
4. docs/slices/00-example-bookmark-v1/<NN>-<slug>.md  — fidelity reference (match or exceed)
5. <any prior sibling docs already written, listed in PROGRESS.md Files Written>
6. <domain-specific reads: CLAUDE.md sections, existing code paths from INTAKE.md>

# Quality bar (non-negotiable)
- Match template structure exactly; fill every [REQUIRED] section
- No banned phrases: drives growth|seamless|powerful|best-in-class|streamlined|cutting-edge|robust|intuitive|game-changing|synergy|leverage|next-gen|world-class|enterprise-grade
- No unfilled {{placeholders}}, TBD, XXX, TODO
- Frontmatter status: ready  (or status: n/a for 08 with justification)
- **Resolution Loop coverage:** every action named in `INTAKE.md § Resolution Loop` must appear in this doc where it applies (02 as a step/branch/atypical; 03 as a control + state contract; 04 as an endpoint or deferred-note; 05 as a transition or audit row; 06 as a permission entry; 10 as a scenario). If a documented action is out of scope for this doc, cite which sibling doc owns it.
- <template-specific checks: e.g., persona has 5+ tools + literal quotes + numeric JTBD metrics; journey has ≥7 steps with screen_id or endpoint_id; **journey golden path contains at least one state-changing action step OR an explicit §9 Non-Goal lock naming the deferred action-loop**>

# After writing the target file
1. **Update docs/slices/<SLICE_ID>/PROGRESS.md** per § Progress Tracking Protocol:
   - Tick the phase-checklist box for this doc
   - Set `current_step` and `next_action` to the next pending sub-step
   - Append your target file's absolute path to "Files Written"
   - Append one line to "Session History": `- <ISO timestamp> session <n>: wrote <NN>-<slug>`
   - Refresh `last_updated`
2. Self-check against `.claude/agents/slice-validator-<name>.md` rules (mentally). Revise if you'd fail.

# Return ≤200 words
- Files written (absolute paths)
- 2–3 key decisions you locked + rationale
- Any contradictions with sibling docs you spotted
- Any validator concerns you could not fully resolve

Do NOT return the file content — it's on disk; main thread will re-read PROGRESS.md.
```

**Reference-example caveat.** `docs/slices/00-example-bookmark-v1/02-JOURNEY.md` predates the §3a Resolution Loop patch. When your fidelity reference lacks a section that the template requires, **follow the template, not the example.** The template is always the normative source; the example is a quality bar for sections both contain.

For **parallel fanout phases** (3.1–3.3, 4.1–4.5): each agent gets the template with its own NN + doc name. Send all N agent calls in one message — Claude Code runs them in parallel.

For **reconciliation (3.4)**: prompt differs — "Read 03, 04, 05. Replace placeholder IDs with reconciled real IDs. Ensure every UI interaction in 03 cites an endpoint in 04 and every endpoint in 04 touches a table in 05."

For **validator fix-up loops (5.2)**: prompt is "Validator returned FAIL with blockers: <list>. Re-read the affected doc, fix only the cited issues, preserve everything else. Do not invent new content."

## Progress Tracking Protocol

`PROGRESS.md` schema (same as before this edit — sub-agents use it):

```markdown
---
slice_id: <SLICE_ID>
mode: interactive | auto | skeleton
status: in-progress | completed | abandoned
one_liner: "<verbatim>"
created_at: <ISO>
last_updated: <ISO>
completed_at: <ISO when status=completed>
current_phase: 0|1|2|3|4|5
current_step: "<phase>.<sub>-<label>"
next_action: "<one-line instruction for next session>"
validator_loops: 0
validator_verdict: pending|PASS|WARN|FAIL
session_count: <int>
---

# Slice Progress: <SLICE_ID>

## Phase Checklist
- [x] Phase 0 — Contract loaded
- [x] Phase 1 — Intake (INTAKE.md)
- [ ] Phase 2 — Foundation
  - [ ] 2.1 01-PERSONA.md
  - [ ] 2.2 02-JOURNEY.md
- [ ] Phase 3 — Contracts
  - [ ] 3.1 03-UI-SPEC.md
  - [ ] 3.2 04-API-CONTRACT.md
  - [ ] 3.3 05-DATA-MODEL.md
  - [ ] 3.x ID reconciliation
- [ ] Phase 4 — Supporting
  - [ ] 4.1 06-SECURITY-TENANCY.md
  - [ ] 4.2 07-OBSERVABILITY.md
  - [ ] 4.3 08-AI-SURFACE.md
  - [ ] 4.4 09-SEED-DATA.md
  - [ ] 4.5 10-TEST-PLAN.md
- [ ] Phase 5 — Index & validation
  - [ ] 5.1 00-SLICE-INDEX.md
  - [ ] 5.2 validator PASS

## Files Written (absolute paths)
- …

## Supporting Artifacts Written
- …

## Key Decisions Locked
<summarized from INTAKE.md>

## Resume Command
`/slice-resume <SLICE_ID>`

## Stop Reason (when status != completed)
<one paragraph>

## Session History
- <ISO> session <n>: <what this session did>
```

### Rewrite PROGRESS.md at these moments
1. Slice dir first created.
2. INTAKE.md finalized (end Phase 1).
3. **Every sub-agent writes it as its final step.**
4. After Phase 3 reconciliation.
5. After Phase 5 validator loop (increment `validator_loops`).
6. On final PASS (`status: completed`).
7. On any early stop (`status: in-progress`, fill Stop Reason).

## Self-checks before declaring done

Run on the slice dir. All must pass:

```bash
ls docs/slices/<SLICE_ID>/ | grep -cE '^(00|01|02|03|04|05|06|07|08|09|10)-.*\.md$'   # == 11

grep -liE "drives growth|seamless|powerful|best-in-class|streamlined|cutting-edge|robust|intuitive|game-changing|synergy|leverage|next-gen|world-class|enterprise-grade" docs/slices/<SLICE_ID>/*.md   # empty

grep -rn "{{.*}}\|\[REQUIRED\]\|TBD\|XXX" docs/slices/<SLICE_ID>/*.md   # empty

grep -h "^status:" docs/slices/<SLICE_ID>/[0-9]*.md   # all ready or n/a
```

Any failure → spawn a fix-up agent (not fix inline) → rerun.

## Failure protocol

- **User abandons Phase 1 mid-way** → proceed autonomously with assumptions documented in INTAKE + PROGRESS.
- **Validator stuck after 3 loops** → `status: in-progress`, `validator_verdict: FAIL`, Stop Reason lists blockers. Emit checkpoint. Do not fabricate PASS.
- **Wall-clock exceeds 2h** → finish current Agent if one is in flight, update PROGRESS.md with Stop Reason, emit PAUSED block. Context growth is handled by Claude Code auto-compact — do not manually gate on it.
- **Sub-agent returns but target file was not written** → re-read PROGRESS.md to confirm state, re-spawn agent with explicit "The previous run failed to write the file. Write it now."
- **Cannot find service/persona in codebase** → main thread asks the user which service/module owns the entity. No guessing.

## Budgets (hard limits)

- Questions: ≤ 10 in Phase 1 (Batch A classification ≤5 + Batch B resolution-loop ≤5). Batch B is mandatory except in `[AUTO]`.
- Validator loops: ≤ 3 in Phase 5
- Time: 60–90 min target, stop at 2h
- Context: not a budget — Claude Code auto-compacts. Do not manually gate on percentage.

## Reference

- Prompt source of truth: `docs/slice-templates/SLICE-BUILDER-PROMPT.md`
- Templates: `docs/slice-templates/*.template.md` (sub-agents read these)
- Validators: `.claude/agents/slice-validator-*.md`
- Reference filled example: `docs/slices/00-example-bookmark-v1/`
- Resume skill: `.claude/skills/slice-resume/SKILL.md`
