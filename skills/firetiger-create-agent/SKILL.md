---
name: firetiger-create-agent
description: >
  Use when creating a Firetiger monitoring agent, installing a prebuilt catalog
  agent, or configuring an agent's triggers and schedules — monitoring something
  automatically, scheduling recurring or one-off analysis, or wiring event-driven
  automation. Always use this skill to create or automate an agent —
  it carries the critical gotchas (prefer create_agent_with_goal and answer the
  planner, agents must be AGENT_STATE_ON to fire, triggers are separate resources
  with six config types, scheduled-agent-runs are one-shot not cron).
license: Apache-2.0
user_invocable: true
user_invocable_description: "Create or automate a monitoring agent in Firetiger"
metadata:
  author: firetiger
  version: "1.1.0"
  homepage: https://firetiger.com
  source: https://github.com/firetiger-oss/skills
---

# Firetiger Create Agent

Firetiger agents autonomously analyze telemetry, detect anomalies, investigate issues, and take actions through
connections (Slack, GitHub, Linear, databases, …). Three ways to get one — **prefer a goal or a catalog
template** over hand-building.

## Quick Start

```
create_agent_with_goal with:
  goal: "Monitor Next.js API routes for errors and slow responses; alert on error-rate spikes and p95 latency increases"
  title: "Checkout Health Monitor"   # optional
```

The agent-planner (a system agent, `agents/agent-planner`) configures the prompt, connections, and triggers
from the goal, then returns the planner conversation. **If the planner asked a question, you must answer it**
or the agent won't finish — reply with `send_agent_message` on the plan session.

## Path A (recommended): create from a goal

1. Call `create_agent_with_goal` with a clear `goal` (optional `title`, `agent_id`, `wait_for_plan` default true).
2. **Handle planner questions.** The planner asks sparingly ("act first, refine later"), via a
   `request_user_input`. Answer on the user's behalf from the codebase/context — or ask the user if genuinely
   unknown — by writing to the plan session:
   ```
   send_agent_message with session: "<plan_session from the tool output>"
     message: "Monitor the Postgres connection at DATABASE_URL; alert threshold p99 > 500ms."
   ```
3. **Confirm.** When the planner completes, the agent flips to `AGENT_STATE_ON`. Share its name and focus.

### Example goals
- "Monitor Next.js API routes for errors and slow responses"
- "Track database query latency and alert if p99 exceeds 500ms"
- "Watch for deployment failures and notify on Slack"
- "Track error rates across all services and open issues for spikes"

## Path B: install a prebuilt catalog agent

Firetiger ships tuned templates for common stacks. Browse and install them:
```
list with resource: "catalog-agents"
create with resource: "catalog-agents/{catalog-agent}/..."   # or the InstallCatalogAgent flow
```
Templates include `postgres-performance`, `postgres-schema`, `mysql-performance`, `mysql-schema`,
`clickhouse-performance`, `temporal-performance`, `aws-cost-optimizer`, and `datadog-cost-expert`.

**Key gotcha:** an installed catalog agent starts in **`AGENT_STATE_RECOMMENDED`**, not ON — it won't run
until a human turns it on. Send it a direct message (or set `state: AGENT_STATE_ON`) to activate.

## Path C: configure manually

Use the generic CRUD tools when you need explicit control. **Always run `schema` first** — fields drift.

```
schema with collection: "agents"
```

Agent fields worth knowing: `title`, `description`, `prompt` (guides every session), `state`, `connections[]`
(`{name: "connections/{id}", enabled_tools: [...]}` — what external tools the agent may use),
`mcp_connections[]` (external MCP servers), `tags[]`, `expire_time` (auto-archive after), `cache_ttl`
(`CACHE_TTL_5M` / `CACHE_TTL_1H`), `network_profile`. `skills` and `plan_session` are output-only.

