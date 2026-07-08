---
name: firetiger-monitor-deploy
description: >
  Use when monitoring a PR or deployment with Firetiger and reviewing what it found
  — watching a deploy, setting up the @firetiger comment flow, registering
  deployments from CI/CD, or reading a monitoring agent's results. Always use this
  skill for deploy monitoring — it carries the critical gotchas (monitor_pr needs
  the full PR URL, the @firetiger flow needs a connected GitHub App, checks run at
  10m/1h/24h/72h anchored to the real deploy, register deploys with Basic auth so
  checkpoints fire on the right release).
license: Apache-2.0
user_invocable: true
user_invocable_description: "Monitor a PR or deployment and review what Firetiger found"
metadata:
  author: firetiger
  version: "1.1.0"
  homepage: https://firetiger.com
  source: https://github.com/firetiger-oss/skills
---

# Firetiger Monitor Deploy

When a change ships, Firetiger builds a monitoring plan from the PR diff, waits for the deploy, then compares
telemetry against a pre-deploy baseline at fixed checkpoints — reporting regressions on the PR and as issues.

## Quick Start

**From your coding session** — you have the PR URL:
```
monitor_pr with:
  pr_url: "https://github.com/org/repo/pull/123"
  initial_message: "Watch checkout error rate and DB latency; this changes the payment flow"   # optional
```

**From GitHub** — comment on the PR (requires the Firetiger GitHub App connected):
```bash
gh pr comment <PR_NUMBER> --body "@firetiger watch for errors and latency spikes in checkout"
```

After either, Firetiger reacts 👀 to acknowledge, posts a monitoring plan as a PR comment, and — after merge +
deploy — runs checks at **10 minutes, 1 hour, 24 hours, and 72 hours** past the deploy.

**Key gotcha:** the checkpoints anchor to the *actual deploy time*, not the comment. If Firetiger can't tell
when you deployed, register deployments from CI/CD (below) so the schedule fires correctly.

## A. The `monitor_pr` MCP tool

Use `monitor_pr` when you have a PR URL in your session. It analyzes the PR, creates the monitoring plan, and
watches for problems after deploy. It **bypasses** the auto-monitor PR filter (so it always monitors, even for
PRs the auto-filter would skip). Seed `initial_message` from the diff (`gh pr diff`) and the user's concerns —
it steers the plan.

## B. The `@firetiger` GitHub comment flow

Commenting `@firetiger` on a PR enables monitoring with no coding session. The flow is asynchronous:

1. Firetiger's GitHub App sees the mention, reacts **👀** on your comment, and queues the work.
2. A per-PR deploy-monitor agent drafts a monitoring plan and posts it as a PR comment.
3. **Steer it by commenting again** — further `@firetiger` comments are appended to the live planner session.
4. After merge + deploy, checks run at 10m / 1h / 24h / 72h; findings post on the PR and as issues.

Prerequisites & setup:
- **GitHub connection** with an installed app (`installation_id`) — this is the linchpin for reactions,
  comments, and merge-ancestry matching. Connect it via `firetiger-setup` (the `onboard_github` tool or the
  `connections` collection) if `list with resource: "connections", filter: connection_type = "GITHUB"` is empty.
- Optional: a **Slack connection** + per-user preferences to DM the PR author when issues are found.
- **PR-opened auto-monitoring** is a separate path: once enabled, Firetiger can auto-monitor qualifying PRs on
  open (author allowlist + a relevance judge) without any comment.

### Example comments
```
@firetiger
```
```
@firetiger monitor for error-rate increases in /api/orders and any 5xx from the new endpoint
```

## Registering deployments from CI/CD (deployments API)

So checkpoints anchor to the real release, POST a deployment event from your pipeline. Get credentials with the
**`get_deploy_credentials`** MCP tool (endpoint + username/password, auto-provisioned):

```bash
curl -X POST "$FIRETIGER_API_URL/deployments" \
  -H "Authorization: Basic $(printf '%s:%s' "$FT_DEPLOY_USER" "$FT_DEPLOY_PASS" | base64)" \
  -H "Content-Type: application/json" \
  -d '{"repository":"owner/repo","environment":"production","sha":"'"$GIT_SHA"'","deploy_time":"'"$(date -u +%FT%TZ)"'"}'
# → {"name":"deployments/<id>"}
```

Fields: `repository` (`owner/repo`), `environment`, `sha` (hex, 6–64), `deploy_time` (RFC 3339, optional —
defaults to now). Inject the credentials as pipeline secrets; never commit them. GitHub Deployment webhooks
also drive this automatically when the GitHub App is connected, so manual registration is only needed when
Firetiger isn't already seeing your deploys.

## Reading & steering the monitoring agent

Monitoring runs as an agent with sessions and a plan.

```
list with resource: "monitoring-plans"                    # find the plan (read/delete only via MCP)
get with name: "monitoring-plans/<id>"                     # plan_content, deployments[], outcome, timeline
list with resource: "agents"                               # the per-PR deploy-monitor agent (tag firetiger:deploy-monitor)
read_agent_messages with session: "agents/<id>/sessions/<id>"
send_agent_message with session: "agents/<id>/sessions/<id>"
  message: "Compare the 503 rate on /api/orders to the previous version and widen the window to 2h."
```

Per-deployment outcomes are `NO_ISSUE` or `ISSUE_DETECTED`. A **failed** deploy (not just a regression) is
itself filed as a tracked issue tagged `deploy-failure`. `monitoring-plans` is created via `monitor_pr` /
`@firetiger` — the generic `create`/`update` tools don't apply (read and delete only).

## Common Mistakes

| # | Mistake | Fix |
|---|---------|-----|
| 1 | **Partial PR ref to `monitor_pr`** | `pr_url` must be the full URL: `https://github.com/org/repo/pull/123`. |
| 2 | **`@firetiger` with no GitHub App connected** | Connect the GitHub integration first, or the mention is ignored. |
| 3 | **Expecting the wrong checkpoints** | The schedule is 10m / 1h / 24h / 72h after deploy — not 30m/2h. |
| 4 | **Not registering deploys** | Without deploy events, checkpoints can't anchor — POST to `/deployments` from CI (or connect the GitHub App). |
| 5 | **Bearer auth on the deployments API** | It's HTTP Basic auth from `get_deploy_credentials`. |
| 6 | **`create` on `monitoring-plans`** | Plans are created by `monitor_pr`/`@firetiger`; the collection is read/delete only. |
| 7 | **Empty `initial_message`** | Seed it from the diff — a focused hint yields a much better plan. |

## Related

- Deep-dive an issue the monitor filed: `firetiger-investigate`.
- Create a standing (non-PR) monitoring agent: `firetiger-create-agent`.
- Connect GitHub/Slack and register deploys during onboarding: `firetiger-setup`.
