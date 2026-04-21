# Hacker News — Show HN

**Venue:** https://news.ycombinator.com/submit
**Format:** Title + URL + optional first comment.

---

## Title (≤ 80 chars — HN truncates at 80)

```
Show HN: Slice Skills – Claude Code skills that plan features the way PMs do
```

Alt titles (use one):

- `Show HN: Claude Code skills for vertical-slice feature planning`
- `Show HN: Turn one-line feature ideas into production vertical slices`

## URL

```
https://github.com/arun-mosai/claude-code-slice-skills
```

## First comment (post immediately after submission — HN convention)

```
Author here. This is a small open-source extraction of a workflow I use in
private codebases. Three Claude Code skills that take "a CSM can flag a
customer as at-risk and get a win-back playbook" and produce a complete
vertical slice: persona, journey, UI spec, API contract, data model,
security, observability, AI surface, seed data, tests — then the
implementation that satisfies the contract.

A few load-bearing ideas that I think are worth discussion even if you don't
use the skills themselves:

1. Phase structure: persona → journey → (UI + API + DB freeze in parallel) →
   everything else fans out. This mirrors how senior engineers actually
   design features, not how LLMs usually write specs.

2. Resolution Loop: most AI-generated "user journeys" degenerate into
   navigate → read → back-button. Phase 1 Batch B forces the planner to name
   closure actions, impact previews, cross-boundary routing, and
   reversibility *before* any UI code is written.

3. Atomic cycle commit: nothing commits mid-cycle. State lives in two
   durable files (PROGRESS.md + BUILD-PROGRESS.md) that survive /clear.
   Partial commits create half-baked review streams.

4. Project-stage calibration: .claude/project-stages.md encodes 6 lifecycle
   knobs across 7 stages (prototype → regulated-production). Skills read the
   stage from CLAUDE.md and gate checks conditionally. Same skills, different
   bar.

5. Sub-agent-only writes: the main thread orchestrates, sub-agents absorb the
   template reads. This is what makes the pipeline survive /clear and
   extended context use.

Stack is TypeScript monorepo + Postgres + Zod + React by default. PORTING.md
documents every stack assumption and how to port to Django/Rails/Next.js/Go.
The skill *structure* is stack-neutral; the defaults happen to match one
house style.

Honest caveats: this is a thin extraction from private code, so expect rough
edges where the extraction generalized something once specific. MIT. Issues
welcome — especially additional project-stage rows and language ports.
```

---

## Guidance for posting

- Post Monday–Thursday, 8:00–10:00 am ET. Highest signal-to-noise window.
- Do NOT post "Show HN:" in the URL bar with emoji; keep it text-only.
- Reply substantively to every top-10 comment within the first 90 minutes.
  HN ranks by early-comment velocity.
- If someone asks "why another planning tool," point them at PORTING.md's
  "what NOT to change" section — that's the load-bearing answer.
