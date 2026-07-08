# Querying connected sources (federation)

The `query` tool isn't limited to Firetiger's own telemetry — it runs **Confit SQL**, which can also reach the
data sources you've connected (databases, Datadog, Prometheus, GCP Monitoring) and even **join across them in a
single query**. Confit SQL is DuckDB dialect plus connection-aware `FROM` resolution plus vendor functions;
only `SELECT` / `SHOW` / `DESCRIBE` are allowed.

The connections available to you are whatever is configured on the org — `list with resource: "connections"`
to see them. Each is addressed by its **full resource name**, `connections/{name}`.

## Two addressing patterns

### 1. SQL sources → fully-qualify the table

Firetiger's own telemetry lake, **`connections/iceberg-gateway`**, is the **default** connection: the `query`
tool auto-selects it (it prepends `USE "connections/iceberg-gateway";` when your SQL has no leading `USE`), so
plain telemetry queries just name the table — `FROM "opentelemetry/traces/checkout-service"` is implicitly
`FROM "connections/iceberg-gateway"."opentelemetry/traces/checkout-service"`. To query a *different* connected
SQL source, **fully-qualify the table with the connection name** — `"connections/{name}"` is the leading
identifier (quoted, because it contains a `/`), followed by the schema/database and table:

```sql
-- Postgres / MySQL:  "connections/{name}".<schema>.<table>
SELECT id, email, created_at
FROM "connections/prod-postgres".public.users
WHERE created_at >= NOW() - INTERVAL '1 day'
ORDER BY created_at DESC
LIMIT 100;

-- ClickHouse:  "connections/{name}".<database>.<table>
SELECT event_type, count(*) FROM "connections/prod-clickhouse".default.events
WHERE timestamp >= NOW() - INTERVAL '1 hour'
GROUP BY event_type LIMIT 100;
```

Prefer this fully-qualified form: each query is self-contained (no hidden default state), and it lets you
**join across connections in one query**. The engine federates the source in server-side, so your SQL never
contains its credentials. Connected **Postgres, MySQL, and ClickHouse** are reachable this way; connected
**Trino** and **Elasticsearch** are *not* reachable through the `query` tool — query those from the Firetiger
dashboard or via a generic HTTP connection.

> **`USE` is optional.** `USE "connections/{name}";` at the start of a statement just sets the default
> connection for *unqualified* table names — handy when a whole query targets one source. It's not required,
> and fully-qualifying is clearer for one-offs and mandatory when a single query spans two connections.

### 2. Observability backends → vendor functions

Datadog, Prometheus, and GCP Monitoring are **not** SQL databases — you reach them through table functions,
passing the **connection name as the first string argument** (a literal, not a `USE`) and the vendor's own
query language as a string.

**Datadog** (native Datadog search syntax / MQL):
```sql
SELECT * FROM datadog_search_logs(
  'connections/prod-datadog',
  'status:error service:api',                 -- Datadog log query syntax
  NOW() - INTERVAL '1 hour', NOW(),
  max_rows => 500);
```
Other Datadog functions: `datadog_search_spans`, `datadog_query_metrics`, `datadog_list_metrics`,
`datadog_list_monitors` / `datadog_get_monitor`, `datadog_list_dashboards` / `datadog_get_dashboard`, and
`datadog_cost_analysis` (reads Datadog **billing** metadata — custom-metric series counts, indexed log volume,
tag cardinality — to find what's driving Datadog spend).

**Prometheus / PromQL:**
```sql
SELECT timestamp, value FROM promql_query_range(
  'connections/prod-prom',
  'sum by (status) (rate(http_requests_total{env="prod"}[5m]))',   -- native PromQL
  NOW() - INTERVAL '1 hour', NOW(), '1m')
LIMIT 200;
```
Also `promql_query` (instant), and discovery: `promql_metadata`, `promql_labels`, `promql_label_values`,
`promql_series`. Use `rate()`/`increase()` on counters, aggregate *outside* the rate, and keep `[range]` ≥ 2×
the scrape interval.

**GCP Cloud Monitoring** uses the same `promql_*` functions pointed at a GCP-type connection. GCP metric names
carry a kind prefix — e.g. `compute_googleapis_com:instance_cpu_utilization`,
`kubernetes_io:container_memory_used_bytes` — resolve the exact name (with its `:` and underscores) first, and
filter by `project_id` + `location`.

## Cross-source joins

A single query can join a vendor function's output to Firetiger telemetry, or two connections to each other —
e.g. compare CPU across two Prometheus environments:

```sql
SELECT p.timestamp, p.value AS prod_cpu, s.value AS staging_cpu
FROM promql_query_range('connections/prod-prom', 'rate(process_cpu_seconds_total[5m])',
       NOW() - INTERVAL '1 hour', NOW(), '1m') p
JOIN promql_query_range('connections/staging-prom', 'rate(process_cpu_seconds_total[5m])',
       NOW() - INTERVAL '1 hour', NOW(), '1m') s
  ON p.timestamp = s.timestamp
ORDER BY p.timestamp
LIMIT 100;
```

## Discovering what's queryable

The available functions depend on which connection types are in scope. List them from SQL:

```sql
SELECT * FROM confit_functions();    -- functions available for your active connections
SELECT * FROM confit_signatures();   -- their argument signatures
SELECT * FROM confit_examples();     -- worked examples
```

## Gotchas

| Gotcha | Detail |
|--------|--------|
| **Connection name form differs by pattern** | In `FROM`, `"connections/pg".schema.table` — the connection is a **quoted identifier**. In a vendor function, `datadog_search_logs('connections/pg', …)` — it's a **string literal argument**. Don't swap them. |
| **ClickHouse needs database + table** | `"connections/ch".<db>.<table>` — both parts after the connection are required. |
| **Time arguments** | Pass `TIMESTAMPTZ` literals or `NOW() - INTERVAL 'N unit'` — never bare unix timestamps. |
| **`max_rows` only on search/list functions** | Adding `max_rows` to a point-query function (e.g. `promql_query`) is a signature error. |
| **No `SELECT *` on `promql_query_range`** | Its `labels` MAP inflates output — select `timestamp, value`. |
| **Vendor APIs bill on window × resolution** | Start with a 15-minute window and widen only after a hypothesis; Datadog/GCP charge per query. |
| **Telemetry lags ~5 min** | For fresher-than-5-minute data, query the source connection directly instead of Firetiger's lake. |
| **Trino / Elasticsearch** | Not reachable via the `query` tool — no federation path. Use the dashboard or a generic HTTP connection. |
| **`LIMIT` still applies** | Add a `LIMIT` to every SELECT, federated or not. |

## Where to go next

- **No connection to query yet?** Create one — per-type config (DSN, API key, region) and the connect model are in the `firetiger-setup` skill's "Connections" reference.
- **Querying Firetiger's own telemetry** (traces/logs/metrics tables): [schema.md](schema.md), [query-examples.md](query-examples.md), [metrics.md](metrics.md).
