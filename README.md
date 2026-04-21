# Claude Code Slice Skills

Turn a one-line feature description into a complete, production-quality **vertical slice** — persona, journey, UI spec, API contract, data model, security, observability, AI surface, seed data, tests — plus working code that implements the contract, all driven by three composable Claude Code skills.

```
/slice "A CSM can flag a customer as at-risk and get a win-back playbook"
   ↓ (design — 11 docs, sub-agent orchestrated)
/slice-resume <id>              # optional; for paused sessions
   ↓
/slice-build <id>
   ↓ (implementation — migrations, handlers, UI, tests, events)
```

A "slice" here means **one persona's one job end-to-end** — UI + API + DB + seed data + tests that together demonstrate the feature works for a named human, not a feature flag hiding behind TODOs.

---

## Why this exists

Most AI feature planners either over-specify (write a novel that drifts from code) or under-specify (hand you a stub that compiles but doesn't demonstrate anything). This repo encodes an opinionated middle path:

1. **Force specificity where LLMs drift.** Named persona with age, tenure, 5+ tools, literal quotes. Seed rows with recognizable names (Acme, Wayne — not `Customer 1`). Every screen covers 7 canonical states. Every endpoint cites the table it touches. Banned-phrase list (`seamless`, `powerful`, `leverage`, ...) to kill marketing slop.
2. **Freeze high-risk contracts before implementation.** Persona → Journey → (UI + API + DB, parallel) are locked before any code is written. Later docs (security, observability, AI, seed, tests) fan out in parallel.
3. **Make every step verifiable.** 12 validator sub-agents run pass/warn/fail checks on each doc. Nothing proceeds to implementation until the whole slice PASSes.
4. **Survive `/clear`.** A durable `PROGRESS.md` checkpoint in the slice directory lets you wipe context mid-design and resume. The 3-skill pipeline is re-entry-safe.
5. **Atomic cycle commit.** Nothing is committed until the full `/slice → /slice-resume → /slice-build` cycle finishes verification **and** the user says "commit." Partial work lives on the working tree backed by `PROGRESS.md` + `BUILD-PROGRESS.md`.

---

## What's in the box

| Path | What |
|------|------|
| `.claude/skills/slice/SKILL.md` | Phase 1–5: design the slice (11 docs) |
| `.claude/skills/slice-resume/SKILL.md` | Resume a paused slice from its checkpoint |
| `.claude/skills/slice-build/SKILL.md` | Phase 6: implement the slice as code + tests + events |
| `.claude/agents/slice-validator-*.md` | 12 validator sub-agents (orchestrator + 11 per-doc) |
| `.claude/project-stages.md` | Lifecycle-stage table that calibrates which gates run (MVP → regulated) |
| `docs/slice-templates/` | 11 doc templates + the master `SLICE-BUILDER-PROMPT.md` |
| `docs/slices/00-example-bookmark-v1/` | Reference filled slice (PM bookmarks a Feature) |

---

## Install

Claude Code reads skills from two locations: per-user (`~/.claude/`) or per-project (`<repo>/.claude/`). Most teams want per-project so the skills are versioned with the codebase.

```bash
# Clone somewhere
git clone https://github.com/arun-mosai/claude-code-slice-skills.git
cd claude-code-slice-skills

# Copy into your project (per-project install)
cp -R .claude/skills/slice           /path/to/your-project/.claude/skills/
cp -R .claude/skills/slice-resume    /path/to/your-project/.claude/skills/
cp -R .claude/skills/slice-build     /path/to/your-project/.claude/skills/
cp    .claude/agents/slice-validator-*.md /path/to/your-project/.claude/agents/
cp    .claude/project-stages.md      /path/to/your-project/.claude/
cp -R docs/slice-templates           /path/to/your-project/docs/
cp -R docs/slices/00-example-bookmark-v1 /path/to/your-project/docs/slices/
```

Then in your project's `CLAUDE.md`, add a stage declaration (top of file):

```markdown
**PROJECT_STAGE:** `developing-mvp` (last reviewed 2026-04-21)
```

Valid values are the stage names in `.claude/project-stages.md` — `prototype-exploration`, `developing-mvp`, `internal-alpha`, `private-beta`, `public-beta`, `general-availability`, `regulated-production`. Each stage adjusts which gates the skills enforce (e.g., `.down.sql` required vs optional, feature-flag ceremony, observability floor).

---

## Quickstart

```text
# 1. Describe a feature in one sentence.
/slice A Customer Success Manager can flag a customer as at-risk and get an auto-generated win-back playbook.

# 2. Answer ≤10 intake questions (Batch A = classification, Batch B = resolution loop).
# 3. Watch sub-agents write the 11 docs in parallel where possible.
# 4. If context fills and you /clear, just:
/slice-resume <slice-id>

# 5. When the validator returns PASS, implement:
/slice-build <slice-id>

# 6. Sub-phases 6.1–6.7 execute in order (data → API → events → AI → UI → tests → verify).
# 7. When the 7-step verification passes AND you say "commit", the cycle commits atomically.
```

See `docs/slices/00-example-bookmark-v1/` for a filled reference slice at the fidelity the validators expect.

---

## The design philosophy

A few ideas in here are load-bearing and worth reading even if you don't use the skills.

### The **Resolution Loop** (Phase 1 Batch B of `/slice`)

Most "user journeys" degenerate into `navigate → read → back-button` and ship view-only products. Batch B forces four decisions *before* UI code is written:

- **Closure actions** — which domain verbs move this persona to "done"? (Derived from the state machine, not invented.)
- **Impact preview** — how does the UI show consequences before commit? (Severity-matched: undo for idempotent writes, confirm dialog for irreversible transitions, side-panel diff for multi-field edits.)
- **Cross-boundary routing** — which other services fan out? (Inspected from the codebase's module-boundary rules.)
- **Reversibility** — what does undo look like? (Driven by whether the state transition is legal to reverse, not by a wish.)

The result: slices that actually let the persona *act*, not just look.

### **Atomic cycle commit**

The full `/slice → /slice-resume → /slice-build` cycle is one atomic unit of work. The skills commit **nothing** mid-cycle — not per phase, not per endpoint, not "for safety." State persists in two durable files (`PROGRESS.md` + `BUILD-PROGRESS.md`) that survive `/clear`; `git status` is the running diff. Only after 7-step verification passes and the user explicitly approves does the cycle produce commits.

Rationale: a slice is a vertical contract. Migrations + API + UI + events + tests interlock. Partial commits create half-baked review streams and bisection confusion.

### **Blocker verification (never trust the marker)**

In `/slice-build` Step 2.5, every ledger row in `00-SLICE-INDEX.md` that says `RESOLVED` / `WAIVED` / `DEFERRED` is verified *live* against its `Resolution:` pointer — the file must exist, the grep must hit, the migration must round-trip. Markers rot; code doesn't. This is generic across any blocker taxonomy (`D*`, `C*`, `GAP-*`, free-form IDs) because it keys on the **Status + Resolution shape**, not an ID prefix.

### **Project stage calibrates ceremony**

`.claude/project-stages.md` encodes six lifecycle knobs (`migration_mutability`, `migration_reversibility`, `schema_mirror_sync`, `data_migration_strategy`, `rollout_safety`, `observability_floor`) across seven stages. Skills read the stage from `CLAUDE.md` and gate each check `[conditional: knob=value]`. Pre-production, `.down.sql` is optional and feature-flag ceremony is skipped. At `regulated-production`, audit-trail retention gates every write. Same skills, different bar.

---

## Portability

The skills ship with an opinionated stack assumption: TypeScript monorepo, Express-style services, raw `pg` + Postgres, Zod validation, API gateway with fail-closed route permissions, React frontend, Vitest + Playwright. See **[PORTING.md](PORTING.md)** for the exact knobs you'd turn to adapt to Django / Rails / Next.js / Go / etc. The skill **structure** (phases, validators, resolution loop, atomic commit, blocker verification) is stack-neutral; the defaults happen to match one particular house style.

---

## Contributing

This is a thin open-source extraction of a workflow used in private codebases. Expect rough edges where the extraction generalized something that was once specific. Issues and PRs welcome — particularly:

- Additional project-stage rows
- Validator agents for new doc types
- Language/framework ports of the implementation assumptions in `/slice-build`

---

## License

MIT. See [LICENSE](LICENSE).
