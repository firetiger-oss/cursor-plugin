# Querying metrics

Metrics use the same per-service-style layout as traces and logs: **one table per metric name**, with the data
points' values inline. There is also a single **metadata catalog** table listing every metric. All table names
contain slashes, so they must be **double-quoted**.

| Table | Contents |
|-------|----------|
| `"opentelemetry/metrics"` | Metadata catalog — one row per unique metric definition. Use it to discover metrics. |
| `"opentelemetry/metrics/{metric_name}"` | Data points for a single metric. The metric name is used verbatim as the path segment, dots included — e.g. `"opentelemetry/metrics/http.server.request.duration"`. |

> There is **no** `opentelemetry_metrics_series` table and **no** `opentelemetry_metrics_gauges` /
> `_counters_cumulative` / `_histograms_cumulative` tables, and **no `series` join**. Values live directly in
> each metric's table.

## Discover available metrics (the catalog)

The `"opentelemetry/metrics"` table is your catalog. Query it first to find metric names, then query the
matching per-metric data table.

| Column | Type | Description |
|--------|------|-------------|
| `name` | STRING | Metric name (e.g. `http.server.request.duration`) — this is the `{metric_name}` in the data table path |
| `type` | STRING | `gauge`, `sum`, `histogram`, or `exponential_histogram` |
| `unit` | STRING | Unit of measurement (`ms`, `bytes`, `1` for dimensionless) |
| `description` | STRING | Human-readable description |
| `aggregation_temporality` | STRING | `delta` (rate metrics) or `cumulative` (counters) |
| `time` | TIMESTAMPTZ | When the metric metadata was first observed (day-partition key) |

```sql
-- List all metrics
SELECT DISTINCT name, type, unit, description
FROM "opentelemetry/metrics"
ORDER BY name
LIMIT 100;
```

```sql
-- Find latency histograms
SELECT name, description, unit
FROM "opentelemetry/metrics"
WHERE type IN ('histogram', 'exponential_histogram')
  AND unit = 'ms'
LIMIT 100;
```

## Per-metric data tables

Every metric's data table shares a common set of columns, plus type-specific value columns. `time` is the
day-partition key — **filter on it first**.

### Common columns

| Column | Type | Description |
|--------|------|-------------|
| `time` | TIMESTAMPTZ | Data point timestamp (partition key) |
| `start_time` | TIMESTAMPTZ | Start of the aggregation window |
| `name` | STRING | Metric name |
| `type` | STRING | `gauge`, `sum`, `histogram`, `exponential_histogram` |
| `unit` | STRING | Unit |
| `description` | STRING | Description |
| `aggregation_temporality` | STRING | `delta` or `cumulative` |
| `attributes` | STRUCT | Data-point attributes (dot-accessed, depth-2 inference — see [schema.md](schema.md)) |
| `resource.attributes.*` | STRUCT | Resource attributes (e.g. `resource.attributes.service.name`) |
| `scope.name` / `scope.version` | STRING | Instrumentation scope |

### Value columns by metric type

| `type` | Value columns |
|--------|---------------|
| `gauge`, `sum` | `value` DOUBLE |
| `histogram` | `count` LONG, `sum` DOUBLE, `min` DOUBLE, `max` DOUBLE, `bucket_counts` LIST<LONG>, `explicit_bounds` LIST<DOUBLE> |
| `exponential_histogram` | `count`, `sum`, `min`, `max`, `scale` INT, `zero_count` LONG, `zero_threshold` DOUBLE, `positive` STRUCT{`offset` INT, `bucket_counts` LIST<LONG>}, `negative` STRUCT{…} |

Always `DESCRIBE "opentelemetry/metrics/{metric_name}"` to confirm the exact columns — attributes are inferred
and vary per metric.

## Examples

### Gauge / sum — read values directly

```sql
SELECT time, value,
       resource.attributes.service.name AS service,
       attributes.http.route AS route
FROM "opentelemetry/metrics/http.server.active_requests"
WHERE time >= NOW() - INTERVAL '1 hour'
ORDER BY time DESC
LIMIT 100;
```

### Sum — aggregate a counter over a window

```sql
SELECT resource.attributes.service.name AS service,
       SUM(value) AS total
FROM "opentelemetry/metrics/http.server.request.count"
WHERE time >= NOW() - INTERVAL '1 hour'
GROUP BY service
ORDER BY total DESC
LIMIT 100;
```

### Histogram — average latency from count and sum

```sql
SELECT attributes.http.route AS route,
       SUM(count) AS requests,
       SUM(sum) / NULLIF(SUM(count), 0) AS avg_ms
FROM "opentelemetry/metrics/http.server.request.duration"
WHERE time >= NOW() - INTERVAL '1 hour'
  AND unit = 'ms'
GROUP BY route
ORDER BY requests DESC
LIMIT 100;
```

## Notes

- Metric data and catalog tables use `time` (not `start_time`) as the day-partition key — filter on it first.
- Attributes follow the same depth-2 inference rule as traces/logs — see [schema.md](schema.md).
- Add a `LIMIT` to bound results (no fixed cap is enforced, but every row is returned as JSON).
- A legacy `"opentelemetry/metrics/metadata"` table may exist in older deployments — prefer `"opentelemetry/metrics"`.

## Where to go next

- **Trace & log columns** and the attribute inference rules: [schema.md](schema.md).
- **Copy-paste trace/log queries** to pair with these metrics: [query-examples.md](query-examples.md).
- **Prometheus / GCP Monitoring metrics** from a connected source (PromQL functions): [querying-connections.md](querying-connections.md).
