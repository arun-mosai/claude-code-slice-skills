---
name: slice-resume
description: >
  Use when continuing a vertical slice that was stopped mid-build — context exhausted,
  user ran /clear, session ended, validator stuck, or manual pause. Reads PROGRESS.md +
  INTAKE.md from the slice directory and picks up at the exact documented checkpoint
  without re-asking Phase 1 questions. Companion to /slice. Uses the same sub-agent
  delegation pattern. Invoked as /slice-resume or /slice-resume <SLICE_ID>.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent
---

# /slice-resume — Continue a Paused Slice

You are resuming a vertical slice previously started via `/slice`. The slice directory contains `PROGRESS.md` (checkpoint state) and `INTAKE.md` (Phase 1 decisions). Trust those files.

> **Sub-agent pattern.** Resume uses the same sub-agent delegation as `/slice`: all slice doc writes go through spawned `Agent` calls; main thread only orchestrates. See `.claude/skills/slice/SKILL.md` § Execution Model + § Sub-Agent Prompt Template.
>
> **Checkpoint-aware.** After each completed phase, emit a Context Checkpoint block so the user can `/clear` again if needed. The slice survives any number of `/clear` → `/slice-resume` cycles because PROGRESS.md is durable.

## Absolute Rule — Atomic Cycle Commit

> `/slice-resume` never commits. It only writes slice docs (01-10, PROGRESS.md, INTAKE.md). The full `/slice` → `/slice-resume` → `/slice-build` cycle commits atomically only at the end, gated on user approval. See `.claude/skills/slice/SKILL.md` for the full rule.

## Efficiency Principle

Do what's necessary. Skip what's not. Never compromise quality.

- **Don't re-read** files that haven't changed since the last read. `PROGRESS.md` + `INTAKE.md` are authoritative for slice state — trust them unless there's concrete reason to doubt.
- **Don't re-run** validators, sub-agents, or phases whose output is still current. If the slice docs are unchanged since the last validator run, the verdict is unchanged.
- **Don't regenerate** already-written slice docs to fit a default shape. If a doc is correct and current, skip it. Resume the next open step, not the last one.
- **Don't re-ask** Phase 1 questions. `INTAKE.md` captured the answers; trust it. Only re-ask if the user explicitly says a decision changed.
- **Don't add** scope the task didn't ask for. No defensive re-validation, no "while we're here" cleanups.
- **Fast is a feature.** Unnecessary re-work burns context, tokens, and the user's time.

When a skill's default flow would re-do work already verified, override it with the surgical path and document the deviation in `PROGRESS.md`. **Quality gates stay in place** — the deviation is about _scope_, not _standards_.

## Step 1 — Select the slice

```bash
# Targeted resume
test -f docs/slices/<SLICE_ID>/PROGRESS.md

# Auto-discover in-progress slices
grep -lE "^status: in-progress$" docs/slices/*/PROGRESS.md 2>/dev/null
```

- **Zero in-progress** → tell user nothing to resume, suggest `/slice <one-liner>`. Stop.
- **Exactly one** → use it, announce SLICE_ID.
- **Multiple** → AskUserQuestion with one question listing `<slice_id>` + `current_step` + `last_updated` per option.
- **User gave SLICE_ID but it's completed / missing** → report state and stop. Do not overwrite.

## Step 2 — Load state (main thread reads minimum viable)

Read ONLY these on main thread:

1. `docs/slices/<SLICE_ID>/PROGRESS.md` — full file. Extract: `current_phase`, `current_step`, `next_action`, `validator_loops`, `validator_verdict`, `session_count`, Files Written list, Stop Reason, `slice_stage:` override (if present).
2. `docs/slices/<SLICE_ID>/INTAKE.md` — full file. All locked decisions live here.
3. `.claude/skills/slice/SKILL.md` — the phase definitions + sub-agent prompt template you'll be dispatching.
4. `CLAUDE.md` top section — resolve `PROJECT_STAGE:` (overridden by `slice_stage:` in PROGRESS.md if present). Emit the stage banner from `/slice` Step 0.25 before continuing. The resumed session must calibrate to the same stage the original `/slice` session used — if the project stage has advanced since the slice was last worked, flag to the user and ask whether to continue at the old stage or re-calibrate the slice docs.

