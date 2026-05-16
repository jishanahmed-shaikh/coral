# SigNoz

**Version:** 0.1.0
**Backend:** HTTP
**Tables:** 6
**Base URL:** your SigNoz instance URL (set via `SIGNOZ_URL`)

Query services, logs, traces, dashboards, and alerts from SigNoz
(Cloud or self-hosted).

## Authentication

Requires a `SIGNOZ_API_KEY` from a service account with the **SigNoz-Viewer** role.

To create one:

1. Open **Settings > Service Accounts** in your SigNoz instance.
2. Click **New Service Account**, give it a name, and click **Create**.
3. In the **Overview** tab, assign the **SigNoz-Viewer** role and click **Save**.
4. Switch to the **Keys** tab, click **Add Key**, enter a name, and click **Create**.
5. Copy the key immediately — it is shown only once.

See the [SigNoz service accounts docs](https://signoz.io/docs/manage/administrator-guide/iam/service-accounts/).

```bash
SIGNOZ_URL=https://signoz.example.com \
SIGNOZ_API_KEY=<your-key> \
  coral source add --file sources/community/signoz/manifest.yaml
```

Run from the repo root. Or interactively:

```bash
SIGNOZ_URL=https://signoz.example.com \
SIGNOZ_API_KEY=<your-key> \
  coral source add --file sources/community/signoz/manifest.yaml --interactive
```

## Tables

| Table | Description | Required filters | Optional filters |
|---|---|---|---|
| `services` | APM services reporting to SigNoz | — | — |
| `operations` | Span operation names for a service | `service_name` | — |
| `logs` | Log entries via query_range API | `start_time`, `end_time` | — |
| `traces` | Trace spans via query_range API | `start_time`, `end_time` | — |
| `dashboards` | Dashboards configured in SigNoz | — | — |
| `alerts` | Alert rules configured in SigNoz | — | — |

### Time filter note

`start_time` and `end_time` on the `logs` and `traces` tables are **epoch milliseconds**.
Each query returns up to 100 rows ordered by timestamp descending.

## Quick start

```bash
# List all instrumented services
coral sql "
  SELECT serviceName, p99, avgDuration, numCalls, errorRate
  FROM signoz.services
  ORDER BY errorRate DESC
"

# List operations for a service
coral sql "
  SELECT operation
  FROM signoz.operations
  WHERE service_name = 'frontend'
"

# Search recent error logs
coral sql "
  SELECT timestamp, body, severity_text, trace_id
  FROM signoz.logs
  WHERE start_time = 1700000000000
    AND end_time   = 1700003600000
  LIMIT 50
"

# Search recent trace spans
coral sql "
  SELECT timestamp, traceID, spanID, name, serviceName, durationNano, statusCode
  FROM signoz.traces
  WHERE start_time = 1700000000000
    AND end_time   = 1700003600000
  LIMIT 50
"

# List all dashboards
coral sql "
  SELECT uuid, title, description, created_by, updated_at
  FROM signoz.dashboards
"

# List all alert rules
coral sql "
  SELECT id, alert, alertType, state, severity, disabled
  FROM signoz.alerts
  ORDER BY state, alert
"
```

## Discovery order

```text
services
  → serviceName
    → operations (WHERE service_name = '...')
    → traces     (WHERE start_time = ... AND end_time = ...)

logs
  → trace_id → traces.traceID

dashboards
  → uuid

alerts
  → id
```
