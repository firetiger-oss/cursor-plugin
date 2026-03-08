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
- Execute investigations and analyze telemetry data
- Interact with external systems via configured connections
- Be triggered on schedules or invoked manually

## Using MCP Tools

The Firetiger MCP server provides tools for creating agents and triggers.

**Important:** Always use `schema` first to discover the current field definitions before creating resources. The examples below are illustrative - verify actual field names with the schema tool.

### Discover Schemas

```
schema with collection: "agents"
schema with collection: "triggers"
```

### List Existing Resources

```
list with resource: "agents"
list with resource: "triggers"
```

### Create Resources

```
create with resource: "agents"
create with resource: "triggers"
```

## Agent Structure

Agents have these core fields (verify with `schema`):

- **name**: Resource name (format: `agents/{agent-id}`)
- **title**: Human-readable title
- **description**: What the agent does
- **prompt**: The initial prompt that guides agent behavior
- **connections**: List of enabled tool connections
- **state**: Agent state (`AGENT_STATE_ON`, `AGENT_STATE_OFF`, etc.)

## Trigger Structure

Triggers are **separate resources** that invoke agents. They are not embedded in the agent definition.

Trigger fields:
- **name**: Resource name (format: `triggers/{trigger-id}`)
- **display_name**: Human-readable name (required)
- **description**: What the trigger does
- **agent**: Target agent (format: `agents/{agent-id}`, required)
- **enabled**: Boolean flag
- **configuration**: One of the following:

### Cron Triggers

Schedule agents to run periodically:

```
configuration:
  cron:
    schedule: "0 9 * * *"      # Standard 5-field cron expression
    timezone: "America/New_York"  # IANA timezone (default: UTC)
```

Cron examples:
- `0 9 * * *` - Daily at 9 AM
- `*/15 * * * *` - Every 15 minutes
- `0 0 * * 1` - Weekly on Monday at midnight

### Manual Triggers

For on-demand invocation only:

```
configuration:
  manual: {}
```

## Agent Design Process

### 1. Define the Agent's Purpose

Before creating an agent, clearly define:
- **What problem does it solve?** (e.g., "Analyze latency trends daily")
- **How will it be triggered?** (cron schedule or manual invocation)
- **What connections does it need?** (query, notifications, tickets)

### 2. Write Clear Prompts

Agent prompts should be:
- **Specific**: Define exactly what the agent should do
- **Scoped**: Limit the agent's responsibilities
- **Actionable**: Include concrete steps to follow

Example prompt:
```
You are a daily health check agent for the checkout service.

Your job:
1. Query the last 24 hours of traces for the checkout service
2. Calculate error rates and p99 latency
3. Compare against the previous 24-hour period
4. If error rate increased >10% or p99 increased >20%, create an investigation
5. Summarize findings in a brief report

Focus only on the checkout service. Do not investigate other services.
```

### 3. Create the Agent

```
create with resource: "agents"
  title: "Checkout Health Monitor"
  description: "Daily health analysis of the checkout service"
  prompt: "You are a daily health check agent..."
  state: "AGENT_STATE_ON"
```

### 4. Create a Trigger

Create a separate trigger to schedule or enable invocation:

```
create with resource: "triggers"
  display_name: "Daily Checkout Health Check"
  description: "Runs checkout health analysis every morning"
  agent: "agents/{agent-id}"
  enabled: true
  configuration:
    cron:
      schedule: "0 9 * * *"
      timezone: "America/Los_Angeles"
```

## Agent Patterns

### Daily Health Monitor

Runs scheduled health checks:
- **Trigger**: Cron schedule (e.g., daily at 9 AM)
- **Actions**: Query metrics, compare to baseline, report anomalies

### On-Demand Investigator

Available for manual invocation when issues arise:
- **Trigger**: Manual (invoked by users or external systems)
- **Actions**: Deep-dive analysis, correlation, root cause identification

### Periodic Report Generator

Creates regular summary reports:
- **Trigger**: Cron schedule (e.g., weekly on Monday)
- **Actions**: Aggregate data, generate insights, send to stakeholders

## Best Practices

1. **Start simple**: Create focused agents that do one thing well
2. **Test manually first**: Use manual triggers to validate agent behavior before enabling cron schedules
3. **Set boundaries**: Define what agents should NOT do in the prompt
4. **Use appropriate schedules**: Don't run agents more frequently than needed
5. **Monitor agents**: Review agent sessions and outcomes regularly
6. **Iterate on prompts**: Improve prompts based on agent performance
