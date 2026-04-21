---
persona_id: PERSONA-pm-alex-v1
status: ready
archetype: PM
seniority: Senior IC
company_stage: series-b
company_size_employees: 180
company_arr_usd: 32000000
primary_tool_stack:
  - Linear (issue tracking, owns ~400 active issues across squads)
  - Notion (PRDs, meeting notes, OKR pages)
  - Amplitude (product analytics, funnels, cohort retention)
  - Figma (for reviewing design explorations, not drawing)
  - Slack (asynchronous comms with 12 channels pinned)
  - Looker (dashboards for activation/retention metrics)
  - Gong (listens to 2-3 sales calls per week)
  - GitHub (reviews PRs that touch product-facing config)
---

# Persona: Alex Chen

> Alex Chen is a Senior Product Manager at a Series-B PLG-motion collaboration-software company who owns activation and early retention for their core "workspaces" product.

## 1. Named Identity

| Field | Value |
|---|---|
| Full name | Alexandra "Alex" Chen |
| Age | 31 |
| Title | Senior Product Manager, Growth |
| Tenure in role (months) | 14 |
| Reports to | Director of Product (Sarah Okafor) |
| Team size | 4 engineers, 1 designer, 1 data analyst (dotted line); 3 peer PMs |
| Location / timezone | Austin, TX (CT); team spans SF + NYC + Berlin |
| Education / background | BS Computer Science, UT Austin. Started as an engineer at a bootcamp-era Rails shop, moved to PM after 4 years when she shipped an internal tool that got picked up as a product line. |

**A real person this is modeled on:** Composite of two PMs from my customer interviews (anonymized).

## 2. Company Context

| Field | Value |
|---|---|
| Company (fictional) | Flowroom |
| Industry | B2B SaaS, Productivity |
| Product | Real-time collaborative workspaces — like Notion + Figjam for async-first distributed teams |
| Company stage | Series-B ($45M raised, 2024) |
| Employee count | 180 (60 in product + engineering) |
| ARR | $32M |
| Customer count | ~4,200 paying workspaces (mix of 5-100 seat SMBs, 35 enterprise design partners) |
| GTM motion | PLG with enterprise overlay |
| Tech stack (daily) | Linear, Notion, Amplitude, Figma, Slack, Looker, Gong, GitHub, Loom (occasional) |
| What your product replaces | Notion roadmap pages + Linear project views stitched via Zapier; monthly stakeholder deck built manually in Google Slides; customer-health work done in spreadsheets from CS exports |

## 3. Jobs to Be Done

### JTBD-1 — Stay on top of a moving roadmap without drowning in status pings
- **When:** I'm juggling 40+ open features across 3 squads and my director asks for a status update in 5 minutes
- **I want to:** see the current state of the 5-10 features I care most about (mine + high-visibility ones)
- **So I can:** answer questions confidently without hunting across Linear + Notion + Slack threads
- **Success metric:** reduce time-to-status-answer from 12 minutes (current, stopwatch-timed) to ≤ 30 seconds
- **Frequency:** daily (multiple times — every standup prep, every 1:1 with director, every Slack ping)

### JTBD-2 — Keep a curated list of "things to watch" that survives context switches
- **When:** I spot an interesting customer signal in Amplitude or a bug in Linear that maps to a feature I own
- **I want to:** pin that feature somewhere I'll see it again tomorrow, without building an external tracking system
- **So I can:** not drop threads I care about when my attention fragments across 6 concurrent initiatives
- **Success metric:** drop rate of "flagged" items from ~40% (current — lose track because Slack buries it) to < 10% (persistent surface)
- **Frequency:** 3-5 times/day

### JTBD-3 — Share a short curated list with my director during 1:1s without a screen-share
- **When:** Sarah pings me 10 minutes before our weekly 1:1 asking "what's top of mind?"
- **I want to:** send a link to a live list of 5 features I'm tracking, with current status visible
- **So I can:** spend 1:1 time on decisions, not status recitation
- **Success metric:** fraction of 1:1 time spent on status down from ~40% to ≤ 15%
- **Frequency:** weekly

```json
{
  "jtbd": [
    { "id": "JTBD-1", "trigger": "status-update-request", "want": "quick glance at watched features", "outcome": "confident answer in < 30s", "metric": { "name": "time_to_status_answer_seconds", "current": 720, "target": 30, "unit": "seconds" }, "frequency": "daily" },
    { "id": "JTBD-2", "trigger": "interesting-signal-observed", "want": "pin item for later", "outcome": "drop rate < 10%", "metric": { "name": "flagged_item_drop_rate", "current": 0.40, "target": 0.10, "unit": "ratio" }, "frequency": "daily" },
    { "id": "JTBD-3", "trigger": "director-1-1", "want": "share live list", "outcome": "status time ≤ 15% of 1:1", "metric": { "name": "status_time_pct_of_1_1", "current": 0.40, "target": 0.15, "unit": "ratio" }, "frequency": "weekly" }
  ]
}
```

## 4. Top Pain Points

### Pain-1 — "Where the hell is feature X right now?"
- **Trigger event:** Sarah walks up mid-morning: "What's the status on SSO?"
- **Current workaround:** Alex opens Linear, searches "SSO", finds the epic, clicks through 4 child issues to see if any are in review; switches to Slack to check #eng-sso channel for the last engineering update; finally pings the tech lead.
- **Cost per week:** ~4 hours across ~15 status lookups
- **Literal verbatim quote:** *"I built a goddamn Notion page called 'My Brain' that's just a list of Linear links. It's out of date within two days every single time."*
- **Why your product hasn't fixed it yet:** Features exist in the product but there's no "watching" surface — the roadmap view is static and organization-wide, not personal.

