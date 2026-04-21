# X / Twitter thread

**Venue:** https://twitter.com/compose/tweet
**Format:** 10-tweet thread. Tweet 1 is the hook; 10 is the CTA.

---

## Tweet 1 (hook)

```
I open-sourced the 3 Claude Code skills I use to turn a one-line feature
idea into a production vertical slice.

Persona → journey → UI + API + DB (parallel) → everything else.
12 validator agents. MIT.

🧵

github.com/arun-mosai/claude-code-slice-skills
```

## Tweet 2 — what a slice is

```
A "slice" = one persona's one job, end-to-end.

UI + API + DB + seed data + tests that together demonstrate the feature
works for a named human.

Not a feature flag hiding behind TODOs. Not a spec that drifts from
code. The thing actually works.
```

## Tweet 3 — the pipeline

```
/slice "A CSM can flag a customer as at-risk and get a win-back playbook"
   ↓  design (11 docs, sub-agent orchestrated)
/slice-resume <id>    ← if you /clear mid-design
   ↓
/slice-build <id>
   ↓  migrations, handlers, UI, tests, events
```

## Tweet 4 — force specificity

```
LLMs drift when you don't pin them. So the skills force it:

— Persona with age, tenure, 5+ tools, literal verbatim quotes
— Seed rows named Acme, Wayne — not "Customer 1"
— Every screen covers 7 canonical states
— Every endpoint cites the table it touches
— Banned phrases: seamless, powerful, leverage
```

## Tweet 5 — Resolution Loop

```
Most AI-generated "user journeys" degenerate into:
  navigate → read → back-button → ship

Phase 1 Batch B forces 4 decisions *before* any UI:
• closure actions
• impact preview (severity-matched)
• cross-boundary routing
• reversibility

Result: slices where the persona can act, not just look.
```

## Tweet 6 — atomic cycle commit

```
Nothing commits mid-cycle. State lives in two durable files
(PROGRESS.md + BUILD-PROGRESS.md) that survive /clear.

Only after 7-step verification passes AND you say "commit" does git
history get produced.

Partial commits create half-baked review streams. We don't.
```

## Tweet 7 — blocker verification

```
Every ledger row that says RESOLVED / WAIVED is verified live against
its Resolution: pointer — file must exist, grep must hit, migration
must round-trip.

Markers rot. Code doesn't. Shape-based, not prefix-based — works
across any ID taxonomy.
```

## Tweet 8 — project stages

```
One skill set, different ceremony bars.

.claude/project-stages.md encodes 6 lifecycle knobs across 7 stages
(prototype → regulated-production).

Pre-prod: .down.sql optional. Flags skipped.
Regulated: audit trail gates every write.

Set PROJECT_STAGE in CLAUDE.md. Skills calibrate automatically.
```

## Tweet 9 — stack neutrality

```
Ships with TS monorepo + Postgres + Zod + React defaults.

PORTING.md documents every stack assumption and maps it to Django /
Rails / Go / Next.js.

The skill *structure* (phases, validators, resolution loop, atomic
commit, blocker verification) is stack-neutral. The defaults happen to
match one house style.
```

## Tweet 10 — CTA

```
MIT. Extracted from a private codebase, so rough edges welcome. Issues
and PRs especially helpful for:

— New project-stage rows (fintech? healthcare?)
— Validator agents for new doc types
— Language ports (Python / Go / Ruby)

github.com/arun-mosai/claude-code-slice-skills
```

---

## Posting notes

- Schedule tweet 1 for 9–11am PT on a Tuesday or Wednesday
- Do NOT auto-post the thread — drip tweets 2-10 over ~2 minutes so each
  shows up in feeds separately
- Quote-tweet the thread from secondary accounts only after 24h
- Tag @AnthropicAI only on tweet 1 and only if the thread gains
  traction (>50 likes) — tagging at zero engagement hurts reach
- Relevant hashtags (put 1 in tweet 1, optional): #ClaudeCode #LLM #AI
