---
# =============================================================================
# UI SPEC — every screen, every state, every interaction. Kills happy-path bias.
# Rule: all 7 states documented per screen; every interaction cites an endpoint.
# =============================================================================
slice_id: <from 00>                           # [REQUIRED]
persona_id: <from 01>                         # [REQUIRED]
journey_id: <from 02>                         # [REQUIRED]
status: draft                                 # draft | ready
design_system:
  tokens_file: apps/web/src/lib/themes-v2.ts  # where design tokens live
  component_lib: radix-ui | custom            # no new colors / spacing scales
  motion_file: apps/web/src/lib/motion.ts
breakpoints:
  mobile: 375
  tablet: 768
  desktop: 1440
screens_count: <n>                            # [REQUIRED]
---

# UI Spec: {{slice_name}}

## 1. Screen Inventory  [REQUIRED]

| screen_id | Route | Page component file | Journey steps served | New vs Modified |
|---|---|---|---|---|
| UI-001 | `/…` | `apps/web/src/pages/…` | STEP-001, STEP-002 | new |
| UI-002 | `/…` | `apps/web/src/pages/…` | STEP-003 | modified |

---

## 2. Per-Screen Spec  [REQUIRED — repeat §2.1–2.10 for each screen]

### 2.1 Screen: `UI-001` — {{screen_name}}

#### Header
| Field | Value |
|---|---|
| Route | `/…` |
| Page component | `apps/web/src/pages/…/<file>.tsx` |
| Access control | authenticated + role `<role>` + permission `<perm>` |
| Journey steps served | STEP-001, STEP-002 |
| Primary API endpoints (from 04) | `ENDPOINT-001`, `ENDPOINT-002` |
| Store slice (Zustand) | `apps/web/src/stores/<file>.ts` |
| TanStack Query keys | `['<key>', orgId, …]` |

#### 2.2 Layout & Scaffold  [REQUIRED]

```
┌─────────────────────────────────────────────┐
│ AppShell header (nav, user menu, search)     │
├───┬─────────────────────────────────────────┤
│ S │                                         │
│ i │         Main content area               │
│ d │                                         │
│ e │                                         │
│ N │                                         │
│ a │                                         │
│ v │                                         │
└───┴─────────────────────────────────────────┘
```

Grid: 12-col desktop, 6-col tablet, single column mobile.
Max content width: `<px>`.
Page structure: `<Header> <FilterBar> <DataGrid> <Pagination>` (name real components).

#### 2.3 State Matrix  [REQUIRED — all 7 states]

| State | Trigger | Visual | Primary copy | Next action |
|---|---|---|---|---|
| **initial** | First render, no data fetched yet | skeleton grid | — | auto-fetch |
| **loading** | Fetch in-flight | skeleton with shimmer | "Loading…" (sr-only) | — |
| **empty** | Fetch succeeded, 0 rows | illustration + CTA | "No <things> yet. Create your first." | primary button `<label>` |
| **populated** | Fetch succeeded, ≥1 row | data grid | — | row click → detail |
| **error** | Fetch 5xx / network | red banner | "Something went wrong. <recovery>" | retry button |
| **partial/stale** | Cache hit but revalidating, or some rows failed | dim + spinner on stale rows | "Showing last known data" | — |
| **permission-denied** | 403 from gateway | lock illustration | "You don't have access to <thing>. Contact <admin>" | link to docs |

Repeat this matrix for EVERY screen. Missing state = validator FAIL.

#### 2.4 Component Inventory  [REQUIRED]

| Component | Source | Props (key ones) | Purpose |
|---|---|---|---|
| `<Button variant="primary">` | Radix | onClick, disabled, loading | create action |
| `<DataGrid>` | custom `apps/web/src/ui-upgrade/DataGrid.tsx` | columns, rows, onRowClick | tabular display |
| `<EmptyState>` | custom | illustration, title, cta | empty state |

No new colors, spacing, or shadow tokens. If you need one, it's a design-system change first.

#### 2.5 Interaction Map  [REQUIRED]

Every input-output pair. Every API call cites an endpoint_id from 04.

