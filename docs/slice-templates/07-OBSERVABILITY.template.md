---
# =============================================================================
# OBSERVABILITY — metrics, logs, traces, dashboards, alerts, events.
# Rule: every error code emits a log event; alerts exist; cardinality budget.
# =============================================================================
slice_id: <from 00>                           # [REQUIRED]
status: draft                                 # draft | ready
dashboards_added:                             # [REQUIRED]
  - name: <dashboard-name>
    grafana_url: <url>
alerts_added:                                 # [REQUIRED — min 3]
  - name: <alert-name>
    severity: p1 | p2 | p3
events_catalog_path: packages/events/src/catalog/<slice>.ts   # [REQUIRED IF events published]
otel_span_prefix: <svc>.<slice>               # [REQUIRED] for trace filtering
---

# Observability: {{slice_name}}

## 1. Metrics Catalog  [REQUIRED — fenced YAML]

```yaml
metrics:
  - name: <svc>.<slice>.<metric>.<total|duration|ratio>
    type: counter | gauge | histogram
    unit: 1 | seconds | bytes | percent
    description: <what it measures>
    labels:
      - tenant_id         # keep cardinality bounded: NEVER label by user_id or raw entity_id
      - status            # enum, low cardinality
      - error_code        # enum from 04 error taxonomy
    cardinality_budget: <n>    # product of label values × active tenants
    slo_target: <n>            # if this metric has an SLO
    alert_names: [<from §5>]
  # Example real metrics (adapt to your slice):
  - name: product.bookmarks.created.total
    type: counter
    unit: 1
    labels: [tenant_id, entity_kind]
    cardinality_budget: 10k
  - name: product.bookmarks.list.duration.seconds
    type: histogram
    unit: seconds
    labels: [tenant_id, status]
    buckets: [0.05, 0.1, 0.25, 0.5, 1, 2, 5]
```

**Cardinality guardrails:** no `user_id`, `entity_id`, `email`, or free-form string as label. Use span attributes for those.

Prometheus implementation: `packages/metrics/src/<slice>.ts` (extend existing `Counter` / `Histogram` helpers).

## 2. Log Event Catalog  [REQUIRED]

Every error code from 04 emits a log event. Every state transition in 02 emits one too.

```yaml
log_events:
  - event: <svc>.<slice>.<entity>.created
    level: info
    fields:
      - tenant_id
      - entity_id
      - actor_user_id       # hashed if PII classification ≥ medium
      - correlation_id
    sample_rate: 1.0        # 1.0 = every event logged
    pii_redaction: [actor_user_id]
  - event: <svc>.<slice>.<error-code>
    level: warn | error
    fields: [tenant_id, error_code, trace_id, message]
    sample_rate: 1.0
```

Logger: `packages/observability/src/logger.ts` (pino with redactors from 06 §PII).

## 3. Distributed Traces  [REQUIRED]

OpenTelemetry span hierarchy for the journey from 02.

```yaml
spans:
  - name: <svc>.<slice>.api.ENDPOINT-001
    parent: gateway.request
    attributes: [tenant_id, endpoint_id, http.status_code]
    children:
      - name: <svc>.<slice>.db.insert_<table>
        attributes: [db.statement (redacted), db.rows_affected]
      - name: <svc>.<slice>.event.publish.<event.name.v1>
        attributes: [event.name, event.id]
      - name: <svc>.<slice>.ai.<feature>      # if AI involved (cross-ref 08)
        attributes: [ai.model, ai.tokens.input, ai.tokens.output, ai.cache.hit]
```

Naming convention: `<service>.<slice>.<layer>.<operation>`. No PII in attributes.

Exporter: OTel SDK configured in `packages/observability/src/tracing.ts`.

## 4. Dashboards  [REQUIRED]

### Dashboard: {{slice_name}} Overview
Grafana file: `infrastructure/grafana/dashboards/<slice>-overview.json` (commit it).

Panels (min 6):
1. **Request rate** — `rate(<svc>_<slice>_requests_total[5m])` by endpoint
2. **Error rate %** — `sum(rate(…error[5m])) / sum(rate(…total[5m]))`
3. **p50/p95/p99 latency** — histogram_quantile on `…duration_seconds`
4. **Active tenants touching this slice** — `count(count by (tenant_id) (rate(…total[24h])))`
5. **Business KPI** — the outcome metric from 00-INDEX §6 (e.g., bookmarks created per hour)
6. **Fleet health** — pod restarts, queue depth if applicable

Access: linked from 00-INDEX doc.

## 5. Alerts  [REQUIRED — min 3]

```yaml
alerts:
  - name: <slice>_p99_latency_breach
    severity: p2
    expr: histogram_quantile(0.99, rate(<metric>_seconds_bucket[5m])) > <SLO-from-04>
    for: 10m
    annotations:
      summary: "p99 latency on <endpoint> exceeds SLO"
      runbook: docs/runbooks/<slice>-latency.md
  - name: <slice>_error_rate_high
    severity: p1
    expr: sum(rate(<metric>_total{status="error"}[5m])) / sum(rate(<metric>_total[5m])) > 0.05
    for: 5m
    annotations:
      summary: "Error rate > 5% on <slice>"
      runbook: docs/runbooks/<slice>-errors.md
  - name: <slice>_business_anomaly
    severity: p3
    expr: <query proving business-logic anomaly — e.g., approvals stuck > 48h>
    for: 1h
```

Each alert needs a runbook at `docs/runbooks/<slice>-<n>.md` with: symptom → triage steps → fix → escalation.

## 6. Event Catalog (cross-service)  [REQUIRED IF events]

Events published by this slice's endpoints that other services consume.

```yaml
events:
  - name: <svc>.<entity>.<verb>.v1    # e.g., product.bookmark.created.v1
    producer: <svc>-svc
    consumers:
      - <other-svc>-svc
    schema:
      type: object
      required: [event_id, tenant_id, entity_id, occurred_at]
      properties:
        event_id: { type: string, format: uuid }
        tenant_id: { type: string, format: uuid }
        entity_id: { type: string, format: uuid }
        actor_user_id: { type: string, format: uuid }
        occurred_at: { type: string, format: date-time }
        payload: { type: object }
    transport: redis-streams              # match existing packages/events
    delivery: at-least-once
    ordering: per-tenant
    retry_policy:
      max_attempts: 5
      backoff: exponential_jitter
    dlq: <slice>.deadletter
    idempotency_key: event_id
    retention_days: 7
```

Registration: add to `packages/events/src/catalog.ts` so consumers can import typed definitions.

## 7. Logging Redaction Verification  [REQUIRED]

Every field in PII inventory (06 §4) verified redacted in logs.

| Field | Redaction strategy | Test file |
|---|---|---|
| `users.email` | hash(sha256, salt-per-tenant) | `packages/observability/__tests__/redactors.test.ts` |

## 8. Runbook Links  [REQUIRED]

| Scenario | Runbook |
|---|---|
| High latency | `docs/runbooks/<slice>-latency.md` |
| High error rate | `docs/runbooks/<slice>-errors.md` |
| Event consumer lag | `docs/runbooks/<slice>-events.md` |

## 9. Validator Will Fail If …

- Any error code from 04 doesn't appear in §2 log events
- Any metric has label `user_id`, `entity_id`, `email`, or free-form text (cardinality explosion)
- Fewer than 3 alerts
- Any alert missing a runbook link
- Event produced in 04 not registered in §6 catalog
- No dashboard JSON committed
- OTel span names don't follow `<service>.<slice>.<layer>.<operation>` convention
- PII fields from 06 not listed in §7 redaction verification
