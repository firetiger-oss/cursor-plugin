---
name: firetiger-run
description: "Run an existing Firetiger agent"
user_invocable: true
user_invocable_description: "Start a session with a Firetiger agent"
---

# Firetiger Agent Execution Guide

You are an expert at running and interacting with Firetiger agents to automate observability workflows.

## Overview

Firetiger agents run in sessions - interactive conversations where you can give the agent tasks and receive responses. Sessions maintain context across multiple messages, allowing for complex multi-step workflows.

## Using MCP Tools

The Firetiger MCP server provides tools for running agents:

### List Available Agents

Find agents you can run:

```
list with resource: "agents"
```

This returns all configured agents with their descriptions and capabilities.

### Get Agent Details

Learn more about a specific agent:

```
get with name: "agents/{agent-id}"
```

This shows the agent's instructions, available tools, and configuration.

### Create a New Session

Start a conversation with an agent:

```
create with resource: "agents/{agent-id}/sessions"
```

This returns a session ID that you'll use for all subsequent interactions.

### Send a Message

Interact with the agent:

```
send_agent_message with session: "agents/{agent-id}/sessions/{session-id}"
  message: "Your message here"
```

The agent will process your message and respond. This is a synchronous operation that waits for the agent's response.

### Read Session History

Review the conversation:

```
read_agent_messages with session: "agents/{agent-id}/sessions/{session-id}"
```

This returns all messages in the session, useful for reviewing what the agent has done.

## Running an Agent

### 1. Find the Right Agent

List available agents to find one that matches your need:

```
list with resource: "agents"
```

Look for:
- **name**: The agent's identifier
- **display_name**: Human-readable name
- **description**: What the agent does

### 2. Review Agent Capabilities

Get details about the agent:

```
get with name: "agents/{agent-id}"
```

Understand:
- What tools the agent has access to
- What instructions guide its behavior
- What triggers it normally responds to

### 3. Start a Session

Create a new session:

```
create with resource: "agents/{agent-id}/sessions"
```

Note the session ID returned - you'll need it for all interactions.

### 4. Give the Agent a Task

Send your first message:

```
send_agent_message with session: "agents/{agent-id}/sessions/{session-id}"
  message: "Investigate the spike in 5xx errors from the checkout service in the last hour"
```

The agent will:
- Parse your request
- Plan its approach
- Execute necessary queries and actions
- Return a response with findings or results

### 5. Continue the Conversation

Send follow-up messages to refine or expand:

```
send_agent_message with session: "agents/{agent-id}/sessions/{session-id}"
  message: "Can you dig deeper into the payment processing errors specifically?"
```

The agent maintains context from previous messages.

### 6. Review the Session

Read the full conversation history:

```
read_agent_messages with session: "agents/{agent-id}/sessions/{session-id}"
```

This shows all messages and agent actions for review or documentation.

## Example Workflows

### Error Investigation

```
# 1. Start session with error investigator agent
create with resource: "agents/error-investigator/sessions"
# Returns: session-id: "sess_abc123"

# 2. Ask the agent to investigate
send_agent_message with session: "agents/error-investigator/sessions/sess_abc123"
  message: "There's been an increase in errors from the API gateway in the last 30 minutes. Can you investigate?"

# 3. Agent responds with initial findings, ask for more detail
send_agent_message with session: "agents/error-investigator/sessions/sess_abc123"
  message: "What's causing the 503 errors specifically?"

# 4. Ask for remediation
send_agent_message with session: "agents/error-investigator/sessions/sess_abc123"
  message: "Please create an investigation ticket and notify the on-call engineer"
```

### Performance Analysis

```
# 1. Start session with performance agent
create with resource: "agents/perf-analyzer/sessions"

# 2. Request analysis
send_agent_message
  message: "Compare the p99 latency of the checkout flow this week vs last week"

# 3. Drill down
send_agent_message
  message: "Which specific endpoint is contributing most to the latency increase?"

# 4. Get recommendations
send_agent_message
  message: "What optimizations would you recommend?"
```

### Deployment Validation

```
# 1. Start session with deployment validator
create with resource: "agents/deploy-validator/sessions"

# 2. Request validation
send_agent_message
  message: "We just deployed version 2.3.1 to the payment service. Can you validate the deployment is healthy?"

# 3. Agent runs checks and reports back
# 4. Ask for specific metrics
send_agent_message
  message: "How do the error rates compare to the previous version?"
```

## Tips for Effective Agent Interactions

1. **Be specific**: Give clear context about what you want investigated
2. **Include time ranges**: Specify when the issue occurred
3. **Name services**: Mention specific services or endpoints
4. **Ask follow-ups**: Drill down into findings with additional questions
5. **Request actions**: Ask the agent to create investigations, send alerts, etc.

## Troubleshooting

### Agent Not Responding

If `send_agent_message` doesn't return:
- Check if the agent is enabled
- Verify the session ID is correct
- Review agent logs for errors

### Unexpected Results

If the agent's response doesn't match expectations:
- Review the agent's instructions with `get`
- Check what tools the agent has access to
- Be more specific in your request

### Session Context Lost

If the agent seems to forget previous context:
- Use `read_agent_messages` to verify session history
- Ensure you're using the same session ID
- Start a new session if the old one is corrupted
