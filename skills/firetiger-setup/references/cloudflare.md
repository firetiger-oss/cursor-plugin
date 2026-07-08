# Cloudflare Workers observability

Send Cloudflare Workers traces and logs to Firetiger via Workers observability destinations. The ingest
endpoint is `https://ingest.cloud.firetiger.com`; auth uses the Step 1 credentials as `$AUTH_HEADER` =
`base64(username:password)`. Requires `which wrangler`.

## Prerequisites

- Account ID (`$ACCOUNT_ID`) and API credentials (`$CF_EMAIL`, `$CF_API_KEY`) — try `wrangler whoami`, or ask
  the user.

## Create the destinations

```bash
# Traces
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/observability/destinations" \
  -H "X-Auth-Email: $CF_EMAIL" -H "X-Auth-Key: $CF_API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"firetiger-traces","enabled":true,"configuration":{"type":"logpush",
       "logpushDataset":"opentelemetry-traces","url":"https://ingest.cloud.firetiger.com/v1/traces",
       "headers":{"Authorization":"Basic '$AUTH_HEADER'"}}}'

# Logs
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/observability/destinations" \
  -H "X-Auth-Email: $CF_EMAIL" -H "X-Auth-Key: $CF_API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"firetiger-logs","enabled":true,"configuration":{"type":"logpush",
       "logpushDataset":"opentelemetry-logs","url":"https://ingest.cloud.firetiger.com/v1/logs",
       "headers":{"Authorization":"Basic '$AUTH_HEADER'"}}}'
```

## Enable observability in `wrangler.toml`

```toml
[observability.traces]
enabled = true
head_sampling_rate = 1.0
destinations = ["firetiger-traces"]

[observability.logs]
enabled = true
head_sampling_rate = 1.0
destinations = ["firetiger-logs"]
```

Then deploy:

```bash
wrangler deploy
```

## Logpush (beyond Workers)

For zone-level HTTP request logs (not just Workers), create a **Logpush** job pointing at Firetiger. Firetiger
accepts Logpush three ways:
- **Direct HTTP** → `https://ingest.cloud.firetiger.com/cloudflare/logs` (or `/cloudflare/logpush/http_requests`).
- **Via S3** → `/cloudflare/logpush/s3` (S3 + EventBridge object notifications).
- **Via GCS** → `/cloudflare/logpush/gcs` (GCS + Pub/Sub notifications).

Use direct HTTP for the simplest path; use the S3/GCS variants when you already push Logpush to object storage.

## Next steps

- **Verify** — after some traffic, run `SHOW TABLES;` with `firetiger-query` (data can lag a few minutes right after setup).
- **App-level traces** — Workers observability captures request logs/traces; for spans from your own code add the OTLP SDK via `firetiger-instrument`.
- **Other sources & integrations** — back to [ingest-sources.md](ingest-sources.md) for more drains, or [connections.md](connections.md) to connect GitHub/Slack and pull-based sources.
