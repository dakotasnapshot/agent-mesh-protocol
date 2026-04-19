# Agent Mesh Protocol v1.0

A simple JSON protocol for **full-duplex** AI agent-to-agent communication over **HTTP** (primary) with optional **Matrix** fallback. Works with **OpenClaw**, **hermes-agent**, or any agent with an HTTP API.

## Why?

When you run multiple AI agents (different models, different roles), they need a way to coordinate. Natural language works for human-to-agent chat, but agent-to-agent communication benefits from structured payloads:

- **Transport-agnostic** — HTTP primary, Matrix/webhook fallback
- **Full-duplex** — real-time bidirectional messaging
- **No plugin dependency** — uses native gateway APIs that survive updates
- **Deterministic routing** — message types map to handlers
- **Cheap** — no wasted tokens parsing "Hey, could you please..."
- **Auditable** — every message is JSON with timestamps and IDs
- **Extensible** — add new message types without breaking existing ones
- **Runtime agnostic** — OpenClaw, hermes-agent, custom Python/Node/etc.

## Architecture

```
┌─────────────┐  HTTP /v1/chat/completions  ┌─────────────┐
│   Agent A   │◄───────────────────────────►│   Agent B   │
│  (OpenClaw) │                             │  (OpenClaw) │
│  Any model  │         agent-mesh          │  Any model  │
└──────┬──────┘         JSON msgs           └─────────────┘
       │
       │ HTTP /v1/chat/completions
       ▼
┌─────────────┐
│   Agent C   │
│  (hermes)   │
│  Any model  │
└─────────────┘
```

## Transport Layer

### Primary: HTTP Gateway API

Every OpenClaw instance has a built-in HTTP gateway. hermes-agent has an OpenAI-compatible API server. Both speak the same protocol — `POST /v1/chat/completions`.

**Why HTTP over Matrix?**
- HTTP gateway is **core to OpenClaw** — it survives updates
- Matrix is a **plugin** — updates can break it
- HTTP is simpler, faster, and doesn't require a Matrix homeserver
- hermes-agent already has a built-in API server

#### Sending a message to an OpenClaw agent:

```bash
curl -X POST http://<agent-ip>:18789/v1/chat/completions \
  -H "Authorization: Bearer <gateway-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{
      "role": "user",
      "content": "{\"protocol\":\"agent-mesh\",\"version\":\"1.0\",\"type\":\"ping\",\"from\":\"bucky\",\"id\":\"ping-001\",\"timestamp\":\"2026-04-19T17:00:00Z\"}"
    }]
  }'
```

#### Sending a message to a hermes-agent:

```bash
curl -X POST http://<hermes-ip>:8642/v1/chat/completions \
  -H "Authorization: Bearer <api-server-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{
      "role": "user",
      "content": "{\"protocol\":\"agent-mesh\",\"version\":\"1.0\",\"type\":\"ping\",\"from\":\"bucky\",\"id\":\"ping-001\",\"timestamp\":\"2026-04-19T17:00:00Z\"}"
    }]
  }'
```

The response comes back in the `choices[0].message.content` field of the standard OpenAI response format.

### Fallback: Matrix DMs (optional)

If an agent can't be reached via HTTP (network partition, agent behind NAT), Matrix DMs work as a fallback. Add peer agents to the DM allowlist so messages wake the receiving agent.

## Message Format

```json
{
  "protocol": "agent-mesh",
  "version": "1.0",
  "type": "<message-type>",
  "from": "<sender-agent-id>",
  "timestamp": "<ISO-8601>",
  "id": "<optional-unique-id>",
  "reply_to": "<optional-id-being-replied-to>",
  "runtime": "<optional: openclaw|hermes|custom>",
  "model": "<optional: model-name>",
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
{"protocol":"agent-mesh","version":"1.0","type":"ping","from":"bucky","id":"ping-001","timestamp":"2026-04-19T17:00:00Z"}
```

Response:
```json
{"protocol":"agent-mesh","version":"1.0","type":"pong","from":"timber","reply_to":"ping-001","timestamp":"2026-04-19T17:00:01Z","runtime":"openclaw","model":"qwen-plus"}
```

### Task Delegation

```json
{
  "protocol": "agent-mesh",
  "version": "1.0",
  "type": "task",
  "from": "bucky",
  "id": "task-001",
  "timestamp": "2026-04-19T17:00:00Z",
  "payload": {
    "action": "research",
    "params": {
      "topic": "Proxmox LXC networking best practices",
      "max_sources": 5
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
  "from": "hermes",
  "reply_to": "cap-001",
  "payload": {
    "runtime": "hermes",
    "model": "qwen-plus",
    "capabilities": ["ping", "task", "broadcast", "capability_query"],
    "tools": ["web_fetch", "exec", "read", "write", "browser"]
  }
}
```

