# Vertical Slice Templates

A locked set of **11 templates** that, when filled for a single persona × journey, give an LLM (Opus 4.7) enough unambiguous signal to produce one complete production-quality vertical slice — UI + API + DB + security + observability + AI + seed data + tests — in a coherent pass.

**Not** for human product discussion (use `docs/ux-workflows/`). **Not** for cross-cutting patterns (use `agent_docs/*-patterns.md`). **Use this** when you want a shippable, demonstrable slice.

---

## The 11 Templates

| # | File | Purpose | Hard-gate for code? |
|---|------|---------|---|
| 00 | `00-SLICE-INDEX.template.md` | Orchestrator: ties all 10 siblings together with DoD checklist | — |
| 01 | `01-PERSONA.template.md` | One named human: JTBD, pain points, day-in-life, anti-patterns | ✅ before 02 |
| 02 | `02-JOURNEY.template.md` | One workflow: golden path + branches + atypical + failure modes | ✅ before 03/04/05 |
| 03 | `03-UI-SPEC.template.md` | Every screen × every state (7 states min) + copy deck | ✅ before UI code |
| 04 | `04-API-CONTRACT.template.md` | OpenAPI fragment + gateway route entries + error taxonomy | ✅ before handler code |
| 05 | `05-DATA-MODEL.template.md` | DDL + indexes + FKs + migrations + invariants | ✅ before any code |
| 06 | `06-SECURITY-TENANCY.template.md` | Permission matrix + tenancy invariants + PII inventory + authz tests | Close before merge |
| 07 | `07-OBSERVABILITY.template.md` | Metrics, logs, traces, dashboards, alerts, event catalog | Close before merge |
| 08 | `08-AI-SURFACE.template.md` | Prompts, schemas, evals, cost model, fallbacks | Close before merge |
| 09 | `09-SEED-DATA.template.md` | Literal rows making slice runnable + assertion queries | Close before merge |
| 10 | `10-TEST-PLAN.template.md` | Gherkin scenarios + unit/integration/e2e/chaos/perf/a11y | Close before merge |

---

## Execution Order

```
01-PERSONA ──┐
             ├─► 02-JOURNEY ──┬─► 03-UI-SPEC ──┐
                              ├─► 04-API ──────┼─► 06-SECURITY ──► 07-OBSERVABILITY ──► 08-AI ──► 09-SEED ──► 10-TEST ──► 00-INDEX (finalize)
                              └─► 05-DATA ─────┘
```

**Hard gate:** Do not start writing code until 01→05 are marked `status: ready` in their frontmatter. 06–10 may fill in parallel with implementation but must close before merge.

---

## Usage

```bash
# 1. Create a slice instance dir
mkdir -p docs/slices/<slice-id>

# 2. Copy templates into it
cp docs/slice-templates/*.template.md docs/slices/<slice-id>/
rename 's/\.template\.md$/\.md/' docs/slices/<slice-id>/*.template.md

# 3. Fill in order: 01 → 02 → 03/04/05 → 06/07/08/09/10 → 00 (index last)
# 4. Validate at each gate: run the matching validator agent
# 5. When all 11 green and status: ready — hand to Opus 4.7 for implementation
```

---

## Validation

Each template has a paired validator agent at `.claude/agents/slice-validator-<n>.md`. Run them individually or all at once via `slice-validator-orchestrator`.

A validator returns one of:
- `PASS` — template ready, proceed to next
- `FAIL` — specific section(s) missing or generic; fix and re-run
- `WARN` — acceptable but has risk flagged for later

The orchestrator returns a **single slice-level verdict** with per-template status. A slice cannot progress to `status: ready-to-build` until orchestrator returns `PASS`.

---

## Why This Exists

Large codebases have dozens of services and hundreds of tables. What it doesn't have is a **single runnable slice** for **one persona** with **concrete seed data** that proves the whole stack works end-to-end. Existing docs are feature-centric and human-oriented; they describe features in isolation, not a persona's full experience.

These templates force the specificity needed to eliminate LLM drift:

- **Abstract persona → median UI** → forced specificity (named person, literal quote, measurable JTBD)
- **Happy-path bias → broken empty/error states** → explicit 7-state matrix per screen
- **"Use Claude" → undefined AI behavior** → literal prompt + eval + cost model + fallback
- **"Seed demo data" → no verification** → named rows with recognizable identifiers
- **"Tests pass" → nothing demonstrated** → Gherkin tied to seed row IDs and journey steps

---

## Cross-Reference IDs

IDs propagate through all docs so a failing test traces back to the pain it serves:

```
persona_id ─► journey_id ─► step_id ─► screen_id ─► endpoint_id ─► seed_row_id ─► scenario_id
```

Format: `SLICE-<slice-id>-<type>-<nnn>` — e.g., `SLICE-bookmark-v1-JOURNEY-001`, `SLICE-bookmark-v1-UI-003`.

---

## Example Slice

See `docs/slices/00-example-bookmark-v1/` for a worked trivial slice (PM persona bookmarks a Feature). Copy its structure as the fidelity reference.
