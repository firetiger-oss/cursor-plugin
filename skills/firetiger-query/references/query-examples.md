# Firetiger query examples

Copy-paste patterns for traces and logs. Every query time-filters first (day-partition pruning) and adds a
`LIMIT` to bound the result. Replace service names with ones from `SHOW TABLES;`.

## Recent spans

```sql
SELECT trace_id, name, resource.attributes.service.name AS service,
       end_time - start_time AS duration, start_time
FROM "opentelemetry/traces/checkout-service"
WHERE start_time >= NOW() - INTERVAL '1 hour'
ORDER BY start_time DESC
LIMIT 100;
```

## Slow spans

```sql
SELECT trace_id, name, resource.attributes.service.name AS service,
       end_time - start_time AS duration
FROM "opentelemetry/traces/api-gateway"
WHERE end_time - start_time > INTERVAL '1 second'
  AND start_time >= NOW() - INTERVAL '1 hour'
ORDER BY (end_time - start_time) DESC
LIMIT 50;
```

## Error spans

```sql
SELECT trace_id, name, status.code, status.message, start_time
FROM "opentelemetry/traces/payment-service"
WHERE status.code = 2  -- ERROR
  AND start_time >= NOW() - INTERVAL '1 hour'
ORDER BY start_time DESC
LIMIT 100;
```

## Aggregate by span name

```sql
SELECT name, COUNT(*) AS span_count,
       AVG(EXTRACT(EPOCH FROM (end_time - start_time))) AS avg_duration_sec,
       MAX(EXTRACT(EPOCH FROM (end_time - start_time))) AS max_duration_sec
FROM "opentelemetry/traces/checkout-service"
WHERE start_time >= NOW() - INTERVAL '1 hour'
GROUP BY name
ORDER BY span_count DESC
LIMIT 50;
```

## Latency percentiles by HTTP route

```sql
SELECT attributes.http.route AS route, COUNT(*) AS requests,
       PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (end_time - start_time))) AS p50_sec,
       PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (end_time - start_time))) AS p95_sec,
       PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (end_time - start_time))) AS p99_sec
FROM "opentelemetry/traces/api-gateway"
WHERE start_time >= NOW() - INTERVAL '1 hour'
  AND attributes.http.route IS NOT NULL
GROUP BY attributes.http.route
ORDER BY requests DESC
LIMIT 50;
```

## Error logs

```sql
SELECT time, severity_text, body, trace_id, resource.attributes.service.name AS service
FROM "opentelemetry/logs/checkout-service"
WHERE time >= NOW() - INTERVAL '1 hour'
  AND severity_number >= 17  -- ERROR and above
ORDER BY time DESC
LIMIT 100;
```

## Trace a request across a service

```sql
SELECT name, resource.attributes.service.name AS service, start_time,
       end_time - start_time AS duration, status.code
FROM "opentelemetry/traces/api-gateway"
WHERE trace_id = x'0123456789abcdef0123456789abcdef'
ORDER BY start_time
LIMIT 100;
```

To follow a trace across *multiple* services, run this against each service table for the same `trace_id`
(discover the tables with `SHOW TABLES;`).

## Where to go next

- **Column reference** for any field used above (traces, logs, nested attributes): [schema.md](schema.md).
- **Metric queries** (gauge/sum/histogram value columns): [metrics.md](metrics.md).
- **Join these tables to a connected database or Datadog/Prometheus**: [querying-connections.md](querying-connections.md).
- **Chasing an incident?** `firetiger-investigate` wraps these same queries in a tracked, agent-driven workflow.
