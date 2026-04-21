# dev.to / Hashnode long-form article

**Venue:** https://dev.to/new or https://hashnode.com/create/story
**Format:** Long-form technical essay with code blocks and philosophy. Medium readers skim; dev.to readers read.

---

## Title

```
How I got Claude Code to plan features the way senior engineers do
```

Alt titles:

- `Vertical slices, not feature flags: 3 Claude Code skills for end-to-end feature design`
- `The Resolution Loop: forcing AI to plan for action, not just navigation`

## Cover image

Use a screenshot of the repo tree or the validator pass/warn/fail output. No stock photos.

## Tags

```
#claude #ai #productivity #opensource
```

## Canonical URL

```
https://github.com/arun-mosai/claude-code-slice-skills
```

---

## Body

```markdown
I spent the last six months turning a private-codebase workflow into three
Claude Code skills. Today I open-sourced them.

**TL;DR** — [github.com/arun-mosai/claude-code-slice-skills](https://github.com/arun-mosai/claude-code-slice-skills).
MIT. Three composable skills that turn a one-line feature idea into a
validated vertical slice (persona, journey, UI, API, data, security,
observability, AI, seed, tests) and then into working code.

This post is not about the skills. It's about the five load-bearing ideas
I found along the way that I think generalize beyond this stack.

---

## What "vertical slice" means here

When I say "slice" I mean **one persona's one job, end-to-end**.

Not a feature flag.
Not a stub behind a TODO.
Not a design doc that drifts from code the moment it ships.

A slice is: UI + API + migration + seed data + tests that together
demonstrate the feature works for a named human. Alex Chen, 31, Senior PM
at Flowroom, using Linear + Notion + Amplitude, literal quote: *"my
Chrome bookmarks bar is a graveyard of good intentions."*

That level of specificity is what makes the rest of the pipeline viable.

---

## Idea 1 — Force specificity where LLMs drift

Every interaction with Claude taught me where it drifts. So the skill
pins those exact spots:

- **Personas** must have age, tenure, a primary tool stack of ≥5 named
  tools, and a verbatim quote you could believe a human actually said.
- **Seed rows** must have recognizable names. Not `Customer 1`. Not
  `Feature A`. `Acme Corp`. `Wayne Enterprises`. The name has to be
  memorable enough to appear in the tests as an assertion.
- **Every screen** covers the same 7 canonical states
  (loading / empty / populated / partial / error / permission-denied /
  stale-data).
- **Every endpoint** cites the table it touches. You can't write an
  endpoint without pinning it to a row-level contract.
- **Banned phrases** — `seamless`, `powerful`, `leverage`, `unlock`. If
  the planner tries to slip marketing slop into the doc, the validator
  fails.

None of these are novel in isolation. The insight is that LLMs drift
*in the same places every time*, so specificity budgets should be spent
exactly there.

---

## Idea 2 — The Resolution Loop

Most AI-generated user journeys look like this:

```
1. User navigates to Page
2. User sees a list
3. User clicks back
```

It's a read-only product. Every time.

Phase 1 Batch B of `/slice` is what I call the **Resolution Loop**. It
forces four decisions before any UI is drawn:

- **Closure actions** — which domain verbs move this persona to "done"?
  (Derived from the state machine, not invented by the planner.)
- **Impact preview** — how does the UI show consequences before commit?
  Severity-matched: undo for idempotent writes, confirm dialog for
  irreversible transitions, side-panel diff for multi-field edits.
- **Cross-boundary routing** — which other services fan out when this
  verb fires? (Inspected from the codebase's module-boundary rules.)
- **Reversibility** — what does undo actually look like? Driven by
  whether the state transition is legal to reverse, not by a wishlist.

The result: slices that let the persona *act*, not just look.

This is the concept I think is most portable outside the skill context.
Any tool that generates UX flows should probably bake a version of this
in.

---

## Idea 3 — Atomic cycle commit

The full cycle — `/slice` → `/slice-resume` → `/slice-build` — is one
atomic unit of work. The skills commit **nothing** mid-cycle.

- Not per phase.
- Not per endpoint.
- Not "for safety."

State lives in two durable files (`PROGRESS.md` and `BUILD-PROGRESS.md`)
that survive `/clear`. The git working tree is the running diff.

Only after the 7-step verification passes **and** the user explicitly
says "commit" does the cycle produce git history.

**Why this matters:** a vertical slice is a contract. Migrations + API +
UI + events + tests all interlock. Partial commits create half-baked
review streams and bisection confusion. The atomic rule forces the
slice to be coherent before it enters history.

---

## Idea 4 — Blocker verification via shape, not prefix

Every slice has a ledger of outstanding blockers. When you claim a
blocker is `RESOLVED` or `WAIVED`, the implementation skill doesn't
trust the marker — it verifies live:

- The file the resolution pointer names **must exist**.
- The grep the resolution claims **must hit**.
- The migration the resolution names **must round-trip**.

The check keys on the `Status + Resolution:` *shape*, not a hardcoded
ID prefix. So it works across any taxonomy — `D-001`, `C-*`,
`GAP-42`, ad-hoc — because the shape is universal.

Markers rot. Code doesn't.

---

## Idea 5 — Project-stage calibration

Same skills, different ceremony bars. `.claude/project-stages.md`
encodes six lifecycle knobs:

- `migration_mutability`
- `migration_reversibility` (`.down.sql` gates)
- `schema_mirror_sync` (Docker init SQL)
- `data_migration_strategy`
- `rollout_safety` (feature flags)
- `observability_floor`

...across seven stages: `prototype-exploration` → `developing-mvp` →
`internal-alpha` → `private-beta` → `public-beta` →
`general-availability` → `regulated-production`.

You declare your stage in `CLAUDE.md`:

```markdown
**PROJECT_STAGE:** `developing-mvp` (last reviewed 2026-04-21)
```

Pre-production: `.down.sql` is optional, feature-flag ceremony skipped.
Regulated-production: audit trails gate every write, migrations must be
mutation-only with paired rollbacks.

Same skills, calibrated bar.

---

## How to try it

```bash
git clone https://github.com/arun-mosai/claude-code-slice-skills
cp -R claude-code-slice-skills/.claude/* /path/to/your-project/.claude/
cp -R claude-code-slice-skills/docs/slice-templates /path/to/your-project/docs/
```

Then in your project's `CLAUDE.md`:

```markdown
**PROJECT_STAGE:** `developing-mvp` (last reviewed 2026-04-21)
```

And in Claude Code:

```text
/slice A Customer Success Manager can flag a customer as at-risk and
get an auto-generated win-back playbook.
```

Watch it go.

---

## Honest caveats

This is a thin open-source extraction of a workflow used in a private
codebase. Expect rough edges where the generalization lost nuance.
`PORTING.md` documents stack assumptions (TypeScript / Postgres / Zod /
React defaults) and how to port to Django, Rails, Go, or Next.js.

Issues and PRs welcome. Especially:
- Additional project-stage rows (fintech? healthcare? regulated-EU?)
- Validator agents for new doc types
- Language ports

MIT license. No strings.

---

**Repo:** [github.com/arun-mosai/claude-code-slice-skills](https://github.com/arun-mosai/claude-code-slice-skills)

If you read this far and any of the five ideas resonates (or doesn't),
I'd genuinely like to hear about it. Issues or DMs.
```

---

## Publishing notes

- dev.to: cross-post to Hashnode using their "canonical URL" field pointing at the dev.to version (or vice versa — whichever you post first)
- Add to the GitHub repo's README footer: "Writeup: dev.to/..."
- Promote in the X thread + HN post's first comment
- Don't publish on a Friday — dev.to algorithm penalizes Friday posts
