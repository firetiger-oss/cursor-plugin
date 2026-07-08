# Telemetry ingest sources

Every way to get telemetry into Firetiger. All endpoints are paths under the ingest host
`https://ingest.cloud.firetiger.com` and use **HTTP Basic auth** (`Authorization: Basic base64(username:password)`,
the username/password from `get_ingest_credentials`), except the self-authenticating webhooks noted below.
Everything lands in Apache Iceberg tables you can then query with `firetiger-query`.

## OpenTelemetry (OTLP) — the default

Signals: logs, metrics, traces. Endpoints: `https://ingest.cloud.firetiger.com/v1/traces`, `/v1/logs`,
`/v1/metrics` (HTTP), plus OTLP gRPC on `:443`. Point any OTEL SDK or Collector at the ingest endpoint with the
Basic auth header. For SDK setup per language, use the **`firetiger-instrument`** skill.

## Platform log/trace drains

| Platform | Signals | How | Recipe |
|----------|---------|-----|--------|
| Vercel | logs, traces | Log Drain + trace drain via Vercel API | [vercel.md](vercel.md) |
| AWS | logs, access logs, ECS events | CloudFormation → CloudWatch subscription / Kinesis Firehose / EventBridge / S3 (ALB, CloudFront, MSK, Kafka) | [aws.md](aws.md) |
| GCP | logs | Cloud Logging sink → Pub/Sub → forwarder, or HTTP push to `/gcp/cloud-logging` | [gcp.md](gcp.md) |
| Cloudflare | request logs, traces | Workers observability destinations + Logpush (direct, or via S3/GCS notifications) | [cloudflare.md](cloudflare.md) |

## Agent / collector forwarders

| Source | Signals | Endpoint(s) | Notes |
|--------|---------|-------------|-------|
| **Datadog Agent** | logs, metrics, traces | `/datadog/logs`, `/datadog/api/v1/series`, `/datadog/api/v2/series`, `/datadog/api/v0.2/traces`, `/datadog/api/v0.7/traces` | Repoint `DD_URL` at Firetiger. Uses `DD-API-KEY` header auth as well as Basic. A drop-in migration path — see [datadog.md](datadog.md). |
| **Prometheus** | metrics | `/api/v1/write` | Add Firetiger as a `remote_write` target. |
| **Vector** | logs, metrics, traces | native Vector sink, or the HTTP sink → `/datapoints/` | Basic auth. |
| **Generic HTTP sink** | arbitrary events | `/datapoints/{table}` | Table chosen from the URL suffix — for Fastly, custom shippers, anything that can POST JSON. |

## Event & product webhooks

| Source | Signals | Endpoint | Auth |
|--------|---------|----------|------|
| **GitHub** | PR/deploy/issue events, `@firetiger` mentions | `/github/events` (GitHub App) | GitHub App (Clerk M2M) |
| **SendGrid** | email delivery/engagement | `/sendgrid/`, `/sendgrid/delivery`, `/sendgrid/engagement` | Self-auth via HMAC signature |
| **Convex** | function/console/audit logs | `/convex/` | Self-auth via HMAC |
| **incident.io** | incident events | `/incident-io/events` | webhook |

## Choosing

- Already running the **Datadog Agent**? Repoint it — you keep all three signals with near-zero code change.
- Already exporting **Prometheus**? Add a `remote_write` target.
- On **Vercel/AWS/GCP/Cloudflare**? Use the platform drain — no app changes.
- Greenfield or want traces from your own code? Instrument with the **OTLP SDK** (`firetiger-instrument`).
- Odd source that can POST JSON? Use **`/datapoints/`**.

After wiring a source, verify data is arriving with `firetiger-query` (`SHOW TABLES;`). Data can lag a few
minutes right after setup — absence immediately after isn't a failure.

## Where to go next

- **Don't want to ship telemetry at all?** Many sources (Datadog, Prometheus, GCP metrics, databases) can be
  **pull-based** connections Firetiger queries *into* instead — see [connections.md](connections.md).
- **Instrumenting your own code** for OTLP: the `firetiger-instrument` skill (per-language SDK setup).
- **Querying what landed**, including joins across connected sources: the `firetiger-query` skill.