**Do NOT read on main thread:** slice templates, reference examples, already-written slice docs, SLICE-BUILDER-PROMPT.md. The sub-agents will re-read anything they need. Keeping those off main thread is the whole reason this workflow can be cleared and resumed many times.

## Step 3 — Advance the session counter

Edit PROGRESS.md:
- `session_count` += 1
- `last_updated` = current ISO timestamp
- Append to Session History: `- <ISO> session <n>: resumed at <current_step>`

## Step 4 — Resume execution via sub-agents

Jump to the phase / sub-step named by `current_step`. Dispatch work per `/slice` SKILL.md's rules for that phase:

| current_step | What to dispatch |
|--------------|------------------|
| `2.1-persona` | Spawn Agent → write `01-PERSONA.md`. On return, proceed to `2.2-journey`. |
| `2.2-journey` | Spawn Agent → write `02-JOURNEY.md`. On return, emit Checkpoint (recommend /clear). |
| `3-contracts` | Spawn **3 Agents in one message** for 03/04/05. Then spawn reconciliation Agent. Then Checkpoint. |
| `4-supporting` | Spawn **5 Agents in one message** for 06/07/08/09/10. Then Checkpoint (recommend /clear). |
| `5.1-index` | Spawn Agent → write `00-SLICE-INDEX.md`. Then move to `5.2-validator`. |
| `5.2-validator` | Spawn `slice-validator-orchestrator`. Handle PASS/WARN/FAIL per `/slice` rules. |

Use the exact **Sub-Agent Prompt Template** from `.claude/skills/slice/SKILL.md`. Never write slice docs from the main thread.

### Resume rules
- **Never re-ask Phase 1 questions.** INTAKE.md has the answers.
- **Never regenerate a file in `Files Written`** unless PROGRESS.md Stop Reason explicitly flags it (e.g., validator FAIL on that doc). In that case, spawn a fix-up agent scoped to the blockers.
- **Respect locked decisions.** If INTAKE.md says `assignee_user_id` column, do not propose an alternative in any dispatched agent's prompt.
- **Rewrite PROGRESS.md at every phase boundary** (sub-agents also update it as their final step — verify after they return).
- **Emit Context Checkpoint** after each completed phase so the user can `/clear` safely.

## Step 5 — Validator-stuck resumes (special case)

If `current_step == 5.2-validator` AND `validator_verdict == FAIL` AND `validator_loops >= 3`:

- Do **not** loop again blindly.
- Read PROGRESS.md Stop Reason for stubborn blockers.
- Offer three paths via AskUserQuestion:
  1. **Targeted fix** — main thread spawns a scoped fix-up agent per blocker (describe the known-hard ones and let the user pick which to try).
  2. **Relax a validator rule** — rare; document why in `00-SLICE-INDEX.md` § Risks.
  3. **Abandon** — mark PROGRESS.md `status: abandoned` with reason; user can rerun `/slice` with narrower scope later.

## Step 6 — Completion

On Phase 5 PASS:
1. Main thread sets PROGRESS.md `status: completed`, `completed_at`, `validator_verdict: PASS` (or WARN).
2. Emit the `SLICE <ID> — COMPLETE` summary from `/slice` Step 3, including `Sessions used: <session_count>`.

## What this skill deliberately does NOT do

- **Does not re-do Phase 1.** If INTAKE.md is missing, report: "Slice is too early-stage to resume — rerun `/slice <one-liner>`."
- **Does not invent new decisions.** All ambiguity resolution lives in INTAKE.md.
- **Does not change SLICE_ID.** Keyed on the existing ID.
- **Does not skip ahead.** If `current_phase: 2` and `03-UI-SPEC.md` somehow exists from a partial prior run, still complete `02-JOURNEY.md` first before touching Phase 3.
- **Does not write slice docs on main thread.** All writes go through sub-agents.

## Reference

- Main skill: `.claude/skills/slice/SKILL.md`
- Execution model: `/slice` § Execution Model
- Sub-agent template: `/slice` § Sub-Agent Prompt Template
- Checkpoint format: `/slice` § Context Checkpoint Protocol
- Progress schema: `/slice` § Progress Tracking Protocol
- Slice contract: `docs/slice-templates/SLICE-BUILDER-PROMPT.md`
- Reference filled example: `docs/slices/00-example-bookmark-v1/`
