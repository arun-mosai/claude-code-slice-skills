# Project Stages — Lifecycle Calibration for Slice Skills

**What this file is.** A lookup table of project-lifecycle stages and the engineering ceremony each stage warrants. The slice skills (`/slice`, `/slice-resume`, `/slice-build`) read this file to decide which gates to enforce vs skip. Change is expensive post-production; ceremony exists to make expensive change safe. Pre-production, that ceremony is premature.

**How the declaration works.** `CLAUDE.md` carries one line: `**PROJECT_STAGE:** <name> (last reviewed <YYYY-MM-DD>)`. The value is a key into the `## Stages` section below. Per-slice override: `PROGRESS.md` frontmatter can carry `slice_stage: <name>` for slices touching a subsystem at different maturity than the project default.

**How agents read this.** Stage name → row in `## Stages` → the six `knobs:` lines. Every skill gate tagged `[conditional: <knob>=<value>]` is evaluated against the loaded profile. Absent/unknown stage → stop and ask (never guess). Absent knob in a known stage → assume the strictest value for that knob (fail safe).

**Updating this file.** Add a new stage by copying an adjacent row and adjusting knobs. Add a new knob only when a real skill gate needs to consult it — never add speculative knobs. Never remove a stage that has been referenced by any slice's `slice_stage:` override without first migrating those slices.

---

## Knobs

Six knobs, each a discrete string value. Ordering of values goes from *least ceremony* → *most ceremony* for that dimension.

