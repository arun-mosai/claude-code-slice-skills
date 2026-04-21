---
slice_id: SLICE-bookmark-v1
persona_id: PERSONA-pm-alex-v1
journey_id: JOURNEY-bookmark-feature-v1
status: ready
design_system:
  tokens_file: apps/web/src/lib/themes-v2.ts
  component_lib: radix-ui
  motion_file: apps/web/src/lib/motion.ts
breakpoints:
  mobile: 375
  tablet: 768
  desktop: 1440
screens_count: 2
---

# UI Spec: Feature Bookmarks

## 1. Screen Inventory

| screen_id | Route | Page component | Journey steps | New/Modified |
|---|---|---|---|---|
| UI-001 | `/features` | `apps/web/src/pages/features/FeatureListPage.tsx` | STEP-001, STEP-002, STEP-003, STEP-006 | modified (add bookmark column + icon) |
| UI-002 | (sidebar across all routes) | `apps/web/src/components/app-shell/BookmarkSidebar.tsx` | STEP-006, STEP-007, STEP-008 | new |

---

## 2. Per-Screen Spec

### 2.1 UI-001 — Feature List Page (modified)

| Field | Value |
|---|---|
| Route | `/features` |
| Component | `apps/web/src/pages/features/FeatureListPage.tsx` |
| Access control | authenticated + permission `product:feature:read` |
| Journey steps | STEP-001, STEP-002, STEP-003, STEP-006 |
| Primary APIs | existing `GET /api/v1/features` + new ENDPOINT-001 (POST bookmark), ENDPOINT-003 (DELETE bookmark) |
| Store | `apps/web/src/stores/bookmarks.store.ts` (new; Zustand slice holds bookmark set) |
| TanStack keys | `['bookmarks', orgId, userId]` (shared with UI-002) |

#### Layout & Scaffold

Existing three-pane app shell (header + left-nav + main). Feature list already renders as a DataGrid. This slice adds a **bookmark column** (leftmost, icon-only, 32px wide, sticky) to each row.

#### State Matrix (for the bookmark icon specifically; page-level states are pre-existing)

| State | Trigger | Visual | Copy | Next action |
|---|---|---|---|---|
| **initial** | page render, bookmarks not yet fetched | outline icon dim | — | auto-fetch from `['bookmarks', orgId, userId]` |
| **loading** | fetch in-flight | outline icon with subtle pulse | — | — |
| **empty** | user has 0 bookmarks | outline icons on all rows | — | click to add |
| **populated** | fetch done | outline OR filled icon per row | tooltip "Bookmark" / "Bookmarked" | click toggles |
| **error** | bookmarks fetch failed | outline icon dim + warning dot | toast "Couldn't load your bookmarks" | retry link in toast |
| **partial/stale** | optimistic update in-flight | icon animating (outline→filled) | sr-only "Saving" | — |
| **permission-denied** | role lacks `product:bookmark:write` | icon hidden entirely | — | — (silent degrade) |

#### Component Inventory

| Component | Source | Props | Purpose |
|---|---|---|---|
| `<BookmarkButton>` | new — `apps/web/src/ui-upgrade/BookmarkButton.tsx` | `featureId`, `isBookmarked`, `onToggle` | icon button in each grid row |
| `<DataGrid>` | existing — `apps/web/src/ui-upgrade/DataGrid.tsx` | `columns[]`, add bookmark column as first | unchanged |
| `<Tooltip>` | Radix | `content` | hover hint |
| `<Toast>` | existing | `variant: error|success` | failure feedback |

#### Interaction Map

| Trigger | Source | Action | API | Optimistic? | Error recovery |
|---|---|---|---|---|---|
| click `<BookmarkButton>` (outline) | row | POST bookmark | ENDPOINT-001 | yes — fill immediately, rollback on 4xx/5xx | icon reverts + toast "Couldn't bookmark" with retry |
| click `<BookmarkButton>` (filled) | row | DELETE bookmark | ENDPOINT-003 | yes — unfill immediately, rollback on 4xx/5xx | icon re-fills + toast |
| keydown `B` when row focused | row | toggle bookmark | ENDPOINT-001/003 | yes | same as click |
| hover `<BookmarkButton>` | row | show tooltip | — | — | — |

