# Connections: pull-based sources, integrations & the connect model

A **Connection** is a configured integration Firetiger reaches an external system through. There are three
shapes, and most customers rely heavily on the **pull-based** ones — Firetiger queries *into* their existing
Datadog, Prometheus, databases, and cloud metrics rather than (or in addition to) ingesting from them.

| Shape | What Firetiger does | Examples |
|-------|---------------------|----------|
| **Pull-based data sources** | Queries *into* the system on demand (no data copied) | Postgres, MySQL, ClickHouse, Datadog, Prometheus (PromQL), GCP Monitoring, AWS, Trino, Elasticsearch, web search |
| **Action / tool integrations** | Reads and takes actions | GitHub, Slack, Linear, PagerDuty, incident.io, Pylon, email |
| **Identity / compliance** | Reads identity/compliance data | Clerk, WorkOS, Vanta |

(Push/ingest sources — where the customer *sends* telemetry to Firetiger — are covered in
[ingest-sources.md](ingest-sources.md). AWS, GCP, and Convex appear on both sides.)

## How connections are created

**A client agent can create most connection types directly** with `create` on the `connections` collection —
run `schema` (collection `connections`) first to get the proto shape, then `create` with the `connection_type`,
`display_name`, `description`, and the matching details (host/DSN, API key, region, …). Secrets are input-only
and stored in a secrets backend — they're never echoed back on `get`/`list`.

The exceptions are the three **OAuth** integrations — **GitHub, Slack, Linear** — whose credentials can only
come from an OAuth handshake. Use the `onboard_github` / `onboard_slack` / `onboard_linear` MCP tools; each
returns a browser URL, the user completes OAuth, and Firetiger creates the connection.

**Key gotcha:** once a connection exists, its credentials are **injected by a host-keyed egress proxy**. Agents
query or call the target by connection name and never handle the secret — never hand-set an `Authorization`
header for a connected service, and never query with an embedded password.

## Pull-based data sources

Once connected, these are queried through the `query` tool — see the `firetiger-query` skill's
"Querying connected sources" reference for the exact SQL (SQL databases via fully-qualified
`"connections/{name}".schema.table`, Datadog/Prometheus/GCP via vendor functions, cross-source joins).

| Type | Config (via `create` on `connections`) | Auth |
|------|------------------------------------------|------|
| `POSTGRES` | `host`, `port` (5432), `database`, `username`, `password`, `ssl_mode`, `read_only` | password |
| `MYSQL` | `host`, `port` (3306), `database`, `username`, `password`, `ssl_mode`, `read_only` | password |
| `CLICKHOUSE` | `host`, `port` (9440/8443), `database`, `username`, `password`, `secure` | password |
| `TRINO` | `host`, `port`, `catalog`, `schema`, `username`, `password`, `secure` | password (optional) |
| `ELASTICSEARCH` | `url`, auth: `basic` / `api_key` / `none`, `tls_skip_verify` | basic or Elastic API key |
| `DATADOG` | `site`, `api_key`, `application_key` | API key + app key |
| `PROMQL` | `base_url`, `timeout`, auth: `basic` / `bearer` / `sigv4` (for AMP) | basic / bearer / SigV4 |
| `GCP` | `service_account_key` (JSON), `project_id`, `region` | service account |
| `AWS` | auth: `assume_role` (`role_arn`, `external_id`) **or** `static_credentials`, `region` | STS AssumeRole (preferred) or IAM keys |
| `ICEBERG` | `catalog_uri`, auth: `basic` / `bearer` / `context` | Firetiger's own lake is the default `iceberg-gateway` |
| `WEB_SEARCH` | (none per-connection) | org-level (Brave) |

> Note: `DATADOG` here is the *query* side (agents read your Datadog). Forwarding the Datadog **Agent** into
> Firetiger is the separate push path in [datadog.md](datadog.md). Similarly `AWS`/`GCP` are both query
> connections and log-forwarding ingest sources.

## Action / tool integrations

| Type | Config | Create path |
|------|--------|-------------|
| `GITHUB` | owner, repositories, permissions, `auto_monitor_pull_requests` | **`onboard_github`** (OAuth) |
| `SLACK` | workspace, bot token, allowed channels | **`onboard_slack`** (OAuth) |
| `LINEAR` | org, access/refresh token | **`onboard_linear`** (OAuth) |
| `PAGERDUTY` | `api_token` | `create` |
| `INCIDENT_IO` | `api_key`, `signing_secret` | `create` |
| `PYLON` | `api_token` | `create` |
| `EMAIL_WEBHOOK` | HTTP base URL + auth + signing secret | `create` |
| `GOOGLE_POSTMASTER` | service account (domain-wide delegation) | `create` |
| `CLERK` / `WORKOS` / `VANTA` | bearer token / OAuth client-credentials, `read_only` | `create` |

What they unlock: **GitHub** — deploy monitoring, codebase search, PR/issue actions, and (with telemetry)
discovery; **Slack** — alerts and PR-author DMs; **PagerDuty / incident.io / Linear** — opening and managing
incidents/tickets.

## Generic connectors — reaching any API (including Axiom)

For a vendor with **no dedicated connector**, use a generic protocol connection. Firetiger's agents then call
it with proxy-injected auth:

| Type | Config | Use for |
|------|--------|---------|
| `OPENAPI` | `spec_url`, `server_url`, auth: `bearer` / `basic` / `oauth_client_credentials`, `read_only` | Any API that publishes an OpenAPI spec (introspectable, read-only-gateable) |
| `HTTP` | `base_url`, `allowed_routes[]`, `headers`, auth: `static_headers` / `bearer` / `basic` / `oauth_client_credentials` | Any raw REST API |
| `GRPC` | `address` (host:port), `protocol`, auth: `basic` / `bearer` (injected on :443) | gRPC services with server reflection |
| `GRAPHQL` | `url`, auth: `bearer` / `basic` / `static_headers` | GraphQL endpoints |

**Axiom** has no dedicated connector. Connect it as an **`OPENAPI`** connection to `https://api.axiom.co` with a
bearer token (Axiom publishes an OpenAPI spec) — or as `HTTP` with a bearer header. (Axiom's
Elasticsearch-compatible endpoint could also back an `ELASTICSEARCH` connection.)

## Private networks (NetworkTransports)

A connection whose target isn't publicly reachable references a **NetworkTransport** (`network_transport`
field) — currently **Tailscale** (OAuth client credentials). Firetiger stands up an on-demand tunnel and dials
the private host through it. The transport's route `domain` must cover the target host **exactly** — an exact
host or a `*.prefix` wildcard, no implicit subdomains; a host no route covers is dialed publicly and silently
times out. Required for any database or service on a private network (e.g. RDS behind a VPC).

## Connectivity & health

A connection is **healthy** when Firetiger can resolve its credentials. Firetiger validates with a read-only
probe per type — `SELECT 1` (SQL databases), instant `up` (PromQL), list metrics (Datadog), STS
`GetCallerIdentity` (AWS), list repos (GitHub), etc. A 2xx/success means the credentials authenticate; 401/403
means they don't; a timeout means a reachability/transport problem, not auth. For **ingest** sources the real
check is instead "is data flowing" — query the landing tables with `firetiger-query`.

See <https://docs.firetiger.com> for per-integration setup detail.