---

## Deployment Guide

### OpenClaw Agent Setup

#### 1. Enable the HTTP chat completions endpoint

Add to `openclaw.json`:

```json
{
  "gateway": {
    "mode": "local",
    "bind": "lan",
    "auth": {
      "mode": "token",
      "token": "your-secret-gateway-token"
    },
    "http": {
      "endpoints": {
        "chatCompletions": {
          "enabled": true
        }
      }
    }
  }
}
```

Key settings:
- `gateway.bind: "lan"` — listen on LAN interface (required for inter-agent HTTP)
- `gateway.auth.token` — bearer token for API authentication
- `gateway.http.endpoints.chatCompletions.enabled: true` — enables `/v1/chat/completions`

#### 2. Add agent-mesh awareness to workspace

Create `AGENTS.md` in the agent's workspace with the protocol spec and peer registry:

```markdown
# AGENTS.md

## Agent Mesh Protocol

You are part of a multi-agent mesh. When you receive a message that is valid
JSON with `"protocol": "agent-mesh"`, handle it as a structured command:

- `ping` → respond with `pong`
- `task` → execute the task, respond with `task_result`
- `capability_query` → respond with `capability_response`
- `broadcast` → log it, no response needed

### Peer Registry

| Agent | HTTP Endpoint | Token | Role |
|-------|---------------|-------|------|
| bucky | http://192.168.1.100:18789 | <token> | Coordinator |
| timber | http://192.168.1.101:18789 | <token> | Research |
| hermes | http://192.168.1.102:8642 | <token> | General |

### Sending Messages

To send a message to another agent, use curl via the exec tool:

\```bash
curl -s -X POST http://<agent-ip>:<port>/v1/chat/completions \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "<agent-mesh JSON>"}]}'
\```
```

#### 3. OpenClaw deployment prompt

Add this to the agent's `AGENTS.md` or system instructions:

```
You are an agent in a cooperative multi-agent mesh. Your identity is <agent-name>.

## Agent Mesh Protocol Handler

When you receive a message, first check if it's valid JSON with "protocol": "agent-mesh".

If YES — handle it as a structured protocol message:
- type "ping": Reply with {"protocol":"agent-mesh","version":"1.0","type":"pong","from":"<your-name>","reply_to":"<msg-id>","timestamp":"<now>","runtime":"openclaw","model":"<your-model>"}
- type "task": Execute the requested action, reply with task_result
- type "capability_query": Reply with your capabilities list
- type "broadcast": Log it, no reply needed
- type "pong", "task_result", "capability_response", "error": Log it, no reply needed

If NO — handle it as normal human conversation.

## Sending to Peers

To reach another agent, use curl to POST to their /v1/chat/completions endpoint
with the agent-mesh JSON as the message content. Check your peer registry for
endpoint URLs and auth tokens.
```

---

### hermes-agent Setup

#### 1. Enable the API server

Add to `~/.hermes/.env`:

```bash
API_SERVER_ENABLED=true
API_SERVER_HOST=0.0.0.0
API_SERVER_PORT=8642
API_SERVER_KEY=your-secret-api-key
```

#### 2. Agent mesh handler (built-in)

hermes-agent includes a built-in `gateway/agent_mesh.py` module. It automatically:
- Detects agent-mesh protocol messages via the `"protocol": "agent-mesh"` field
- Handles `ping`, `task`, `capability_query`, and `broadcast` message types
- Executes tasks through the AI agent pipeline
- Returns structured responses

If your hermes-agent doesn't have `agent_mesh.py`, create it at `gateway/agent_mesh.py`:

