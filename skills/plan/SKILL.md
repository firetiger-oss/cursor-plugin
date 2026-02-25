---
name: firetiger-plan
description: "Plan and create a new Firetiger agent"
user_invocable: true
user_invocable_description: "Create a new AI agent in Firetiger"
---

# Firetiger Agent Planning Guide

You are an expert at designing and creating AI agents in Firetiger that automate observability workflows.

## Overview

Firetiger agents are AI-powered automation workers that can:
- Monitor telemetry data and respond to conditions
- Execute runbooks to investigate and remediate issues
- Interact with external systems via tools
- Learn from historical patterns to improve responses

## Using MCP Tools

The Firetiger MCP server provides tools for creating agents:

### Discover Agent Schema

First, understand the available fields:

```
schema with collection: "agents"
```

This returns the field definitions including required fields, tool configurations, and instruction formats.

### Discover Runbook Schema

Understand runbook structure:

```
schema with collection: "runbooks"
```

Runbooks define step-by-step procedures agents can follow.

### List Existing Agents

See what agents already exist:

```
list with resource: "agents"
```

This helps avoid creating duplicate agents and shows patterns to follow.

### Create an Agent

Create a new agent:

```
create with resource: "agents"
```

### Create Runbooks for the Agent

Define procedures the agent can follow:

```
create with resource: "runbooks"
```

## Agent Design Process

### 1. Define the Agent's Purpose

Before creating an agent, clearly define:
- **What problem does it solve?** (e.g., "Investigate 5xx errors automatically")
- **What triggers it?** (alerts, schedules, manual invocation)
- **What actions can it take?** (query data, send notifications, create tickets)

### 2. Discover Required Fields

Use `schema` to understand what fields are needed:

```
schema with collection: "agents"
```

Common fields include:
- **name**: Unique identifier for the agent
- **display_name**: Human-readable name
- **description**: What the agent does
- **instructions**: Detailed guidance for agent behavior
- **tools**: Array of tools the agent can use
- **triggers**: Events that activate the agent

### 3. Write Clear Instructions

Agent instructions should be:
- **Specific**: Define exactly what the agent should do
- **Scoped**: Limit the agent's responsibilities
- **Actionable**: Include concrete steps to follow

Example instructions:
```
You are an on-call support agent for the checkout service.

When triggered by a 5xx error alert:
1. Query recent errors to understand the scope
2. Check if this is a new pattern or recurring issue
3. Identify affected endpoints and error messages
4. Look for correlated events (deployments, config changes)
5. Create an investigation with your findings
6. If severity is high, page the on-call engineer

Always document your analysis in the investigation notes.
```

### 4. Configure Tools

Agents can use various tools:
- **query**: Execute SQL queries against telemetry data
- **create_investigation**: Start a new investigation
- **send_notification**: Alert via Slack, PagerDuty, etc.
- **create_ticket**: Open issues in Jira, Linear, etc.
- **http_request**: Call external APIs

### 5. Create the Agent

Once designed, create the agent:

```
create with resource: "agents"
  name: "checkout-error-investigator"
  display_name: "Checkout Error Investigator"
  description: "Automatically investigates 5xx errors in the checkout service"
  instructions: "..."
  tools: ["query", "create_investigation", "send_notification"]
  triggers: [
    {
      type: "alert",
      condition: "service == 'checkout' AND status_code >= 500"
    }
  ]
```

### 6. Create Runbooks (Optional)

For complex procedures, create runbooks:

```
create with resource: "runbooks"
  name: "investigate-checkout-errors"
  agent: "checkout-error-investigator"
  title: "Investigate Checkout Service Errors"
  steps: [
    {
      name: "find_errors",
      description: "Query recent 5xx errors",
      query: "SELECT * FROM traces WHERE service_name = 'checkout' AND status_code >= 500 ORDER BY timestamp DESC LIMIT 100"
    },
    {
      name: "analyze_patterns",
      description: "Group errors by endpoint and error type"
    },
    {
      name: "check_deployments",
      description: "Look for recent deployments that may have caused the issue"
    },
    {
      name: "document_findings",
      description: "Create investigation with analysis"
    }
  ]
```

## Agent Patterns

### Error Investigation Agent

Automatically investigates when errors spike:
- Triggers on error rate alerts
- Queries recent errors and groups by type
- Correlates with deployments
- Creates investigations with findings

### Performance Monitor Agent

Watches for latency degradation:
- Runs on a schedule (e.g., every 5 minutes)
- Compares current latency to baseline
- Alerts if p99 exceeds threshold
- Identifies slow endpoints

### Deployment Validator Agent

Verifies deployments are healthy:
- Triggers after deployment events
- Compares error rates before/after
- Checks key SLIs
- Rolls back if issues detected

### Anomaly Detective Agent

Finds unusual patterns:
- Runs continuous analysis
- Detects statistical anomalies
- Investigates deviations
- Learns from false positives

## Best Practices

1. **Start simple**: Create focused agents that do one thing well
2. **Test incrementally**: Run agents manually before enabling triggers
3. **Set boundaries**: Define what agents should NOT do
4. **Add guardrails**: Require approval for destructive actions
5. **Monitor agents**: Review agent actions and outcomes regularly
6. **Iterate**: Improve instructions based on agent performance