| Knob | Values (loose → strict) | What it gates in slice skills |
|---|---|---|
| `migration_mutability` | `editable` → `immutable` | `editable` = edit past migration files in place when design changes; `immutable` = once shipped to any shared env, a migration is history; corrections go in new forward migrations. Gates: `/slice-resume` warning when a past migration is modified; `/slice-build` preflight that migration filenames haven't changed since last session. |
| `migration_reversibility` | `not-required` → `recommended` → `required-tested` | Whether `.down.sql` must exist, and whether it must be round-trip verified. Gates: Step 2 `.down.sql` existence preflight; Step 2.5 round-trip verifier; 6.1 reversibility check. |
| `schema_mirror_sync` | `hand-synced` → `derived-from-migrations` | How `packages/database/postgresql/schema.sql` + `infrastructure/docker/postgres/init/02-schema.sql` are kept in agreement with migrations. `hand-synced` = hand-edit both mirrors alongside migrations (CLAUDE.md common-mistake #6 still applies). `derived-from-migrations` = mirrors are regenerated from cumulative migration replay; hand-editing is forbidden. Gates: Step 2 mirror-drift preflight still blocks in both modes — only the expected *mechanism* differs. |
| `data_migration_strategy` | `reset-tolerant` → `backfill-tolerant` → `zero-downtime` | Assumptions about existing row state when schema changes. `reset-tolerant` = seed always runs with `--reset`; NOT NULL column adds may ship as one migration. `backfill-tolerant` = existing rows must be backfilled but breakage tolerated. `zero-downtime` = expand-contract dance required (nullable → backfill → NOT NULL in a second pass); online DDL patterns (e.g., `CREATE INDEX CONCURRENTLY`) required. Gates: 6.1 backfill-first blocker on NOT NULL column adds; 05-DATA contract checks during `/slice`. |
| `rollout_safety` | `ship-direct` → `flagged` → `canary+flag` → `change-control` | Whether feature flags / canary deploys / formal change tickets are required. Gates: 6.4 AI surface flag requirement; 6.5 UI layer flag wiring; `/slice` Phase 4 Agent 4.3 flag-section in 08-AI-SURFACE.md. |
| `observability_floor` | `logs-only` → `metrics+runbooks` → `slo+alerts+dashboards` → `full-audit-trail` | Minimum observability ceremony. `logs-only` = structured logs; no dashboards or alerts required. `metrics+runbooks` = Prometheus metrics + per-error-code runbooks. `slo+alerts+dashboards` = Grafana dashboard + alerts wired to runbooks. `full-audit-trail` = compliance-grade audit events and retention. Gates: 6.3 events/observability sub-phase; `/slice` Phase 4 Agent 4.2 requirements for 07-OBSERVABILITY.md. |

**Informational knobs (documented; not yet gated in slice skills).** Add gates only when a specific need arises.
- `api_compatibility`: `break-freely` → `heads-up-breaks` → `deprecate-then-remove` → `forever-stable`
- `test_rigor`: `smoke` → `unit+integration` → `+contract+spec` → `+mutation+chaos`
- `docs_rigor`: `adhoc` → `slice-only` → `+runbooks+architecture` → `+compliance-artifacts`

---

## Stages

Stages are ordered by ordinal (earliest maturity → strictest). A project advances through stages; it does not skip them without deliberate acceleration. The ordinal is informational — skills key on the name, not the number.

### `prototype-exploration` (ordinal 1)

Throwaway code proving a capability. Nobody uses it yet. Schema may live entirely in `schema.sql` without migrations. Tests are optional.

```
knobs:
  migration_mutability:       editable
  migration_reversibility:    not-required
  schema_mirror_sync:         hand-synced
  data_migration_strategy:    reset-tolerant
  rollout_safety:             ship-direct
  observability_floor:        logs-only
```

Notes: A slice at this stage should be small enough that it mostly *skips* skill phases: often `[SKELETON]` mode is right. If this stage is declared, reconsider whether the slice skills are appropriate — a spike branch with ad-hoc scripts may be simpler.

### `developing-mvp` (ordinal 2)

Building toward first user; team-only. Seeded mock data only. Migrations exist as *living delivery scripts* so co-workers / CI / your long-running local dev DB can forward-upgrade without `--reset`. Files are editable; if a column's shape changes mid-slice, edit the migration in place and re-run against a reset DB rather than stacking a second migration on top.

```
knobs:
  migration_mutability:       editable
  migration_reversibility:    not-required
  schema_mirror_sync:         hand-synced
  data_migration_strategy:    reset-tolerant
  rollout_safety:             ship-direct
  observability_floor:        logs-only
```

Notes:
- `.down.sql` files not required. Round-trip verification is skipped. Rollback path is "drop + reseed."
- Schema-mirror drift **is still gated** — a `docker compose up` from the docker-init file must produce the same DB as a forward-migrated env. This is CLAUDE.md common-mistake #6 and it's stage-independent.
- CHECK constraints added retroactively go in the migration that introduced the column (edit in place), not a follow-up migration.
- Feature flags only for genuinely risky business logic, not as a rollout mechanism.
- Backwards-compat shims are explicitly forbidden — they're cargo at this stage.

### `internal-alpha` (ordinal 3)

Company dogfood. Team + colleagues use the product on daily work. Data loss in dogfood DBs is annoying but not catastrophic. Basic observability emerges so the team can debug its own usage.

```
knobs:
  migration_mutability:       editable
  migration_reversibility:    recommended
  schema_mirror_sync:         hand-synced
  data_migration_strategy:    backfill-tolerant
  rollout_safety:             flagged
  observability_floor:        metrics+runbooks
```

Notes:
- `.down.sql` becomes recommended but not gated — prefer writing them, skip only when clearly wasteful.
- Non-null column adds start needing a thought about existing rows; a one-shot backfill in the same migration is often fine.
- Feature flags start to matter for shipping risky changes.

### `private-beta` (ordinal 4)

Named external customers under NDA or design-partner arrangement. Data loss is now a real incident. Migrations become immutable once they've landed in a shared env. `.down.sql` files are required. Observability is real because you need to see what customers are doing without asking them.

```
knobs:
  migration_mutability:       immutable
  migration_reversibility:    required-tested
  schema_mirror_sync:         hand-synced
  data_migration_strategy:    backfill-tolerant
  rollout_safety:             flagged
  observability_floor:        slo+alerts+dashboards
```

Notes:
- "Immutable once shipped" means `/slice-resume` should warn if a past migration file is about to be modified.
- Round-trip verification becomes a Step 2.5 gate.
- Alerts must link to runbooks.

### `public-beta` (ordinal 5)

Open signup; informal SLAs. Scaling concerns emerge. Schema changes increasingly need expand-contract because a migration lock on a large table is a real outage. Canary deploys become standard.

```
knobs:
  migration_mutability:       immutable
  migration_reversibility:    required-tested
  schema_mirror_sync:         derived-from-migrations
  data_migration_strategy:    zero-downtime
  rollout_safety:             canary+flag
  observability_floor:        slo+alerts+dashboards
```

Notes:
- `schema.sql` transitions to derived-from-migrations. The hand-editing ergonomics of earlier stages stop being safe at this scale.
- NOT NULL column adds require the expand-contract dance enforced as a 6.1 gate.

### `general-availability` (ordinal 6)

Paying customers with contractual SLAs. Zero-downtime migrations are the norm. Deprecation windows for any API shape change. Full observability with SLO tracking.

```
knobs:
  migration_mutability:       immutable
  migration_reversibility:    required-tested
  schema_mirror_sync:         derived-from-migrations
  data_migration_strategy:    zero-downtime
  rollout_safety:             canary+flag
  observability_floor:        slo+alerts+dashboards
```

Notes:
- The main difference from `public-beta` is in the informational knobs: `api_compatibility: deprecate-then-remove`, `test_rigor: +contract+spec`, `docs_rigor: +runbooks+architecture`.
- If you've been running at `public-beta` with real SLAs for > 6 months, you're effectively at GA — bump the declaration.

### `regulated-production` (ordinal 7)

GA plus formal change control. Every migration needs an approved change ticket. Privacy / audit / financial compliance layers on top (HIPAA, SOX, PCI, SOC 2 Type II).

```
knobs:
  migration_mutability:       immutable
  migration_reversibility:    required-tested
  schema_mirror_sync:         derived-from-migrations
  data_migration_strategy:    zero-downtime
  rollout_safety:             change-control
  observability_floor:        full-audit-trail
```

Notes:
- `change-control` rollout gate means `/slice-build` should refuse to proceed without a linked change-ticket ID in the slice contract.
- `full-audit-trail` observability means every state-changing write must have an audit-table row with a retention policy meeting the relevant regime.

---

## Orthogonal Modifiers (optional)

Modifiers stack on top of a stage. Document as `PROJECT_STAGE_MODIFIERS: [<mod>, ...]` in `CLAUDE.md` if used.

- **`safety-critical`** — Medical, financial, or autonomous-systems context. Certain invariants (dose calculation, transaction idempotency) can't be loosened even in early stages. Effect: *tightens* specific ceremony even when the base stage is loose.
- **`research-code`** — Academic-paper-adjacent. Reproducibility matters more than operational ceremony. Effect: emphasizes deterministic seeds, frozen dependency versions, and test-data provenance.
- **`airgapped`** — On-premises deploy. No continuous delivery; migrations ship in versioned releases. Effect: stricter upgrade-path testing across release boundaries; `.down.sql` becomes required earlier.

Modifiers are not required today; they're documented here so future skill changes can consume them.

---

## Default Behavior for Unknown Values

- **Stage name not in this file** → slice skill stops and asks the user to define the stage here (do not guess from the name).
- **Knob absent from a defined stage** → assume the strictest value for that knob (fail safe); log a warning suggesting the stage be updated.
- **`PROJECT_STAGE:` line missing from CLAUDE.md** → slice skill stops and asks the user to declare a stage (the correct response is rarely to assume MVP — it's to ask).
- **`last reviewed` date > 90 days old** → slice skill emits a one-line advisory at session start but does not block. Advisory says: "Stage declaration last reviewed <date>; confirm still accurate or update." The user can silence by re-dating the line.

---

## Consumer Contract for Slice Skills

When a skill reads this file, it must:

1. Load the project-level stage from `CLAUDE.md`.
2. If the current slice's `PROGRESS.md` has `slice_stage:`, override with that.
3. Emit a one-time banner at session start showing the stage + the six knobs.
4. For each gate annotated `[conditional: <knob>=<value>]`: evaluate against the loaded profile; if condition is met, run the gate; if not, print `⊘ skipped (stage=<name>, <knob>=<actual-value>)` inline and proceed.
5. For absolute gates: run unconditionally; phase-ignorance is not an option.

Gate annotations in SKILL.md files use the shorthand `[conditional: knob=value]` (condition to *run*, not to skip). `[conditional: knob!=value]` inverts. `[absolute]` is explicit where the distinction matters.

---

## Current Declaration

See `CLAUDE.md` top section for the live `PROJECT_STAGE:` line.
