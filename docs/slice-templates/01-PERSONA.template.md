---
# =============================================================================
# PERSONA — the one human this slice serves.
# Rule: if a teammate could not recognize this person in real life, FAIL.
# =============================================================================
persona_id: PERSONA-<kebab-name>-v1           # [REQUIRED] e.g. PERSONA-pm-alex-v1
status: draft                                 # draft | ready
archetype: PM|CPO|CSM|EngLead|SalesOps|RevOps|Finance|Support|Admin|EndUser
seniority: IC | Senior IC | Manager | Director | VP | C-level
company_stage: seed | series-a | series-b | series-c | growth | enterprise
company_size_employees: <number>
company_arr_usd: <number>
primary_tool_stack:                           # [REQUIRED] 5+ named tools used daily
  - <tool-name>
  - <tool-name>
---

# Persona: {{full_name}}

> One-sentence positioning (first draft in plain English, rewritten after filling everything else):
> "{{full_name}} is a {{seniority}} {{archetype}} at a {{company_stage}} {{industry}} company who {{top_responsibility}}."

## 1. Named Identity  [REQUIRED]

| Field | Value |
|---|---|
| Full name | |
| Age | |
| Title | |
| Tenure in role (months) | |
| Reports to | (title, not name) |
| Team size (direct reports + peers) | |
| Location / timezone | |
| Education / background | |

**A real person this is modeled on:** (optional but strongly encouraged — former coworker, LinkedIn profile, customer-interview transcript)

## 2. Company Context  [REQUIRED]

| Field | Value |
|---|---|
| Company (fictional) | |
| Industry | |
| Product (what the company sells) | |
| Company stage | (repeats frontmatter) |
| Employee count | (repeats frontmatter) |
| ARR | (repeats frontmatter) |
| Customer count | |
| Go-to-market motion | PLG \| Sales-led \| Hybrid |
| Tech stack (5+ tools used daily) | |
| What your platform replaces in the current stack | |

## 3. Jobs to Be Done  [REQUIRED — max 3, min 1]

Format literally: "When I {{situation}}, I want to {{motivation}}, so I can {{outcome}}". Each outcome MUST have a numeric success metric.

### JTBD-1
- **When:**
- **I want to:**
- **So I can:**
- **Success metric (numeric):** e.g. "reduce time from signal to customer-reachout from 4 days to < 1 hour"
- **Frequency:** daily | weekly | monthly | event-driven

### JTBD-2
(same structure)

### JTBD-3
(same structure)

```json
{
  "jtbd": [
    {
      "id": "JTBD-1",
      "trigger": "<when>",
      "want": "<want>",
      "outcome": "<outcome>",
      "metric": { "name": "<metric>", "current": <n>, "target": <n>, "unit": "<unit>" },
      "frequency": "<freq>"
    }
  ]
}
```

## 4. Top Pain Points  [REQUIRED — min 3]

Each pain point needs: trigger event, current workaround, time/money cost, literal quote. Generic pains ("things are slow", "we need better visibility") FAIL validation.

### Pain-1
- **Trigger event:** (what happens that surfaces this pain)
- **Current workaround:** (tools used, manual steps)
- **Cost per week:** `<hours>h` or `$<n>`
- **Literal verbatim quote:** (in quotes, sounds like a real person — not marketing)
- **Why your platform hasn't fixed it yet:** (honest)

### Pain-2
### Pain-3

## 5. Day in the Life  [REQUIRED]

Hour-by-hour for a typical Tuesday. Name tools at each hour. Mark pain moments with ⚠️.

| Time | Activity | Tools used | Notes |
|---|---|---|---|
| 8:30am | | | |
| 9:00am | | | |
| 10:00am | | | |
| 11:00am | | | |
| 12:00pm | | | |
| 1:00pm | | | |
| 2:00pm | | | |
| 3:00pm | | | |
| 4:00pm | | | |
| 5:00pm | | | |
| 6:00pm+ | | | |

## 6. Atypical Days  [REQUIRED — min 2]

### Atypical-1: `<name>` (e.g. Incident Day, Board Day, Quarter-End)
- Trigger: …
- What changes vs. typical day: …
- Which product surfaces matter more / less: …

### Atypical-2: `<name>`
(same)

## 7. Anti-patterns  [REQUIRED — 3 things this persona will NEVER do]

Helps prevent scope creep. If you catch yourself building one of these, stop.

1. Will never …
2. Will never …
3. Will never …

## 8. Collaborators & Conflicts  [REQUIRED]

| Collaborates with (persona) | On what | Conflict points |
|---|---|---|
| | | |
| | | |

## 9. Current-State Quote  [REQUIRED]

The single sentence that best captures this persona's frustration with the status quo. Put it in your README. Tattoo it on the team's forehead.

> "<literal quote>"
>  — {{full_name}}, {{title}}

## 10. Validator Will Fail If …

- Any section contains the phrases: `drives growth`, `seamless`, `powerful`, `best-in-class`, `streamlined`, `cutting-edge`, `robust`, `intuitive`
- JTBD success metric is non-numeric
- Pain points have no literal quotes
- Day-in-life is blank or has fewer than 8 time slots filled
- Tech stack has fewer than 5 named tools
- Anti-patterns are generic ("won't do sloppy work")
