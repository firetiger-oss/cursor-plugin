# Datadog with Firetiger

There are two ways to use Datadog with Firetiger, and they're equally valid — pick by what the customer wants,
or do both:

- **Query your existing Datadog** (pull-based connection) — Firetiger's agents read your Datadog logs, spans,
  metrics, monitors, and dashboards on demand. Nothing moves; your data stays in Datadog.
- **Forward the Datadog Agent** (push/ingest) — repoint the agent so its telemetry lands in Firetiger's lake.
  Useful when consolidating onto Firetiger or migrating off Datadog.

## Query your existing Datadog (pull)

Add a `DATADOG` **Connection** — `schema` then `create` on the `connections` collection with `site`,
`api_key`, and `application_key`. Once connected, agents query it through the `query` tool with the Datadog
functions (`datadog_search_logs`, `datadog_query_metrics`, `datadog_cost_analysis`, …) — see the
`firetiger-query` "Querying connected sources" reference, and [connections.md](connections.md) for the
connection config. This needs no changes to your Datadog setup at all.

## Forward the Datadog Agent (push)

If the project already runs the **Datadog Agent**, repoint it at Firetiger's Datadog-compatible ingest to send
logs, metrics, and traces with almost no code change (uses the Step 1 ingest credentials).

Firetiger accepts Datadog-agent traffic at `https://ingest.cloud.firetiger.com` on these paths:

| Signal | Endpoint(s) |
|--------|-------------|
| Logs | `/datadog/logs` |
| Metrics | `/datadog/api/v1/series`, `/datadog/api/v2/series` |
| Traces | `/datadog/api/v0.2/traces`, `/datadog/api/v0.7/traces` |

Authentication accepts the Datadog **`DD-API-KEY`** header (use the Firetiger-issued key) as well as HTTP Basic
auth. Discovery/validation endpoints (`/datadog/info`, `/datadog/api/v1/validate`) are public.

### Repoint the agent

Set the base URL (and API key) in `datadog.yaml` or via environment:

```yaml
# datadog.yaml
dd_url: "https://ingest.cloud.firetiger.com/datadog"        # metrics
logs_config:
  logs_dd_url: "ingest.cloud.firetiger.com:443"
  use_http: true
apm_config:
  apm_dd_url: "https://ingest.cloud.firetiger.com/datadog"  # traces
api_key: "<firetiger-issued-key>"
```

Or with environment variables:

```bash
DD_DD_URL="https://ingest.cloud.firetiger.com/datadog"
DD_APM_DD_URL="https://ingest.cloud.firetiger.com/datadog"
DD_LOGS_CONFIG_LOGS_DD_URL="ingest.cloud.firetiger.com:443"
DD_API_KEY="<firetiger-issued-key>"
```

### Dual-shipping

To validate before cutting over, run both backends in parallel (Datadog + Firetiger) using the agent's
multi-destination config, compare the data in Firetiger with `firetiger-query`, then remove the Datadog
destination once confident.