#### Copy Deck

```yaml
copy:
  tooltips:
    bookmark:   "Bookmark (B)"
    bookmarked: "Bookmarked — click to remove"
  toasts:
    create_success: "Bookmarked"
    delete_success: "Bookmark removed"
    create_error:   "Couldn't bookmark that feature"
    delete_error:   "Couldn't remove bookmark"
    limit_hit:      "You've hit 50 bookmarks — remove one to add another"
  aria:
    button_outline: "Bookmark {featureName}"
    button_filled:  "Remove bookmark on {featureName}"
    async_saving:   "Saving bookmark"
    async_saved:    "Bookmark saved"
```

#### Accessibility

- Tab order: existing grid row → bookmark button → other row actions (no reorder).
- Icon button has `aria-label` (see copy deck).
- `aria-pressed` reflects bookmarked state.
- Live region `<div aria-live="polite">` announces "Saving"/"Saved".
- Contrast: icon outline uses `--color-muted-foreground` (≥ 4.5:1); filled uses `--color-primary`.
- Keyboard: `B` bookmarks focused row (bound on `<DataGrid>`); `Escape` cancels in-flight toast.

#### Motion Spec

| Transition | Duration | Easing | Reduced-motion |
|---|---|---|---|
| outline → filled (bookmark add) | 180ms | ease-out | instant fill (no scale) |
| filled → outline (remove) | 120ms | ease-in | instant |
| toast enter/exit | 180ms | ease-in-out | fade only |

`prefers-reduced-motion: reduce` respected via `motion.ts` helper.

#### Responsive

| BP | Change |
|---|---|
| `< 768` | bookmark column becomes a row-swipe action (iOS-style); icon hidden inline |
| `768-1279` | icon-only column, 32px |
| `≥ 1280` | icon-only column, 32px + keyboard hint in tooltip |

#### Real Data Example

Using seed rows from 09:

```json
{
  "features": [
    { "id": "aaaaaaa1-...001", "name": "SSO Integration",    "status": "IN_PROGRESS", "isBookmarked": false },
    { "id": "aaaaaaa1-...002", "name": "Bulk Export to CSV",  "status": "PLANNED",     "isBookmarked": false },
    { "id": "aaaaaaa1-...003", "name": "Presence Indicators", "status": "PROPOSED",    "isBookmarked": true },
    { "id": "aaaaaaa1-...004", "name": "Dark Mode",           "status": "IN_QA",       "isBookmarked": false },
    { "id": "aaaaaaa1-...005", "name": "Better Onboarding Tooltips", "status": "SHIPPED", "isBookmarked": false }
  ]
}
```

---

### 2.2 UI-002 — BookmarkSidebar (new, global)

| Field | Value |
|---|---|
| Route | rendered in `<AppShell>`, visible on every authenticated route |
| Component | `apps/web/src/components/app-shell/BookmarkSidebar.tsx` |
| Access control | authenticated + permission `product:bookmark:read` (hidden if missing) |
| Journey steps | STEP-006, STEP-007, STEP-008 |
| Primary APIs | ENDPOINT-002 (list), ENDPOINT-003 (remove inline) |
| Store | `apps/web/src/stores/bookmarks.store.ts` (shared with UI-001) |
| TanStack keys | `['bookmarks', orgId, userId]` (staleTime: 30s; invalidated on POST/DELETE) |

#### Layout & Scaffold

Collapsible right-side panel, 280px wide expanded, 40px collapsed. Toggle button in app header. Collapsed state shows icon with count badge.

#### State Matrix

| State | Trigger | Visual | Copy | Next action |
|---|---|---|---|---|
| **initial** | app loads, first render | skeleton 3 rows | — | auto-fetch |
| **loading** | fetch in-flight | skeleton shimmer | — | — |
| **empty** | fetch done, 0 bookmarks | illustration (bookmark icon) + CTA | "No bookmarks yet. Click the bookmark icon on any feature to pin it here." | — |
| **populated** | 1+ bookmarks | list of rows, newest first | — | click row → navigate; click × → remove |
| **error** | fetch failed | error icon + retry button | "Couldn't load bookmarks" | retry |
| **partial/stale** | mutation in-flight | item animating in/out | sr-only "Updating" | — |
| **permission-denied** | missing `product:bookmark:read` | panel not rendered | — | — |

