---
name: firetiger
description: >
  Use when the user mentions Firetiger or wants to work with their observability
  data — setting up Firetiger, instrumenting an app with OpenTelemetry, querying
  traces/logs/metrics with SQL, investigating an incident, monitoring a PR or
  deployment, or creating a monitoring agent. Always use this skill when the user
  says "Firetiger", even for simple asks — it routes to the specialized skill that
  carries the critical gotchas (Basic-auth ingest, DuckDB SQL over per-service
  tables, the @firetiger comment flow) that prevent common mistakes.
license: Apache-2.0
user_invocable: true
user_invocable_description: "Firetiger observability toolkit — setup, instrumentation, queries, investigations, deploy monitoring, and agents"
metadata:
  author: firetiger
  version: "1.0.0"
  homepage: https://firetiger.com
  source: https://github.com/firetiger-oss/skills
---

# Firetiger

[Firetiger](https://firetiger.com) is an AI-powered observability platform. Telemetry (traces, logs, metrics)
lands in Apache Iceberg tables you query with SQL, and autonomous agents monitor your services, investigate
issues, and watch deployments.

This skill is a **router**. Identify what the user wants and invoke the matching specialized skill — each one
is self-contained and carries the gotchas for its task.

## How you talk to Firetiger

Two mechanisms, referenced throughout the skills:

- **The Firetiger MCP server** — `https://api.cloud.firetiger.com/mcp/v1` (OAuth 2.0 bearer; the first tool
  call opens a browser to sign in). Tools: `get_ingest_credentials`, `get_deploy_credentials`, `query`,
  `monitor_pr`, `create_agent_with_goal`, `send_agent_message`, `read_agent_messages`, `resolve_url`,
  `get_subscription_status`/`get_checkout_url` (billing), `onboard_github`/`onboard_slack`/`onboard_linear`
  (OAuth connect), and generic CRUD (`schema`/`list`/`get`/`create`/`update`/`delete`). The collections a
  client agent works with: `agents` + `sessions` (create/run agents), `triggers` + `scheduled-agent-runs`
  (automate them), `investigations` (AI diagnosis sessions), `issues` (Known Issues, IDs are `FT-{n}` call
  signs), `monitoring-plans` (deploy monitoring, read/delete), and `connections` (integrations). Always
  `schema` a collection before `create`/`update`. Some tools and collections are gated by deployment config
  and API-key policy.
- **The `@firetiger` GitHub flow** — comment `@firetiger` on a pull request to have Firetiger monitor the
  deployment that PR produces.

**Key gotcha:** if the MCP tools aren't available, the user hasn't connected the Firetiger MCP server yet. The
target skill handles this — it tells the user to connect `https://api.cloud.firetiger.com/mcp/v1` and sign in,
then retries.

## Routing

| The user wants to… | Skill | Trigger phrases |
|---|---|---|
| Onboard a project end-to-end (detect stack → instrument → connect → agent) | **`firetiger-setup`** | "set up Firetiger", "onboard this project", "connect my app" |
| Add OpenTelemetry instrumentation (Node/Next.js/Python/Go/Rust) | **`firetiger-instrument`** | "instrument my app", "add OpenTelemetry", "send traces" |
| Query traces, logs, or metrics with SQL | **`firetiger-query`** | "find traces", "search logs", "show me errors", "analyze latency" |
| Investigate an incident and track findings | **`firetiger-investigate`** | "investigate", "diagnose", "what's wrong with", "troubleshoot" |
| Monitor a PR/deployment | **`firetiger-monitor-deploy`** | "monitor this PR", "watch this deploy", "@firetiger" |
| Create a monitoring agent, or configure agents/triggers | **`firetiger-create-agent`** | "create an agent", "monitor X automatically", "schedule an agent" |

## Execution

1. Classify the request into one row above.
2. Invoke the corresponding skill with the Skill tool (e.g. `firetiger-query`).
3. If a request spans categories ("set up Firetiger and monitor my next PR"), handle them sequentially —
   usually `firetiger-setup` first, then the follow-up.

## Resources

- [Firetiger Documentation](https://docs.firetiger.com)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
