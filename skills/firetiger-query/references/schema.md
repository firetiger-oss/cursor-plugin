# Firetiger telemetry schema

Firetiger infers schema dynamically from incoming OpenTelemetry data, so exact columns vary per service. Run
`DESCRIBE "table_name"` to see the live shape. The columns below are always present.

One row per **span** (there is no separate spans table). `trace_id`/`span_id` surface as BLOB in DuckDB.

## Traces — `"opentelemetry/traces/{service_name}"`

| Column | Type | Description |
|--------|------|-------------|
| `trace_id` | fixed[16] (BLOB) | W3C Trace Context trace ID (16 bytes). Match with `x'...'` literals. |
| `span_id` | fixed[8] (BLOB) | Span ID within the trace (8 bytes) |
| `parent_span_id` | fixed[8] (BLOB) | Parent span ID (8 bytes; empty for root spans) |
| `name` | STRING | Logical operation name (span name) |
| `kind` | INT | 0=UNSPECIFIED, 1=INTERNAL, 2=SERVER, 3=CLIENT, 4=PRODUCER, 5=CONSUMER |
| `start_time` | TIMESTAMPTZ | Span start, UTC (day-partition key — filter on this first) |
| `end_time` | TIMESTAMPTZ | Span end, UTC |
| `status.code` | INT | 0=UNSET, 1=OK, 2=ERROR |
| `status.message` | STRING | Status message |
| `attributes` | STRUCT | Span attributes (dynamically inferred) |
| `scope.name` / `scope.version` | STRING | Instrumentation scope |
| `resource.attributes.service.name` | STRING | Service name |
| `resource.attributes.service.namespace` | STRING | Service namespace |
| `resource.attributes.service.version` | STRING | Service version |
| `events` | LIST<STRUCT> | Span events (nested; event attributes are dropped at ingest) |
| `links` | LIST<STRUCT> | Span links (nested) |
| `sample_rate` | DOUBLE (nullable) | Head sampling rate when present — multiply counts by `1 / sample_rate` to estimate true volume |
| `trace_state`, `flags`, `dropped_*_count` | — | Less-common OTLP fields; `DESCRIBE` to see them |

**Duration is calculated, not stored:** `end_time - start_time AS duration`.

## Logs — `"opentelemetry/logs/{service_name}"`

| Column | Type | Description |
|--------|------|-------------|
| `time` | TIMESTAMPTZ | Log timestamp (partition key — filter on this first) |
| `observed_time` | TIMESTAMPTZ | When the log was observed |
| `severity_number` | INT | TRACE=1-4, DEBUG=5-8, INFO=9-12, WARN=13-16, ERROR=17-20, FATAL=21-24 |
| `severity_text` | STRING | TRACE, DEBUG, INFO, WARN, ERROR, FATAL |
| `body` | inferred (STRING or STRUCT) | Log message body — string, or a struct for structured logs |
| `trace_id` | fixed[16] (BLOB) | Trace ID for correlation (optional) |
| `span_id` | fixed[8] (BLOB) | Span ID for correlation (optional) |
| `attributes` | STRUCT | Log attributes (dynamically inferred) |
| `scope.name` / `scope.version` | STRING | Instrumentation scope |
| `event_name`, `flags`, `dropped_attributes_count` | — | Less-common OTLP fields |
| `resource.attributes.service.name` | STRING | Service name |

Structured-log signals (records carrying `__type__`/`__name__`) also land in per-name sub-tables:
`"opentelemetry/logs/{service}/events/{name}"`, `.../counters/{name}`, `.../gauges/{name}`,
`.../histograms/{name}`, `.../metrics/{name}`.

## Accessing attributes

Attributes and resource attributes are nested structs — access with dot notation:

```sql
-- Span attributes (semantic conventions)
attributes.http.method
attributes.http.route
attributes.http.status_code
attributes.db.system
attributes.db.statement

-- Resource attributes
resource.attributes.service.name
resource.attributes.service.namespace
resource.attributes.telemetry.sdk.language
```

Keys are snake_case-normalized at ingest, and dotted keys become nested struct paths. Attributes are structs,
not maps — use dot access, never `attributes['key']`.

### Type inference depth limit

Firetiger infers attribute types to a **depth of 2 levels**:

- **Levels 1–2** — expanded into queryable struct columns (`attributes.http.route`).
- **Level 3+** — collapsed into a single JSON-typed column at that path. Extract with DuckDB JSON functions:

```sql
SELECT
  json_extract_string(attributes.request.context, '$.tenant_id') AS tenant_id
FROM "opentelemetry/traces/api"
WHERE start_time >= NOW() - INTERVAL '1 hour'
LIMIT 100;
```

## Where to go next

- **Ready-to-run queries** using these columns — recent/slow/error spans, latency percentiles, error logs: [query-examples.md](query-examples.md).
- **Metrics tables** (catalog + per-metric value columns), which follow the same depth-2 attribute rule: [metrics.md](metrics.md).
- **Querying connected sources** (Postgres/MySQL/ClickHouse, Datadog/Prometheus/GCP) and cross-source joins: [querying-connections.md](querying-connections.md).
- **No rows coming back?** Confirm the service is instrumented and exporting with `firetiger-instrument`.
