---
name: firetiger-investigate
description: >
  Use when diagnosing an issue or incident with Firetiger — launching or reading a
  Firetiger investigation, troubleshooting with telemetry, or working the issues an
  investigation surfaces. Always use this skill for incident diagnosis — it carries
  the critical gotchas (an investigation is an AI agent session, not a findings form;
  its status is server-set; issues are managed by an expert agent you steer rather
  than mutate) that keep you working with the product instead of against it.
license: Apache-2.0
user_invocable: true
user_invocable_description: "Start or read a Firetiger investigation to diagnose issues"
metadata:
  author: firetiger
  version: "1.1.0"
  homepage: https://firetiger.com
  source: https://github.com/firetiger-oss/skills
---

# Firetiger Investigate

A Firetiger **investigation is an autonomous AI agent session**, not a document you fill in. You give it a
problem statement; the `investigate` agent grounds itself in the affected services and open issues, runs
telemetry queries, and reasons toward a root cause. You read and steer that session — you don't hand-edit
findings fields.

There are two ways to work an incident: **let the investigation agent drive** (this skill), or **query the
data yourself** (the `firetiger-query` skill). Use both — launch an investigation for breadth, drop into direct
SQL when you want to dig a specific thread.

## Quick Start

**Launch an investigation** — `create` on the `investigations` collection with a clear problem statement:
```
create with resource: "investigations"
  body: {
    "display_name": "Checkout 5xx spike",
    "description": "Error rate on checkout-service jumped ~4x starting 14:20 UTC; find the cause and blast radius."
  }
```
The resource ID *is* the agent session ID. Creating it starts the `investigate` agent.

**Read an existing investigation** — you have a Firetiger URL or ID:
```
resolve_url with url: "https://ui.<org>.firetigerapi.com/investigations/<id>"   # → the resource name
read_agent_messages with session: "agents/investigate/sessions/<id>"            # the agent's reasoning + findings
```
(There is also a built-in MCP **prompt** `read-investigation` that takes a `url_or_id` and walks this for you.)

## The investigation resource

Deliberately thin — it wraps a session; the findings live in the transcript, not in fields:

| Field | Notes |
|-------|-------|
| `display_name` | Short title |
| `description` | The problem statement — what to investigate and why |
| `status` | **Server-set**: `EXECUTING` (agent working) or `WAITING` (agent paused, needs input). You do not set it. |
| `created_by`, `create_time` | Output-only |

There is **no** `findings`, `root_cause`, `resolution`, `time_range`, or `services` field — don't try to write
them. To progress or redirect the investigation, message its session.

## Steering an investigation

```
# See where it's at (blocks until the agent is WAITING if you pass wait_for_completion)
read_agent_messages with session: "agents/investigate/sessions/<id>"

# Redirect or answer its question
send_agent_message with session: "agents/investigate/sessions/<id>"
  message: "Focus on the payment dependency — compare error rate to the deploy at 14:15 and check DB latency."
```

`send_agent_message` waits for the agent to finish its turn by default. A `WAITING` status means it wants your
input.

## Diagnosing directly with SQL

When you'd rather drive, use the `query` tool (full mechanics in `firetiger-query`). Telemetry lives in
per-service Iceberg tables; discover them with `SHOW TABLES;`. Duration = `end_time - start_time`; errors =
`status.code = 2`; filter the time column first. Bound results with a `LIMIT`.

**Errors in the window:**
```sql
SELECT trace_id, name, status.code, status.message, start_time
FROM "opentelemetry/traces/checkout-service"
WHERE status.code = 2
  AND start_time BETWEEN TIMESTAMPTZ '{start}' AND TIMESTAMPTZ '{end}'
ORDER BY start_time DESC
LIMIT 100;
```

**Latency by operation:**
```sql
SELECT name, COUNT(*) AS count,
       AVG(EXTRACT(EPOCH FROM (end_time - start_time))) * 1e3 AS avg_ms,
       MAX(EXTRACT(EPOCH FROM (end_time - start_time))) * 1e3 AS max_ms
FROM "opentelemetry/traces/checkout-service"
WHERE start_time BETWEEN TIMESTAMPTZ '{start}' AND TIMESTAMPTZ '{end}'
GROUP BY name
ORDER BY avg_ms DESC
LIMIT 100;
```

**Error logs → then pull the trace by ID** (`x'...'` literal):
```sql
SELECT time, severity_text, body, trace_id
FROM "opentelemetry/logs/checkout-service"
WHERE time BETWEEN TIMESTAMPTZ '{start}' AND TIMESTAMPTZ '{end}'
  AND severity_number >= 13   -- WARN and above
ORDER BY time DESC
LIMIT 100;
```

## Issues — the durable output of a diagnosis

A Firetiger **Issue** (a "Known Issue") is the record an agent files when it finds a real, human-worthy
problem. Issue IDs are call signs: **`FT-{number}`**, resource name `issues/FT-{number}` (server-generated).
Full CRUD is available on the `issues` collection.

**Workflow state** (`workflow_state`): `INVESTIGATING → ACTIONABLE → VERIFYING_FIX → CLOSED`. New issues start
`INVESTIGATING`. Closing requires a `closure.reason`, one of: `RESOLVED`, `ACCEPTED_RISK`, `FALSE_POSITIVE`,
`DUPLICATE`, `NOT_USEFUL`. There is no severity field.

**Key gotcha:** issues are triaged and enriched by an autonomous **Issue Expert agent** (the issue's
`expert_session`). Don't hand-edit issue fields to steer triage — `send_agent_message` to the `expert_session`
and let the expert own `workflow_state`, `details`, and closure. Read the issue with `get`, note its
`source`, `services`, `pull_requests`, and `observations`.

To find or review issues:
```
list with resource: "issues"                 # filter by workflow_state, service, etc.
get with name: "issues/FT-42"
```

## Common Mistakes

| # | Mistake | Fix |
|---|---------|-----|
| 1 | **Treating an investigation as a form** | It's an agent session — set `description` and read/steer via the session; there are no findings/root_cause fields to write. |
| 2 | **Setting `status: resolved`** | `status` is server-set (`EXECUTING`/`WAITING`) — you can't mark an investigation resolved. Close the loop by filing/closing an **issue**. |
| 3 | **Editing issue fields to drive triage** | Message the issue's `expert_session` instead; the expert owns state and enrichment. |
| 4 | **Inventing an issue ID** | `FT-{number}` call signs are server-generated on create — don't pass one. |
| 5 | **Closing an issue without a reason** | `closure.reason` is required (RESOLVED / ACCEPTED_RISK / FALSE_POSITIVE / DUPLICATE / NOT_USEFUL). |
| 6 | **Querying a generic `traces` table** | Data is per-service — `SHOW TABLES;`, then `"opentelemetry/traces/{service}"`. See `firetiger-query`. |
| 7 | **`status_code = 'ERROR'` / `duration_ns`** | Use `status.code = 2` and `end_time - start_time`. |

## Related

- Query mechanics and full schema: `firetiger-query`.
- Create a standing agent to investigate on a schedule or trigger: `firetiger-create-agent`.
- Tie an investigation to a specific deploy: `firetiger-monitor-deploy`.
