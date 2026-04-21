# Reddit — r/ClaudeAI

**Venue:** https://www.reddit.com/r/ClaudeAI/submit
**Flair:** "Resources & API" or "Showcase" (depends on current sub rules)

---

## Title

```
I open-sourced the 3 Claude Code skills I use to turn a one-line feature idea into a production vertical slice
```

Alt titles:

- `Vertical-slice feature planning with Claude Code: 3 skills + 12 validator agents, MIT`
- `Opinionated Claude Code skills for going from "a user can do X" → working code`

## Body

```
TL;DR — https://github.com/arun-mosai/claude-code-slice-skills

Three Claude Code skills that compose:

- /slice "a CSM can flag a customer as at-risk and get a win-back playbook"
- /slice-resume <id>   (if context fills and you /clear)
- /slice-build <id>

A "slice" here = one persona's one job, end-to-end. UI + API + DB +
seed data + tests, all together, demonstrating the feature works for a
named human — not a feature flag hiding behind TODOs.

---

## What problem this solves

Most AI feature planners either over-specify (write a novel that drifts
from code) or under-specify (hand you a stub that compiles but
demonstrates nothing). This encodes a middle path:

- Named persona with age, tenure, 5+ tools, literal verbatim quotes
- Seed rows with recognizable names (Acme, Wayne) — not "Customer 1"
- Every screen covers 7 canonical states
- Every endpoint cites the DB table it touches
- Banned-phrase list (seamless, powerful, leverage...) to kill slop
- 12 validator sub-agents run pass/warn/fail checks before implementation

---

## Load-bearing ideas

1. **Resolution Loop** — phase 1 Batch B forces 4 decisions before UI:
   closure actions, impact preview, cross-boundary routing, reversibility.
   Result: slices that let the persona *act*, not just *look*.

2. **Atomic cycle commit** — nothing commits mid-cycle. State lives in
   PROGRESS.md + BUILD-PROGRESS.md (survive /clear). Only after full
   7-step verification AND user "commit" does the cycle produce git
   history.

3. **Blocker verification via shape, not prefix** — every RESOLVED /
   WAIVED marker is verified live against its Resolution: pointer. The
   file must exist, the grep must hit, the migration must round-trip.
   Works across any ID taxonomy (D-*, C-*, GAP-*, ad-hoc).

4. **Project-stage calibration** — 7 stages (prototype → regulated)
   gate ceremony. Pre-prod: .down.sql optional, feature flags skipped.
   Regulated: audit retention gates every write. Same skills, different
   bar.

5. **Sub-agent-only writes** — main thread orchestrates; sub-agents
   absorb template reads. This is what makes the pipeline survive
   long-context / /clear.

---

## Stack defaults

TypeScript monorepo + Postgres + raw pg + Zod + React + Vitest +
Playwright. PORTING.md covers Django / Rails / Go / Next.js and which
parts are stack-neutral vs opinionated.

---

## What I'd love feedback on

- Does the Resolution Loop reasoning land or feel overengineered?
- Project-stage table — any lifecycle rows I'm missing (regulated
  fintech? healthcare?)
- Any validator agents you'd want that I didn't ship?

MIT, issues welcome. Not a product — just an open extraction of a
workflow from a private codebase.
```

---

## Cross-post candidates

- r/LocalLLaMA — reframe as "multi-agent orchestration pattern for code generation"
- r/programming — only if a top HN comment lands well; otherwise rolls into noise
- r/ChatGPTCoding — may get traction with OpenAI adapt-it-yourself crowd

## Timing

- Reddit weekdays 9–11 am ET or Sunday evening work best for r/ClaudeAI
- Avoid Friday/Saturday (low engagement)
- Engage with top 5 comments within 2 hours
