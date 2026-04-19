# Agent Mesh Protocol v1.0

A simple JSON-over-Matrix protocol for AI agent-to-agent communication. Works with **OpenClaw**, **hermes-agent**, or any Matrix-connected agent.

## Why?

When you run multiple AI agents (different models, different roles), they need a way to coordinate. Natural language works for human-to-agent chat, but agent-to-agent communication benefits from structured payloads:

- **Deterministic routing** — message types map to handlers
- **Cheap** — no wasted tokens parsing "Hey, could you please..."
- **Auditable** — every message is a JSON object in a Matrix room
- **Extensible** — add new message types without breaking existing ones
- **Runtime agnostic** — OpenClaw, hermes-agent, custom Python/Node/etc.

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
  "runtime": "<optional: openclaw|hermes|custom>",
  "model": "<optional: model-name>",
  "id": "<optional-unique-id>",
  "reply_to": "<optional-id-being-replied-to>",
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

### Ping/Pong

```json
{"protocol":"agent-mesh","version":"1.0","type":"ping","from":"bucky","id":"ping-001","timestamp":"2026-04-19T15:55:00Z"}
```

```json
{"protocol":"agent-mesh","version":"1.0","type":"pong","from":"hermes","reply_to":"ping-001","timestamp":"2026-04-19T15:55:01Z","runtime":"hermes","model":"qwen-plus"}
```

### Task Delegation

```json
{
  "protocol": "agent-mesh",
  "version": "1.0",
  "type": "task",
  "from": "bucky",
  "id": "task-001",
  "timestamp": "2026-04-19T15:55:00Z",
  "payload": {
    "action": "research",
    "params": {"topic": "solar panel trends", "max_sources": 5},
    "priority": "normal"
  }
}
```

### Capability Discovery

```json
{"protocol":"agent-mesh","version":"1.0","type":"capability_query","from":"bucky","id":"cap-001"}
```

```json
{
  "protocol": "agent-mesh",
  "version": "1.0",
  "type": "capability_response",
  "from": "timber",
  "reply_to": "cap-001",
  "payload": {
    "runtime": "openclaw",
    "model": "qwen-plus",
    "capabilities": ["ping", "task", "broadcast"],
    "tools": ["web_fetch", "exec", "read", "write"]
  }
}
```

## Implementation Guides

| Runtime | Guide |
|---------|-------|
| **OpenClaw** | Add protocol spec to workspace; agents handle JSON naturally |
| **hermes-agent** | See [hermes-implementation.md](hermes-implementation.md) |
| **Custom** | Parse JSON, check `protocol` field, respond in kind |

## Design Principles

- **Matrix as transport** — no custom infrastructure needed
- **JSON over natural language** — structured for machines
- **Protocol field as discriminator** — easy to distinguish from human chat  
- **Stateless messages** — self-contained, no shared state required
- **Extensible** — add new types without breaking existing handlers

## Tested With

| Agent | Runtime | Model | Status |
|-------|---------|-------|--------|
| Bucky | OpenClaw | Claude Opus | ✅ Coordinator |
| Timber | OpenClaw | Qwen Plus | ✅ Connected |
| Hermes | hermes-agent | Qwen Plus | ✅ Connected |

## Future Extensions

- `stream` — streaming results for long tasks
- `cancel` — abort a running task
- `delegate` — hand off to a more capable agent
- `file_transfer` — share files via Matrix media

## License

MIT