**`state` (`AgentState`) values:** `AGENT_STATE_ON`, `AGENT_STATE_OFF`, `AGENT_STATE_RECOMMENDED`,
`AGENT_STATE_REVIEW_REQUIRED`, `AGENT_STATE_UNSPECIFIED`. Only an **ON** agent fires from triggers or schedules.

```
create with resource: "agents"
  body: { "title": "Checkout Health Monitor",
          "description": "Daily health analysis of checkout",
          "prompt": "You are a daily health check agent...",
          "connections": [{"name": "connections/slack-prod", "enabled_tools": ["post_message"]}],
          "state": "AGENT_STATE_ON" }
```

## Triggers — how agents fire (six kinds)

Triggers are **separate resources** (`triggers/{id}`) referencing an agent. `InvokeTrigger` and automatic
firing require the target agent be `AGENT_STATE_ON`. Common fields: `display_name`, `agent` (required,
`agents/{id}`), `enabled`, `associated_resources`, and one `configuration`:

| Config | Fires when | Key fields |
|--------|-----------|------------|
| `cron` | On a recurring schedule | `schedule` (5-field cron), `timezone` (IANA, default UTC) |
| `manual` | On `InvokeTrigger` or a POST to its webhook | `webhook_token`, `webhook_address` (output-only) |
| `post_deploy` | Once when a SHA (or descendant) deploys | `repository`, `environment`, `sha`, `delay` |
| `row` | When a matching row lands in a table | `table_name`, `predicate` (SQL), `cooldown` (default 15m) |
| `slack_message_posted` | On messages in Slack channels | `slack_connection`, `channels[]`, `include_thread_replies` |
| `slack_agent_mentioned` | When the agent's Slack handle is mentioned | `slack_handle`, `channels[]` |

Cron example:
```
create with resource: "triggers"
  body: { "display_name": "Daily Checkout Health Check",
          "agent": "agents/{agent-id}", "enabled": true,
          "configuration": { "cron": { "schedule": "0 9 * * *", "timezone": "America/Los_Angeles" } } }
```

## Scheduled agent runs — one-shot, not cron

`scheduled-agent-runs` are **single deferred invocations** ("run this prompt on agent X in 1 hour"), not
recurring schedules. Fields: `agent_id`, `prompt`, `time_delta` (delay from now), plus `use_latest_session` /
`incognito`; `scheduled_time` and `has_run` are output-only. For anything recurring, use a **cron trigger**.

## Writing good agent prompts

Specific, scoped, actionable — and say what the agent should NOT do.

```
You are a daily health check agent for the checkout service.

Your job:
1. Query the last 24 hours of traces for checkout.
2. Calculate error rates and p99 latency; compare to the previous 24h.
3. If error rate rose >10% or p99 >20%, open an issue.
4. Summarize findings briefly.

Focus only on checkout. Do not investigate other services.
```

## Common Mistakes

| # | Mistake | Fix |
|---|---------|-----|
| 1 | **Ignoring the planner's question** | `create_agent_with_goal` isn't done until you answer via `send_agent_message` on the plan session. |
| 2 | **Expecting a catalog agent to run immediately** | It installs as `AGENT_STATE_RECOMMENDED` — turn it ON. |
| 3 | **Trigger created but agent is OFF** | Firing requires `AGENT_STATE_ON`. |
| 4 | **Using `scheduled-agent-runs` for recurring work** | Those are one-shot — use a `cron` trigger for recurrence. |
| 5 | **Manual `create` without `schema`** | Field names drift — call `schema` for `agents`/`triggers` first. |
| 6 | **Embedding the trigger in the agent** | Triggers are separate resources referencing `agents/{id}`. |
| 7 | **Unbounded prompt** | State what the agent should NOT do; scope it to specific services. |

## Related

- Run and steer an existing agent's session: `firetiger-monitor-deploy`.
- Manual investigations and the issues agents open: `firetiger-investigate`.
- Connect the integrations an agent acts through: `firetiger-setup`.