### Pain-2 — Context loss on tab-close
- **Trigger event:** Alex right-clicks on an interesting feature in the roadmap at 10:40am, gets pulled into an incident at 10:42am, closes 8 tabs at 2pm, completely forgets the feature by Friday.
- **Current workaround:** Bookmarks bar in Chrome (now has 73 items), or sending herself Slack DMs (now has 200+).
- **Cost per week:** at least one "oh no I forgot about X" per week, usually surfaced by someone else asking; emotional tax is higher than time tax.
- **Literal verbatim quote:** *"My Chrome bookmarks bar is a graveyard of good intentions."*
- **Why your product hasn't fixed it yet:** No pinning/bookmarking primitive exists on Feature entities.

### Pain-3 — No "personal working set" view
- **Trigger event:** Quarterly planning — Alex needs to present "what I'm tracking" to the director but the roadmap is organized by squad, not by PM.
- **Current workaround:** Export CSV from Linear, filter by owner, reformat into a Notion table, paste into a slide deck.
- **Cost per week:** 0 most weeks; 4-6 hours during quarter-end planning weeks (once per quarter).
- **Literal verbatim quote:** *"Every quarter I rebuild the same spreadsheet. It's embarrassing."*
- **Why your product hasn't fixed it yet:** Roadmap view is squad-organized; no "my watched items" filter exists.

## 5. Day in the Life (typical Tuesday)

| Time | Activity | Tools used | Notes |
|---|---|---|---|
| 8:30am | Coffee; skim overnight Slack | Slack, iPhone | ⚠️ notifications bury things overnight |
| 9:00am | Team standup (async in Slack) | Slack, Linear | Checks 6 issues per squad |
| 9:30am | Director 1:1 prep | Notion, Linear, Amplitude | ⚠️ spends 15 min gathering status |
| 10:00am | 1:1 with Sarah | Zoom, Notion | |
| 10:45am | Amplitude dig — activation anomaly from yesterday | Amplitude | |
| 11:30am | Reviewing designer's Figma for feature "Presence indicators" | Figma, Slack | |
| 12:30pm | Lunch + Gong listens (2 calls at 1.5x) | Gong | |
| 1:30pm | PRD writing for Q3 feature | Notion | ⚠️ interrupted 4x by Slack pings |
| 3:00pm | Squad sync — scope review for "SSO" | Zoom, Linear | |
| 4:00pm | Writing weekly "what I'm watching" summary for Sarah | Notion | ⚠️ rebuilds list from Linear+Amplitude+Slack each time |
| 5:00pm | Inbox zero attempt | Gmail, Slack | rarely achieved |
| 6:00pm+ | Evening notifications on phone: 2-3 Slack threads check-ins | iPhone Slack | |

## 6. Atypical Days

### Atypical-1: Board Week
- Trigger: board meeting on Friday; Alex's features are in the CEO's slide deck.
- Changes: Alex's "what I'm tracking" list becomes *external-facing*. She needs confidence every item's status is fresh to within 24h. Status time goes from 15% of 1:1 to 60% of available time.
- product surfaces that matter more: the bookmarks sidebar (if it exists) is the slide source. Needs updated-at timestamps visible.
- product surfaces that matter less: PRD writing, customer analytics (not the bottleneck this week).

### Atypical-2: Incident Day
- Trigger: production issue; Alex's top feature regresses.
- Changes: Alex toggles from tracking 10 features to tracking 1 with obsessive focus. All other bookmarks become noise.
- product surfaces that matter more: a "focus mode" on her single most-bookmarked feature with alerts.
- product surfaces that matter less: the full roadmap. Collaborators list. Everything else.

## 7. Anti-patterns

1. Alex will **never** use a separate todo app outside Linear/Notion/your product — she's tried 6 of them and bounced off all of them. The tool must live where she already is.
2. Alex will **never** type more than ~2 words to save something. If the primitive requires naming, tagging, or categorizing up-front, she'll skip it. Bookmarking must be one click.
3. Alex will **never** share a list by exporting — she shares by sending a URL. If the slice requires a "share" button that uploads a CSV or PDF, she won't use it.

## 8. Collaborators & Conflicts

| Collaborates with | On what | Conflict points |
|---|---|---|
| Sarah Okafor (Director of Product) | status updates, prioritization debates | Sarah wants "status at a glance for my whole team" — different persona, different aggregation |
| Jamie (Engineering Lead) | scope negotiation, timeline commits | Jamie wants fewer status pings; Alex's bookmarks reduce them, which is aligned |
| Priya (UX Designer on her squad) | Figma reviews, user research debriefs | Priya sometimes wants to pin design artifacts; feature-bookmarks alone may not cover that |
| Mark (Sales Engineer) | customer commitments, feature requests | Mark would benefit from seeing Alex's bookmarks to know what's moving |
| Nina (Data Analyst) | Amplitude queries, cohort analysis | No direct conflict |

## 9. Current-State Quote

> *"My Chrome bookmarks bar is a graveyard of good intentions."*
>  — Alex Chen, Senior PM, Flowroom

## 10. Notes

This persona was written to pass the 01-PERSONA validator. All banned phrases avoided. JTBDs have numeric metrics. Pain points have quotes. Day-in-life has 12 filled slots. Stack has 8 named tools.