#### Component Inventory

| Component | Source | Props | Purpose |
|---|---|---|---|
| `<BookmarkSidebar>` | new | — (uses store) | container |
| `<BookmarkItem>` | new | `id`, `featureName`, `featureStatus`, `createdAt`, `onRemove` | row |
| `<FeatureStatusChip>` | existing | `status` | colored status dot |
| `<EmptyState>` | existing | — | empty panel |

#### Interaction Map

| Trigger | Source | Action | API | Optimistic? | Error recovery |
|---|---|---|---|---|---|
| click sidebar toggle | header button | expand/collapse | — | — | — |
| click `<BookmarkItem>` | row body | navigate to `/features/:id` | — | — | — |
| click `×` on row | row | DELETE bookmark | ENDPOINT-003 | yes, remove row immediately | re-insert row + toast |
| keyboard `⌘B` | anywhere | toggle sidebar | — | — | — |

#### Copy Deck

```yaml
copy:
  header: "Bookmarks"
  empty:
    title:  "No bookmarks yet"
    body:   "Click the bookmark icon on any feature to pin it here."
  toasts:
    remove_success: "Bookmark removed"
    remove_error:   "Couldn't remove bookmark"
  aria:
    toggle:        "Toggle bookmarks panel (⌘B)"
    remove_item:   "Remove bookmark on {featureName}"
```

#### Accessibility

- Sidebar is a landmark region: `<aside aria-label="Bookmarks">`.
- Toggle button has `aria-expanded` + `aria-controls="bookmark-sidebar"`.
- Focus trap when expanded on mobile (< 768 overlay).
- Each item: `<a>` for navigation (not `<div onClick>`).
- Remove button separate from link; stops event propagation so `×` doesn't navigate.

#### Motion

| Transition | Duration | Easing | Reduced-motion |
|---|---|---|---|
| expand/collapse | 220ms | ease-in-out | instant |
| item add | 200ms | spring | fade only |
| item remove | 160ms | ease-in | fade only |

#### Responsive

| BP | Behavior |
|---|---|
| `< 768` | full-screen overlay; toggle from bottom-right FAB |
| `768-1279` | 240px panel |
| `≥ 1280` | 280px panel, keyboard hint visible |

#### Real Data Example

Sidebar after Alex bookmarks "SSO Integration" (the new one) on top of seed's pre-existing "Presence Indicators":

```json
{
  "bookmarks": [
    { "id": "bbbbbbb1-0000-0000-0000-000000000042",
      "feature_id": "aaaaaaa1-0000-0000-0000-000000000001",
      "feature_name": "SSO Integration",
      "feature_status": "IN_PROGRESS",
      "created_at": "2026-04-17T14:03:12.441Z" },
    { "id": "bbbbbbb1-0000-0000-0000-000000000001",
      "feature_id": "aaaaaaa1-0000-0000-0000-000000000003",
      "feature_name": "Presence Indicators",
      "feature_status": "PROPOSED",
      "created_at": "2026-04-16T10:00:00.000Z" }
  ]
}
```

---

## 3. Shared Primitives

| Component | Used on | Notes |
|---|---|---|
| `<BookmarkButton>` | UI-001 | icon button with outline/filled/loading states |
| `bookmarks.store.ts` (Zustand) | UI-001, UI-002 | source of truth; TanStack Query feeds it |

## 4. Dark Mode / Theme

All tokens from `apps/web/src/lib/themes-v2.ts`:
- outline icon: `--color-muted-foreground`
- filled icon: `--color-primary`
- sidebar bg: `--color-surface-2`

No new tokens introduced.

## 5. Design Assets

| Asset | Path | Status |
|---|---|---|
| Screenshots (light) | `screenshots/bookmark-v1/light/*.png` | pending implementation |
| Screenshots (dark)  | `screenshots/bookmark-v1/dark/*.png`  | pending implementation |