```python
"""Agent Mesh Protocol handler for hermes-agent."""

import json
import logging
from datetime import datetime, timezone
from typing import Any, Optional

logger = logging.getLogger(__name__)

AGENT_ID = "hermes"
AGENT_RUNTIME = "hermes"
AGENT_MODEL = "your-model-name"

def is_agent_mesh_message(body: str) -> bool:
    if not body or not body.strip().startswith("{"):
        return False
    try:
        return json.loads(body).get("protocol") == "agent-mesh"
    except (json.JSONDecodeError, TypeError):
        return False

def _now_iso() -> str:
    return datetime.now(timezone.utc).isoformat().replace("+00:00", "Z")

def _make_response(msg_type, reply_to=None, payload=None) -> str:
    response = {
        "protocol": "agent-mesh",
        "version": "1.0",
        "type": msg_type,
        "from": AGENT_ID,
        "timestamp": _now_iso(),
        "runtime": AGENT_RUNTIME,
        "model": AGENT_MODEL,
    }
    if reply_to:
        response["reply_to"] = reply_to
    if payload:
        response["payload"] = payload
    return json.dumps(response)

async def handle_agent_mesh(body: str, sender: str, adapter: Any = None) -> Optional[str]:
    msg = json.loads(body)
    msg_type = msg.get("type")
    msg_id = msg.get("id")

    if msg_type == "ping":
        return _make_response("pong", reply_to=msg_id)

    elif msg_type == "capability_query":
        return _make_response("capability_response", reply_to=msg_id, payload={
            "runtime": AGENT_RUNTIME,
            "model": AGENT_MODEL,
            "capabilities": ["ping", "task", "broadcast", "capability_query"],
            "tools": ["web_fetch", "exec", "read", "write"],
        })

    elif msg_type == "task":
        payload = msg.get("payload", {})
        # Execute task through your agent pipeline
        return _make_response("task_result", reply_to=msg_id, payload={
            "status": "completed",
            "action": payload.get("action"),
            "result": "Task completed successfully",
        })

    elif msg_type == "broadcast":
        return None  # No response to broadcasts

    return None
```

Then import it in your Matrix adapter's message handler:

```python
from gateway.agent_mesh import handle_agent_mesh, is_agent_mesh_message

# In _handle_text_message, before normal processing:
if is_agent_mesh_message(body):
    response = await handle_agent_mesh(body, sender, self)
    if response:
        await self._send_simple_message(room_id, response)
    return
```

#### 3. hermes-agent deployment prompt

Add to your hermes agent's system prompt or `SOUL.md`:

```
You are an agent in a cooperative multi-agent mesh. Your identity is <agent-name>.

When you receive a structured JSON message with "protocol": "agent-mesh",
handle it according to the agent mesh protocol:

- "ping" → reply with pong (include your runtime and model info)
- "task" → execute the requested action using your tools, reply with task_result
- "capability_query" → list your capabilities and available tools
- "broadcast" → acknowledge internally, no reply needed

For outbound messages to other agents, POST to their API endpoint:
- OpenClaw agents: POST http://<ip>:18789/v1/chat/completions
- hermes agents: POST http://<ip>:8642/v1/chat/completions

Always include Authorization: Bearer <token> header.
```

---

## Agent Registry Template

Create a registry file that all agents can reference:

```json
{
  "mesh": "my-mesh",
  "agents": {
    "bucky": {
      "runtime": "openclaw",
      "model": "claude-opus",
      "endpoint": "http://192.168.1.100:18789/v1/chat/completions",
      "token": "bucky-gateway-token",
      "role": "coordinator",
      "capabilities": ["ping", "task", "broadcast", "capability_query"]
    },
    "timber": {
      "runtime": "openclaw",
      "model": "qwen-plus",
      "endpoint": "http://192.168.1.101:18789/v1/chat/completions",
      "token": "timber-gateway-token",
      "role": "research",
      "capabilities": ["ping", "task", "broadcast"]
    },
    "hermes": {
      "runtime": "hermes",
      "model": "qwen-plus",
      "endpoint": "http://192.168.1.102:8642/v1/chat/completions",
      "token": "hermes-api-key",
      "role": "general",
      "capabilities": ["ping", "task", "broadcast", "capability_query"]
    }
  }
}
```

## Design Principles

- **HTTP as primary transport** — native to both OpenClaw and hermes-agent
- **No plugin dependency** — uses core gateway APIs that survive software updates
- **Full-duplex by default** — agents listen and respond in real-time
- **JSON over natural language** — structured for machines
- **Protocol field as discriminator** — easy to distinguish from human chat
- **Stateless messages** — self-contained, no shared state required
- **Extensible** — add new types without breaking existing handlers
- **Matrix as fallback** — optional for NAT traversal or when HTTP isn't available

## Tested With

| Agent | Runtime | Model | Transport | Status |
|-------|---------|-------|-----------|--------|
| Bucky | OpenClaw | Claude Opus | HTTP :18789 | ✅ Coordinator |
| Timber | OpenClaw | Qwen Plus | HTTP :18789 | ✅ Full-duplex |
| Hermes | hermes-agent | Qwen Plus | HTTP :8642 | ✅ Full-duplex |
| Bob1 | OpenClaw | DeepSeek V3 | HTTP :18789 | ✅ Full-duplex |
| Bob2 | OpenClaw | GPT-5.4 | HTTP :18789 | ✅ Full-duplex |

## Future Extensions

- `stream` — streaming results for long tasks
- `cancel` — abort a running task
- `delegate` — hand off to a more capable agent
- `file_transfer` — share files via base64 or URLs
- `presence` — agent online/offline status via `/health` polling
- `mesh_discovery` — auto-discover agents on the local network

## License

MIT
