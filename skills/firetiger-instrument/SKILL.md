---
name: firetiger-instrument
description: >
  Use when adding OpenTelemetry instrumentation to an application so it sends
  telemetry to Firetiger — setting up tracing, configuring the OTLP exporter, or
  connecting a Node.js, Next.js, Python, Go, or Rust app to Firetiger. Always use
  this skill when instrumenting for Firetiger — it carries the critical gotchas
  (init must load before any other import, Firetiger ingest uses HTTP Basic auth
  not Bearer, credentials come from the get_ingest_credentials MCP tool) that make
  telemetry actually show up.
license: Apache-2.0
user_invocable: true
user_invocable_description: "Instrument your application with OpenTelemetry for Firetiger"
metadata:
  author: firetiger
  version: "1.0.0"
  homepage: https://firetiger.com
  source: https://github.com/firetiger-oss/skills
references:
  - references/nodejs.md
  - references/python.md
  - references/go.md
  - references/rust.md
  - references/custom-spans.md
---

# Firetiger Instrument

Instrument an application with OpenTelemetry so it exports traces (and logs/metrics) to Firetiger. For full
onboarding that also connects integrations and creates a monitoring agent, use `firetiger-setup` — this skill
is instrumentation only.

## Workflow

1. **Get credentials.** Call the Firetiger MCP tool **`get_ingest_credentials`** → OTLP endpoint + username +
   password. If the MCP tools aren't available, tell the user to connect the Firetiger MCP server
   (`https://api.cloud.firetiger.com/mcp/v1`) and sign in, then retry.
2. **Detect the framework** (table below) and open the matching reference.
3. **Check for existing instrumentation** — search for `@opentelemetry` (Node), `opentelemetry` (Python), or
   `go.opentelemetry.io` (Go) imports. If present, just repoint the exporter at Firetiger instead of adding a
   second SDK.
4. **Add the SDK init + env vars**, run, and verify data lands.

| Detected file | Stack | Reference |
|---------------|-------|-----------|
| `next.config.js` / `next.config.ts` | Next.js (uses `@vercel/otel`) | [references/nodejs.md](references/nodejs.md) |
| `package.json` | Node.js / TypeScript | [references/nodejs.md](references/nodejs.md) |
| `requirements.txt` / `pyproject.toml` / `setup.py` | Python | [references/python.md](references/python.md) |
| `go.mod` | Go | [references/go.md](references/go.md) |
| `Cargo.toml` | Rust | [references/rust.md](references/rust.md) |

For business-critical spans, log/trace correlation, and attribute best practices, see
[references/custom-spans.md](references/custom-spans.md).

## Critical: initialization order

**OpenTelemetry MUST initialize before any other import.** If instrumented libraries (Express, HTTP clients,
database drivers) load before the SDK starts, auto-instrumentation never attaches and you get no spans. Load
the instrumentation file first (`node --import ./instrumentation.js …`, `opentelemetry-instrument python …`,
or an init call at the very top of `main`).

## Authentication & environment variables

Firetiger authenticates ingest with **HTTP Basic auth** — the OTLP header is
`Authorization=Basic <base64(username:password)>`, **not** a Bearer token. Configure these wherever the app
runs (`.env.local` for dev, the deploy platform for prod), using values from `get_ingest_credentials`:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT="https://ingest.cloud.firetiger.com"   # use the value from get_ingest_credentials
OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic <base64(username:password)>"
OTEL_SERVICE_NAME="your-service-name"
```

Add credential files to `.gitignore` (`.env.local`, `.env.*.local`). Never commit credentials.

## Verify

1. Generate some traffic.
2. Query with the `firetiger-query` skill:
   `SELECT name, start_time FROM "opentelemetry/traces/your-service-name" WHERE start_time >= NOW() - INTERVAL '15 minutes' LIMIT 10`
   (run `SHOW TABLES;` first to confirm your service's table appeared).

## Common Mistakes

| # | Mistake | Fix |
|---|---------|-----|
| 1 | **Init imported after app code** | Load the instrumentation file first — otherwise auto-instrumentation can't patch already-imported libraries. |
| 2 | **Using `Authorization=Bearer …`** | Firetiger ingest is HTTP Basic auth — `Authorization=Basic <base64(user:pass)>`. |
| 3 | **Hardcoding the endpoint** | Use the endpoint from `get_ingest_credentials`; it can differ per org/region. |
| 4 | **Adding a second SDK on top of existing OTEL** | Detect existing instrumentation and repoint the exporter instead. |
| 5 | **`SimpleSpanProcessor` in production** | Use `BatchSpanProcessor` with `gzip` compression for throughput. |
| 6 | **Committing `.env.local`** | Add credential files to `.gitignore`. |
| 7 | **Service name mismatch** | The `OTEL_SERVICE_NAME` you set is the `{service_name}` in the query table `"opentelemetry/traces/{service_name}"`. |
| 8 | **No graceful shutdown** | Call `sdk.shutdown()` on `SIGTERM` so buffered spans flush before exit. |

## Related

- **Verify the data landed and start querying it:** `firetiger-query` (`SHOW TABLES;`, then the per-service tables this instrumentation creates).
- **Add business-critical spans, log/trace correlation, and attribute hygiene:** [references/custom-spans.md](references/custom-spans.md).
- **Full onboarding beyond instrumentation** (connect integrations, register deploys, create a monitoring agent): `firetiger-setup`.
- **Not writing app code?** If you only have platform logs/traces (Vercel, AWS, GCP, Cloudflare) or a Datadog Agent, wire a drain via `firetiger-setup` instead of an SDK.
- **Once your change ships:** have Firetiger watch the deploy with `firetiger-monitor-deploy`.
