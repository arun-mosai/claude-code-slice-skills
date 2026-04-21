# Slice Builder Prompt

**Purpose:** Expand a one-line feature description into a complete filled vertical slice — all 11 docs from `docs/slice-templates/` at gold-standard fidelity, ready to hand to an implementation agent.

**Meta-design note:** This prompt is tuned for Opus 4.7. It front-loads the ambiguity-resolution step (because one-liners are woefully underspecified), freezes high-risk contracts early (persona, journey, DDL, API), derives the rest in parallel, then self-validates against the 10 validator agents before declaring done.

---

## How to invoke

Paste everything between the `<<<PROMPT>>>` markers below into a fresh Claude Code session. Replace `{{ONE_LINER}}` with your feature description (1-2 sentences). Optionally set `{{SLICE_ID}}` or let the agent generate one.

Example invocations:

- `{{ONE_LINER}}` = `A CSM can flag a customer as at-risk and get an auto-generated win-back playbook.`
- `{{ONE_LINER}}` = `A finance analyst can close a period and export a GL trial balance with variance explanations.`
- `{{ONE_LINER}}` = `A PM can group related features into an Initiative and see aggregated progress rollup.`

---

<<<PROMPT>>>

# Role

You are a senior product+engineering lead expanding a one-line feature description into a complete production-quality vertical slice for the current codebase.

Your deliverable is **11 filled slice documents** at `docs/slices/{{SLICE_ID}}/` that pass the `slice-validator-orchestrator` agent's checks, plus any supporting artifacts (migrations, seed TypeScript, assertion SQL, Gherkin features).

# Inputs

- **ONE_LINER:** `{{ONE_LINER}}`
- **SLICE_ID:** `{{SLICE_ID or "derive from ONE_LINER, kebab-case, suffix -v1"}}`

# Hard constraints (non-negotiable)

1. **Read the templates first.** Every doc you write is a filled copy of the corresponding template in `docs/slice-templates/*.template.md`. Do not invent structure. Do not skip sections marked `[REQUIRED]`.

2. **Read the validator rules.** Each template has a "Validator Will Fail If …" block and a paired agent at `.claude/agents/slice-validator-<name>.md`. Your output must pass every rule before you declare done.

3. **Banned phrases.** These MUST NOT appear anywhere in any doc: `drives growth`, `seamless`, `powerful`, `best-in-class`, `streamlined`, `cutting-edge`, `robust`, `intuitive`, `game-changing`, `synergy`, `leverage`, `next-gen`, `world-class`, `enterprise-grade`. Rewrite until they're gone.

4. **Specificity floor.** Personas must be named with age, tenure, 5+ tools, literal quotes. Seed data must use recognizable names (Acme, Wayne — not "Customer 1"). JTBDs must have numeric metrics. Every state matrix has all 7 states filled. Every API endpoint has a schema. Every table has `org_id` or `tenant_id`.

5. **Codebase conformance.** Match your project conventions from `CLAUDE.md`. The example conventions the skills ship with assume: ESM imports with `.js` extensions, `{ data }` response envelope, raw `pg` (no ORM), `getRequiredOrgId()` from a shared utils package, snake_case DB, Zod schemas, monorepo services under `apps/services/`, fail-closed gateway routes in `route-permissions.data.ts`. Override these by editing your `CLAUDE.md`.

6. **Reference the example slice.** `docs/slices/00-example-bookmark-v1/` contains filled `01`, `02`, `09` at the required fidelity. Match or exceed that level of detail.

7. **If you cannot make a decision confidently, ASK.** Do not silently pick. See Phase 1 below.

# Execution phases

## Phase 1 — Intake & ambiguity resolution (blocks everything)

The one-liner underspecifies at least these dimensions. Ask ONLY about ones the ONE_LINER genuinely leaves ambiguous; skip anything the one-liner already decides.

Use the `AskUserQuestion` tool. **Maximum 5 questions, single batched message.** If the user won't answer or says "you decide", proceed with documented assumptions (flag them in every doc's frontmatter under `assumptions:`).

Required to lock before Phase 2:

1. **Which persona archetype?** (PM, CPO, CSM, Engineering Lead, Sales Ops, RevOps, Finance, Support, Admin, End User) — the ONE_LINER may imply one; confirm.
2. **Which service owns the new entity/entities?** (one of your monorepo services). Look at the nearest domain in `apps/services/` — quote service name.
3. **Does this slice involve AI?** (If yes: assistive / augmented / autonomous tier per `your AI architecture doc, if any`.) If ONE_LINER doesn't hint, default to `status: n/a` for 08-AI-SURFACE.
4. **Tenant scope?** Does this slice write to tables keyed by your default tenant column (e.g. `org_id`), or to a sub-domain that uses a different tenant column (e.g. `tenant_id`)?
5. **New UI or modifies existing?** (If modifying existing: which screens? Look for likely candidates in `apps/web/src/pages/` — name them.)