| Trigger | Source | Action | API call | Optimistic? | Error recovery |
|---|---|---|---|---|---|
| click | `<Button id="create">` | open modal | — | — | — |
| submit | `<CreateForm>` | POST | `ENDPOINT-001` | yes, rollback on 4xx/5xx | toast + keep modal open |
| keydown `Cmd+K` | page | open command palette | — | — | — |
| row click | `<DataGrid>` | navigate detail | — | — | — |
| hover row | `<DataGrid>` | show actions | — | — | — |
| drag-drop | `<DataGrid>` | reorder | `ENDPOINT-002` | yes | toast + revert order |
| right-click | `<DataGrid>` | context menu | — | — | — |
| copy (⌘C) | selected row | clipboard | — | — | — |
| undo (⌘Z) | after mutation | revert | `ENDPOINT-003` | — | toast |

#### 2.6 Copy Deck  [REQUIRED — fenced YAML]

```yaml
copy:
  title: "<page title>"
  subtitle: "<optional subtitle>"
  buttons:
    primary: "<label>"
    secondary: "<label>"
  empty_state:
    title: "<heading>"
    body: "<body copy>"
    cta: "<button label>"
  errors:
    generic: "<fallback error copy>"
    not_found: "<404 copy>"
    forbidden: "<403 copy>"
    validation:
      required_field: "<field> is required"
  toasts:
    create_success: "<thing> created"
    update_success: "<thing> updated"
    delete_success: "<thing> deleted"
  confirmations:
    delete: "Delete <thing>? This can't be undone."
  tooltips:
    <action>: "<tooltip>"
```

#### 2.7 Accessibility  [REQUIRED]

- **Tab order:** (explicit — skip-to-main → header → sidenav → filter bar → grid → pagination → footer)
- **ARIA roles:** (list any non-default role, e.g., `role="grid"` on data grid)
- **ARIA labels:** (every icon-only button, every region)
- **Contrast:** verified ≥ 4.5:1 text / ≥ 3:1 UI on both light/dark via tokens
- **Keyboard-only completion:** (yes — golden path completable without mouse)
- **Live regions:** (`aria-live="polite"` on toasts, `aria-live="assertive"` on errors)
- **Focus management:** (modal traps focus, esc closes, focus returns to trigger)
- **Screen reader announcements:** (async mutations announced: "Created X")

#### 2.8 Motion Spec  [REQUIRED]

| Transition | Duration | Easing | Reduced-motion fallback |
|---|---|---|---|
| Modal enter | 150ms | ease-out | no transform, fade only |
| List item add | 200ms | spring | fade only |
| Toast enter/exit | 180ms | ease-in-out | fade only |
| Drag reorder | 120ms | ease-out | instant |

All animations MUST respect `prefers-reduced-motion: reduce`.

#### 2.9 Responsive Behavior  [REQUIRED]

| Breakpoint | Layout change |
|---|---|
| `< 768` (mobile) | Sidenav becomes bottom tab bar; data grid becomes card list |
| `768-1279` (tablet) | Sidenav collapses to icon rail; grid keeps 4 cols |
| `≥ 1280` (desktop) | Full layout |

#### 2.10 Real Data Example  [REQUIRED]

One literal populated state using seed row IDs from 09-SEED-DATA.md.

```json
{
  "rows": [
    { "id": "seed-row-001", "name": "Acme Corp", "status": "active", "updated_at": "2026-04-15T..." },
    { "id": "seed-row-002", "name": "Wayne Industries", "status": "active", "updated_at": "2026-04-14T..." }
  ]
}
```

---

## 3. Shared Primitives Used Across Screens  [REQUIRED]

If multiple screens share a new component, spec it here once.

| Component | Source | Used on screens | Props | Notes |
|---|---|---|---|---|
| | | | | |

## 4. Dark Mode / Theme  [REQUIRED]

- Token file: `apps/web/src/lib/themes-v2.ts`
- Any new tokens needed? If yes, list them + why existing tokens don't suffice.
- Dark mode verified on all screens: yes | no

## 5. Design Assets  [OPTIONAL]

| Asset | Path / URL | Status |
|---|---|---|
| Figma frames | | |
| Screenshots (light) | `screenshots/<slice>/…` | |
| Screenshots (dark) | `screenshots/<slice>/…` | |

## 6. Validator Will Fail If …

- Any screen is missing any of the 7 states
- Any interaction row is missing an endpoint_id (when the interaction triggers an API call)
- Copy deck has `TODO` or placeholder text
- New color/spacing/shadow tokens added without justification
- Accessibility section skipped or says "standard a11y"
- Motion spec missing `prefers-reduced-motion` handling
- Real data example uses fake/round-number IDs instead of seed row IDs
