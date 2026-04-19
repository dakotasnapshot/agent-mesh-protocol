# Agent Mesh Protocol v1.0

A simple JSON-over-Matrix protocol for [OpenClaw](https://github.com/openclaw/openclaw) agent-to-agent communication.

## Overview

Agent Mesh lets multiple OpenClaw instances communicate using structured JSON messages sent through Matrix rooms. No custom servers needed — just Matrix accounts and DM rooms between agents.

## Why?

When you run multiple AI agents (different models, different roles), they need a way to coordinate. Natural language works for human-to-agent chat, but agent-to-agent communication benefits from structured payloads:

- **Deterministic routing** — message types map to handlers
- **Cheap** — no wasted tokens parsing "Hey, could you please..."
- **Auditable** — every message is a JSON object in a Matrix room
- **Extensible** — add new message types without breaking existing ones

## Prerequisites

- Two or more OpenClaw instances, each with:
  - A Matrix account (user + access token)
  - Matrix plugin enabled in OpenClaw config
  - DM allowlists updated to include peer agents

## Message Format

All messages are JSON objects with these required fields:

```json
{
  "protocol": "agent-mesh",
  "version": "1.0",
  "type": "<message-type>",
  "from": "<sender-agent-id>",
  "timestamp": "<ISO-8601>",
  "id": "<optional-unique-id>",
  "reply_to": "<optional-id-of-message-being-replied-to>",
  "payload": { ... }
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `protocol` | ✅ | Always `"agent-mesh"` — used to distinguish from human messages |
| `version` | ✅ | Protocol version (`"1.0"`) |
| `type` | ✅ | Message type (see below) |
| `from` | ✅ | Sender's agent ID |
| `timestamp` | ✅ | ISO-8601 timestamp |
| `id` | ❌ | Unique message ID for correlation |
| `reply_to` | ❌ | ID of the message being responded to |
| `payload` | ❌ | Type-specific data object |

## Message Types

### `ping` / `pong`

Health check between agents.

**Request:**
```json
{
  "protocol": "agent-mesh",
  "version": "1.0",
  "type": "ping",
  "from": "agent-alpha",
  "id": "ping-001",
  "timestamp": "2026-04-19T15:55:00Z"
}
```

**Response:**
```json
{
  "protocol": "agent-mesh",
  "version": "1.0",
  "type": "pong",
  "from": "agent-beta",
  "timestamp": "2026-04-19T15:55:01Z",
  "reply_to": "ping-001"
}
```

### `task`

Request another agent to perform work.

```json
{
  "protocol": "agent-mesh",
  "version": "1.0",
  "type": "task",
  "from": "agent-alpha",
  "id": "task-001",
  "timestamp": "2026-04-19T15:55:00Z",
  "payload": {
    "action": "research",
    "params": {
      "topic": "solar panel efficiency trends 2026",
      "max_sources": 5
    },
    "priority": "normal",
    "deadline": "2026-04-19T16:00:00Z"
  }
}
```

### `task_result`

Response to a task request.

```json
{
  "protocol": "agent-mesh",
  "version": "1.0",
  "type": "task_result",
  "from": "agent-beta",
  "timestamp": "2026-04-19T15:58:00Z",
  "reply_to": "task-001",
  "payload": {
    "status": "completed",
    "result": {
      "summary": "...",
      "sources": ["..."]
    },
    "error": null
  }
}
```

**Status values:** `accepted`, `in_progress`, `completed`, `failed`, `cancelled`

### `broadcast`

Announce information to all agents in a shared room.

```json
{
  "protocol": "agent-mesh",
  "version": "1.0",
  "type": "broadcast",
  "from": "agent-alpha",
  "timestamp": "2026-04-19T15:55:00Z",
  "payload": {
    "topic": "service_status",
    "message": "Monitoring alerts cleared"
  }
}
```

## Transport

| Method | Use Case |
|--------|----------|
| **Matrix DM** | Direct 1:1 communication between two agents |
| **Matrix Room** | Shared room for broadcasts and multi-agent coordination |

Messages are sent as standard `m.room.message` events with `msgtype: "m.text"` and the JSON payload as the body.

## Agent Registry

Maintain an agent registry (in a shared workspace file or config) so each agent knows its peers:

```json
{
  "agents": {
    "agent-alpha": {
      "matrix_user": "@alpha:example.com",
      "gateway": "http://10.0.0.1:18789",
      "role": "coordinator",
      "model": "claude-opus-4"
    },
    "agent-beta": {
      "matrix_user": "@beta:example.com",
      "gateway": "http://10.0.0.2:18789",
      "role": "research",
      "model": "qwen-plus"
    }
  }
}
```

## Message Handling

When an agent receives a Matrix message:

1. **Check for protocol marker:** Is the body valid JSON with `"protocol": "agent-mesh"`?
2. **If yes:** Parse the message type and route to the appropriate handler
3. **If no:** Treat as normal human conversation

### Example handler logic (pseudocode)

```
on_message(body):
  try:
    msg = JSON.parse(body)
    if msg.protocol == "agent-mesh":
      switch msg.type:
        "ping" → reply with pong
        "task" → execute and reply with task_result
        "broadcast" → log/process
        _ → ignore unknown types
  catch:
    handle_as_human_message(body)
```

## OpenClaw Setup

### 1. Create Matrix accounts for each agent

Register users on your Matrix homeserver (e.g., via Synapse admin API or `register_new_matrix_user`).

### 2. Configure OpenClaw Matrix plugin

Each agent's `openclaw.json`:

```json
{
  "plugins": {
    "entries": {
      "matrix": { "enabled": true }
    }
  },
  "channels": {
    "matrix": {
      "homeserverUrl": "http://your-synapse:8008",
      "userId": "@agent-name:your-domain.com",
      "accessToken": "<token>",
      "dm": {
        "enabled": true,
        "allowFrom": [
          "@human:your-domain.com",
          "@other-agent:your-domain.com"
        ]
      }
    }
  }
}
```

### 3. Add protocol awareness to agent workspace

Include the protocol spec in the agent's workspace (e.g., `AGENTS.md` or `docs/agent-mesh-protocol.md`) so the LLM knows how to handle structured messages.

### 4. Create DM rooms between agents

Use Matrix client API or a client app to create DM rooms and invite agents to each other's rooms.

## Design Principles

- **Matrix as transport** — no custom infrastructure; works with any Matrix homeserver
- **JSON over natural language** — structured data for machine-to-machine, English for human-to-machine
- **Protocol field as discriminator** — agents can easily distinguish mesh commands from human chat
- **Stateless messages** — each message is self-contained; agents don't need shared state
- **Extensible** — add new message types without breaking existing handlers

## Future Extensions

| Type | Purpose |
|------|---------|
| `capability_query` | Ask what an agent can do |
| `delegate` | Hand off a complex multi-step task |
| `stream` | Streaming partial results for long tasks |
| `cancel` | Abort a running task |
| `status` | Query agent health, load, model info |
| `file_transfer` | Share files between agents via Matrix media |

## License

MIT — use it however you want.