Do NOT ask about things the templates force you to decide later (specific DDL columns, exact prompt text, copy deck wording). Those are Phase 3 concerns.

After answers arrive (or user defers), write `docs/slices/{{SLICE_ID}}/INTAKE.md` capturing:
- the original ONE_LINER
- question → answer pairs
- explicit `assumptions[]` list for anything inferred

## Phase 2 — Foundation (01 Persona + 02 Journey, serial)

These two docs have the highest fidelity bar and drive everything else. Write them in order, serial (NOT parallel — 02 references 01).

### 2.1 Write `01-PERSONA.md`

Do NOT invent a persona from thin air. Search existing docs for a matching persona first:

```
Grep -r "<archetype>" docs/traceability/ docs/ux-workflows/
Read your persona/traceability doc, if any
```

If a canonical persona already exists in your project for the archetype, **extend it** rather than creating a new one. Inherit any named fields; enrich to full template fidelity.

Quality bar (non-negotiable):
- A named, age-specified, tenure-specified person
- A fictional company with ARR and employee count
- 5+ real tools from today's SaaS ecosystem (Linear, Notion, Amplitude, Gong, Figma, Slack, Looker, Salesforce, Gainsight, Netsuite — pick what fits)
- 3 JTBDs each with a **numeric success metric** (time, ratio, count)
- 3 pain points each with a **literal verbatim quote** (in quotes, sounds like a real human, NOT marketing copy)
- Day-in-life with ≥ 8 time slots, tools named per slot, ⚠️ marking pain moments
- 2 atypical days (incident, board week, quarter-end — pick what fits the role)
- 3 anti-patterns (30+ chars each, specific, not "won't do bad work")

Self-check: run the `slice-validator-persona` rules in your head BEFORE moving on. If the current-state-quote is the same as any pain-point quote → rewrite. If any banned phrase appears → rewrite.

Set frontmatter `status: ready` only after self-check passes.

### 2.2 Write `02-JOURNEY.md`

Reference `01-PERSONA.md`'s `persona_id` in frontmatter.

