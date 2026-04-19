# Agent Mesh Protocol v1.0

A simple JSON-over-Matrix protocol for **full-duplex** AI agent-to-agent communication. Works with **OpenClaw**, **hermes-agent**, or any Matrix-connected agent.

## Why?

When you run multiple AI agents (different models, different roles), they need a way to coordinate. Natural language works for human-to-agent chat, but agent-to-agent communication benefits from structured payloads:

- **Full-duplex** — real-time bidirectional messaging, not request-response polling
- **Deterministic routing** — message types map to handlers
- **Cheap** — no wasted tokens parsing "Hey, could you please..."
- **Auditable** — every message is a JSON object in a Matrix room
- **Extensible** — add new message types without breaking existing ones
- **Runtime agnostic** — OpenClaw, hermes-agent, custom Python/Node/etc.

## Architecture

```
┌─────────────┐     Matrix DM      ┌─────────────┐
│   Bucky     │◄──────────────────►│   Timber    │
│  (OpenClaw) │                    │  (OpenClaw) │
│  Claude     │     agent-mesh     │  Qwen       │
└──────┬──────┘     JSON msgs      └─────────────┘
       │
       │ Matrix DM
       ▼
┌─────────────┐
│   Hermes    │
│  (hermes)   │
│  Qwen       │
└─────────────┘
```

Each agent:
1. Has a Matrix account
2. Maintains a persistent Matrix sync
3. Receives messages in real-time (not polling)
4. Handles `agent-mesh` protocol messages automatically
5. Passes non-protocol messages to normal conversation flow

## Full-Duplex Setup

For true bidirectional communication, each agent must:

1. **Add peer agents to DM allowlist** — so incoming messages trigger handlers
2. **Maintain Matrix sync** — persistent connection, not periodic polling
3. **Handle inbound protocol messages** — respond to pings, execute tasks, log broadcasts

### OpenClaw Example

```json
{
  "channels": {
    "matrix": {
      "dm": {
        "allowFrom": [
          "@dakota:matrix.lert.fyi",
          "@timber:matrix.lert.fyi",
          "@hermes:matrix.lert.fyi"
        ]
      }
    }
  }
}
```

### hermes-agent Example

```bash
MATRIX_ALLOWED_USERS=@dakota:matrix.lert.fyi,@bucky:matrix.lert.fyi,@timber:matrix.lert.fyi
```

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

| Type | Purpose | Response Expected |
|------|---------|-------------------|
| `ping` | Health check | `pong` |
| `pong` | Health check response | No |
| `task` | Request work | `task_result` |
| `task_result` | Work completion | No |
| `broadcast` | Announce to all | No |
| `capability_query` | Discover abilities | `capability_response` |
| `capability_response` | Abilities list | No |
| `error` | Report errors | No |

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
    "action": "send_message",
    "params": {
      "to": "@dakota:matrix.lert.fyi",
      "message": "Hello from the mesh!"
    }
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
| **OpenClaw** | Add peer agents to `channels.matrix.dm.allowFrom`; protocol handling is automatic |
| **hermes-agent** | See [hermes-implementation.md](hermes-implementation.md) |
| **Custom** | Parse JSON, check `protocol` field, respond in kind |

## Design Principles

- **Matrix as transport** — no custom infrastructure needed
- **Full-duplex by default** — agents listen and respond in real-time
- **JSON over natural language** — structured for machines
- **Protocol field as discriminator** — easy to distinguish from human chat
- **Stateless messages** — self-contained, no shared state required
- **Extensible** — add new types without breaking existing handlers

## Tested With

| Agent | Runtime | Model | Status |
|-------|---------|-------|--------|
| Bucky | OpenClaw | Claude Opus | ✅ Coordinator |
| Timber | OpenClaw | Qwen Plus | ✅ Full-duplex |
| Hermes | hermes-agent | Qwen Plus | ✅ Full-duplex |

## Future Extensions

- `stream` — streaming results for long tasks
- `cancel` — abort a running task
- `delegate` — hand off to a more capable agent
- `file_transfer` — share files via Matrix media
- `presence` — agent online/offline status

## License

MIT
