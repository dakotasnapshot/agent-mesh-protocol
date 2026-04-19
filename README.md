# Agent Mesh Protocol v1.0

A simple JSON-over-Matrix protocol for agent-to-agent communication. Works with OpenClaw, hermes-agent, or any Matrix-connected agent.

## Overview

Agent Mesh lets multiple AI agents communicate using structured JSON messages sent through Matrix rooms. No custom servers needed — just Matrix accounts and DM rooms between agents.

## Why?

When you run multiple AI agents (different models, different roles), they need a way to coordinate. Natural language works for human-to-agent chat, but agent-to-agent communication benefits from structured payloads:

- **Deterministic routing** — message types map to handlers
- **Cheap** — no wasted tokens parsing "Hey, could you please..."
- **Auditable** — every message is a JSON object in a Matrix room
- **Extensible** — add new message types without breaking existing ones

## Quick Start

1. Give each agent a Matrix account
2. Add the agent-mesh handler to each agent's message processing
3. Create DM rooms between agents
4. Send structured JSON messages

## Message Format

```json
{
  "protocol": "agent-mesh",
  "version": "1.0",
  "type": "<message-type>",
  "from": "<sender-agent-id>",
  "timestamp": "<ISO-8601>",
  "id": "<optional-unique-id>",
  "reply_to": "<optional-id-of-message-being-replied-to>",
  "runtime": "<optional: openclaw|hermes|custom>",
  "payload": { ... }
}
```

## Message Types

| Type | Purpose |
|------|---------|
| `ping` / `pong` | Health check |
| `task` | Request work from another agent |
| `task_result` | Response to a task |
| `broadcast` | Announce to all agents |
| `capability_query` / `capability_response` | Discover agent capabilities |
| `error` | Report errors |

## Examples

### Ping

```json
{"protocol":"agent-mesh","version":"1.0","type":"ping","from":"agent-alpha","id":"ping-001","timestamp":"2026-04-19T15:55:00Z"}
```

### Pong

```json
{"protocol":"agent-mesh","version":"1.0","type":"pong","from":"agent-beta","timestamp":"2026-04-19T15:55:01Z","reply_to":"ping-001","runtime":"hermes","model":"gpt-4o"}
```

### Task

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
    "priority": "normal"
  }
}
```

### Capability Query

```json
{"protocol":"agent-mesh","version":"1.0","type":"capability_query","from":"agent-alpha","id":"cap-001","timestamp":"2026-04-19T15:55:00Z"}
```

### Capability Response

```json
{
  "protocol": "agent-mesh",
  "version": "1.0",
  "type": "capability_response",
  "from": "agent-beta",
  "reply_to": "cap-001",
  "timestamp": "2026-04-19T15:55:01Z",
  "payload": {
    "runtime": "hermes",
    "model": "gpt-4o",
    "capabilities": ["ping", "task", "broadcast"],
    "tools": ["web_fetch", "exec", "read", "write"]
  }
}
```

## Implementation

### OpenClaw

Add the protocol spec to your agent's workspace (`AGENTS.md` or `docs/agent-mesh-protocol.md`). OpenClaw agents handle JSON messages naturally.

### hermes-agent

See [hermes-implementation.md](hermes-implementation.md) for the Python handler code.

### Custom

Any Matrix client can participate:
1. Listen for DM messages
2. Check if body is JSON with `"protocol": "agent-mesh"`
3. Handle structured messages, respond in kind
4. Pass non-JSON messages to normal conversation flow

## Design Principles

- **Matrix as transport** — no custom infrastructure
- **JSON over natural language** — structured for machines
- **Protocol field as discriminator** — easy to distinguish from human chat
- **Stateless messages** — self-contained, no shared state needed
- **Extensible** — add new types without breaking existing handlers

## Future Extensions

- `stream` — streaming results for long tasks
- `cancel` — abort a running task  
- `delegate` — hand off to a more capable agent
- `file_transfer` — share files via Matrix media

## License

MIT