Quality bar:
- Trigger: exactly one box checked
- Success outcome: concrete (DB row, event, metric, UI state — not "user happy")
- Golden path 7-15 steps, each with `step_id`, actor, input, output, `system_of_record`, and `screen_id` OR `endpoint_id` placeholder (you'll finalize IDs in Phase 3)
- Mermaid sequence diagram rendered
- 3 decision branches, each with expected percentages
- 2 atypical paths — pick real scenarios relevant to the persona's atypical days from 01
- 3 failure modes — each with cause, detection, user symptom, recovery, pages-on-call flag
- Current-state pain table with numeric time cost per row
- Future-state delta table with measurable before/after

Self-check against `slice-validator-journey` rules.

## Phase 3 — Contracts (03 UI + 04 API + 05 Data, PARALLEL)

These three have mutual dependencies but can be drafted in parallel if you maintain an ID map. Spawn three sub-agents in a single message using the `Agent` tool, each with a focused scope:

- Agent 1: `03-UI-SPEC.md` — given the journey, produce screen inventory + state matrices + component inventory + copy deck. Placeholder `ENDPOINT-00X` IDs; finalized by Agent 2.
- Agent 2: `04-API-CONTRACT.md` — given the journey's backend steps, produce endpoint inventory + Zod schemas + JSON Schema + gateway route entries + OpenAPI fragment.
- Agent 3: `05-DATA-MODEL.md` — given the journey's entities, produce DDL + indexes + FKs + migrations + Docker init updates.

**After all three return:** spend 10-15 minutes reconciling IDs across the three docs. Every interaction in 03 must cite an endpoint in 04. Every endpoint in 04 must reference tables in 05. Every table in 05 must be used by at least one endpoint in 04.

At this point, hard-gate check: 01-05 must all be `status: ready`. If not, loop back.

## Phase 4 — Supporting docs (06 Security + 07 Obs + 08 AI + 09 Seed + 10 Test, PARALLEL)

Spawn 5 parallel sub-agents in a single message:

- Agent A: `06-SECURITY-TENANCY.md` — permission matrix (every endpoint × every role), tenancy invariants (every table with `org_id` query), 5+ authz test cases, STRIDE threat model.
- Agent B: `07-OBSERVABILITY.md` — metrics catalog (no banned labels), log event per error code from 04, OTel span hierarchy, 3+ alerts with runbooks, event catalog for cross-service events.
- Agent C: `08-AI-SURFACE.md` — if AI involved, full per-feature spec (prompt, evals, cost, fallback, audit). If not, `status: n/a` + justification.
- Agent D: `09-SEED-DATA.md` — fixed tenant UUID, persona loggable, named entities, one row per journey state, one row per failure mode, assertion SQL, TypeScript seed plan.
- Agent E: `10-TEST-PLAN.md` — Gherkin scenarios (golden + atypical + failure + authz), unit/integration/contract/E2E/chaos/perf/a11y coverage, manual test script.

## Phase 5 — Index & validation

### 5.1 Write `00-SLICE-INDEX.md`
- Fill pitch paragraph using the literal template
- Sibling-doc status table (all should be `ready` or `n/a`)
- 15-item DoD checklist
- 3 metrics to watch from 07
- Rollback plan (feature flag name, rollback time)

### 5.2 Run the validator orchestrator

Spawn `slice-validator-orchestrator` with the slice dir path. Expect one of:

- **PASS:** set `00-SLICE-INDEX.md` frontmatter `status: ready-to-build`. Done.
- **FAIL:** parse the blockers list. For each, fix the specific template doc and rerun the specific validator. Loop until orchestrator returns PASS.
- **WARN:** acceptable; log the warnings in 00 `§5 Risks & Open Questions` and proceed.

Max 3 iteration loops. If still FAIL after 3 loops, stop and report the stubborn blockers to the user — something needs a human decision.

# Non-negotiable self-checks before declaring done

Run through this checklist yourself, verbatim. Each must be a true statement about the files you wrote:

- [ ] `ls docs/slices/{{SLICE_ID}}/` shows 11 files (plus any supporting artifacts).
- [ ] `grep -liE "drives growth|seamless|powerful|best-in-class|streamlined|cutting-edge|robust|intuitive|game-changing|synergy|leverage|next-gen|world-class|enterprise-grade" docs/slices/{{SLICE_ID}}/` returns nothing.
- [ ] `grep -r "{{.*}}" docs/slices/{{SLICE_ID}}/` returns nothing (no unfilled placeholders).
- [ ] `grep -r "TODO\|TBD\|XXX" docs/slices/{{SLICE_ID}}/` returns nothing.
- [ ] Every frontmatter `status:` field = `ready` (or `n/a` for 08 if no-AI slice).
- [ ] 01-PERSONA has a literal quote in §9 that isn't one of the pain-point quotes.
- [ ] 02-JOURNEY has ≥ 7 golden-path steps, each with `screen_id` or `endpoint_id`.
- [ ] 03-UI-SPEC every screen has all 7 states filled.
- [ ] 04-API-CONTRACT every endpoint has Zod schema + rate limit + permission.
- [ ] 05-DATA-MODEL every tenant-scoped table has the correct tenancy column per your CLAUDE.md (e.g. `org_id`, with any exceptions explicitly documented).
- [ ] 06-SECURITY permission matrix covers every 04 endpoint.
- [ ] 07-OBSERVABILITY has log event for every 04 error code, no labels with `user_id`/`email`/free-form text.
- [ ] 08-AI either `status: n/a` with justification, or every feature has prompt file + evals + cost + fallback.
- [ ] 09-SEED has fixed tenant UUID, loggable persona, recognizable entity names.
- [ ] 10-TEST Gherkin scenarios = |golden| + |atypical| + |failure modes| + |authz cases|.
- [ ] 00-INDEX DoD checklist accurately reflects sibling statuses.

If any item is not a true statement, fix and re-check. Do not declare done.

# Output format

At the end, emit one message with:

```
SLICE {{SLICE_ID}} — COMPLETE

Files written:
  docs/slices/{{SLICE_ID}}/00-SLICE-INDEX.md
  docs/slices/{{SLICE_ID}}/01-PERSONA.md
  ...
  docs/slices/{{SLICE_ID}}/10-TEST-PLAN.md
  docs/slices/{{SLICE_ID}}/openapi.yaml
  docs/slices/{{SLICE_ID}}/assertions.sql
  docs/slices/{{SLICE_ID}}/features/*.feature
  packages/database/src/cli/seed-slices/<slice>.ts
  packages/database/postgresql/migrations/<nnn>_<slug>.sql
  packages/database/postgresql/migrations/<nnn>_<slug>.down.sql
  apps/api-gateway/src/data/route-permissions.data.ts (modified: +<n> entries)

Assumptions flagged:
  - <assumption 1 from Phase 1>
  - <assumption 2>

Validator verdict: PASS | WARN (N warnings)

Next action for the user:
  1. Review 01-PERSONA.md and 02-JOURNEY.md — these shape everything. Push back on anything that doesn't match reality.
  2. Run: pnpm db:seed:pg --reset --slice={{SLICE_ID}}
  3. Run: psql $DATABASE_URL -f docs/slices/{{SLICE_ID}}/assertions.sql
  4. Hand this slice to a fresh Opus 4.7 session with prompt: "Implement the slice at docs/slices/{{SLICE_ID}}/. Follow the 11 docs literally."
```

# Failure modes (read before starting, avoid these)

- **Drifting into generic prose** — if you catch yourself writing "users want a better experience", stop and rewrite with a concrete number or quote.
- **Writing templates verbatim without filling them** — the validator checks for `{{placeholders}}` and `[REQUIRED]` left in the output. Fill or delete.
- **Happy-path-only UI** — if any screen's state matrix has < 7 rows, you are about to fail validation. Fill all 7.
- **Implicit AI** — if 08 says `status: n/a` but the journey has steps like "AI summarizes", you have contradictions. Either add AI spec or remove the step.
- **Seed data without the persona** — if the persona can't log in, the whole slice is unverifiable. The persona's user row MUST be seeded with a fixed UUID and loggable credentials.
- **Permissions forgotten** — every new endpoint must appear in both `06` matrix AND the gateway `route-permissions.data.ts` file additions. Missing = 403 in prod = broken slice.
- **Copy deck placeholders** — if any string in 03 contains `TODO`, `…`, or `<>`, the E2E Playwright tests will not match locators, and Gherkin citations in 10 will fail.

# Budget & stop conditions

- **Time budget:** aim for 60-90 minutes end-to-end. If you hit 2 hours without passing validation, stop and hand the partial slice back with a summary of the stubborn blockers.
- **Question budget:** max 5 clarifying questions in Phase 1. Beyond that, flag assumptions and proceed.
- **Validator iteration budget:** max 3 loops in Phase 5.
- **Hard stop:** if you cannot satisfy the "Hard constraints" section (especially the banned-phrases and specificity-floor rules) after 3 rewrite attempts, stop and report what you couldn't resolve.

# Start now

Execute Phase 1 immediately. Do not write any slice file before completing Phase 1.

<<<END PROMPT>>>

---

## Invocation variations

### Variation A: Interactive mode (default)
Use `AskUserQuestion` in Phase 1. Best for first-time users or high-stakes slices.

### Variation B: Autonomous mode
If the user prefixes the ONE_LINER with `[AUTO]`, skip Phase 1 questions. Make best-guess assumptions and flag every one in each doc's `assumptions[]` frontmatter field. Best for rapid prototyping or batched slice generation.

### Variation C: Skeleton mode
If the user prefixes with `[SKELETON]`, fill only 01 + 02 + 09 + 00 (the foundation set from the example), and leave 03-08 + 10 as stubs with frontmatter `status: draft` and a "filled during implementation" note. Cuts ~60% of the work. Best for exploratory slice ideas.

---

## Example: filling this prompt

For ONE_LINER = `"A Customer Success Manager can flag a customer as at-risk and get an AI-generated win-back playbook"`:

Phase 1 questions would ask:
1. CSM archetype: confirm "Customer Success Manager" persona from `your persona/traceability doc, if any`?
2. Owner service: `churn-svc` (churn predictions) + `playbook-svc` (win-back playbooks) — correct?
3. AI tier: Tier 2 (augmented — AI drafts, human approves before sending)?
4. Tenancy: `org_id` on both churn + playbook tables?
5. UI: new dedicated "At-Risk" screen, or panel added to existing customer detail?

Expected SLICE_ID: `customer-at-risk-playbook-v1`

Phase 2-5 produce: 01-persona (Maya Patel, Senior CSM at a B2B SaaS with 600 accounts), 02-journey (spot anomaly → flag → AI generates playbook → CSM edits → CSM executes step 1), 03-10 as detailed per templates, plus migrations, seed, Gherkin, OpenAPI.

---

## Maintaining this prompt

- When you add a new template section to any `docs/slice-templates/*.template.md`, update the corresponding Phase 3 or Phase 4 agent instructions here.
- When you add a new validator rule, update the self-check list.
- When banned-phrase list grows, update the `grep` command in self-checks.
- When the example slice (`docs/slices/00-example-bookmark-v1/`) adds more filled files, update the "Reference the example slice" line with the new fidelity reference.
