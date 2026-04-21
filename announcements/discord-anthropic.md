# Anthropic Discord — #showcase or #claude-code

**Venue:** Anthropic Discord server → appropriate showcase / tools channel
**Format:** Short casual message + link. Discord rewards brevity.

---

## Primary message

```
Just open-sourced 3 Claude Code skills I use to turn a one-line feature
idea into a complete vertical slice (persona → journey → UI + API + DB
→ tests → code).

github.com/arun-mosai/claude-code-slice-skills

A few things in there that might be more interesting than the skills
themselves:

• **Resolution Loop** — forces 4 decisions (closure actions, impact
  preview, cross-boundary routing, reversibility) before UI exists, so
  slices let the persona *act* not just look
• **Atomic cycle commit** — nothing commits mid-cycle; state lives in
  PROGRESS.md so you can /clear freely
• **Project-stage calibration** — 7 lifecycle stages × 6 knobs; same
  skills enforce different bars based on CLAUDE.md declaration
• **Blocker verification** — every RESOLVED marker verified live
  against its Resolution pointer (shape-based, works across any ID
  taxonomy)

TypeScript + Postgres + Zod by default, PORTING.md for other stacks. MIT.

Would love feedback, especially on whether the Resolution Loop concept
lands or feels overengineered. And curious if anyone has run similar
validator-agent orchestration at scale.
```

## Follow-up message (if people ask "how long does /slice take?")

```
Design phase (/slice → 11 validated docs): ~15-25 min of wall-clock,
depending on how complete your CLAUDE.md is and how many resolution-
loop iterations the intake triggers.

Build phase (/slice-build): varies wildly by slice size. Simple CRUD
slice = ~10 min. Anything with AI surfaces, cross-service events, or
non-trivial state machines = 30-60 min.

Both phases are /clear-safe and resumable, so you can step away and
come back.
```

## If someone asks about Anthropic Skills vs this

```
Orthogonal. These are Claude Code *skills* (project-local markdown
files that Claude Code invokes via slash commands). The Anthropic
Skills product is a different abstraction — application-level skill
packaging.

These work today with any Claude Code install. No API changes needed.
```

---

## Channel etiquette

- Post in #showcase (primary) or #claude-code (secondary) — NOT #general
- Don't @ Anthropic staff; they lurk these channels
- Reply in-thread to comments, not top-level
- Expect 2-3 substantive replies over 24h, not a flood
