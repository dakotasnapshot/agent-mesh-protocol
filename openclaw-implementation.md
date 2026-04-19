# OpenClaw Implementation Guide

## Overview

OpenClaw agents participate in the mesh via their built-in HTTP gateway, specifically the OpenAI-compatible `/v1/chat/completions` endpoint.

## Quick Setup

### 1. Enable Chat Completions Endpoint

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

Or via the gateway tool:

```
gateway(action="config.patch", raw='{"gateway":{"bind":"lan","http":{"endpoints":{"chatCompletions":{"enabled":true}}}}}')
```

### 2. Restart

```bash
openclaw gateway restart
```

### 3. Verify

```bash
curl -s http://<agent-ip>:18789/health \
  -H "Authorization: Bearer <gateway-token>"
```

## Configuration Reference

| Setting | Type | Description |
|---------|------|-------------|
| `gateway.bind` | `"local"` / `"lan"` | Network bind mode. `"lan"` required for mesh. |
| `gateway.auth.mode` | `"token"` | Auth mode |
| `gateway.auth.token` | string | Bearer token for API access |
| `gateway.http.endpoints.chatCompletions.enabled` | boolean | Enable /v1/chat/completions |
| `gateway.http.endpoints.chatCompletions.maxBodyBytes` | integer | Max request size (default 20MB) |

## Agent Mesh Awareness

OpenClaw agents handle agent-mesh messages through their normal conversation flow. The key is giving the agent context about the protocol in its workspace files.

### AGENTS.md Template

Place this in the agent's workspace (`~/.openclaw/workspace/AGENTS.md`):

```markdown
# Agent Mesh Protocol

You are part of a multi-agent mesh. Your agent ID is `<your-name>`.

## Protocol Handling

When you receive a message that is valid JSON with `"protocol": "agent-mesh"`:

1. Parse the JSON
2. Check the `type` field
3. Handle accordingly:
   - `ping` → Reply with pong JSON
   - `task` → Execute the action, reply with task_result JSON
   - `capability_query` → Reply with your capabilities
   - `broadcast` → Log it, no reply needed

## Response Format

Always respond with valid agent-mesh JSON:

```json
{
  "protocol": "agent-mesh",
  "version": "1.0",
  "type": "<response-type>",
  "from": "<your-name>",
  "reply_to": "<original-msg-id>",
  "timestamp": "<ISO-8601 UTC>",
  "runtime": "openclaw",
  "model": "<your-model>",
  "payload": { ... }
}
```

## Peer Registry

| Agent | Endpoint | Token |
|-------|----------|-------|
| bucky | http://192.168.1.100:18789/v1/chat/completions | <token> |
| timber | http://192.168.1.101:18789/v1/chat/completions | <token> |
| hermes | http://192.168.1.102:8642/v1/chat/completions | <token> |
```

### Sending Messages to Peers

From within an OpenClaw session, use the `exec` tool:

```bash
curl -s -X POST http://<peer-ip>:<port>/v1/chat/completions \
  -H "Authorization: Bearer <peer-token>" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "<agent-mesh JSON string>"}]}'
```

Parse the response from `choices[0].message.content`.

## Default Port

OpenClaw gateway listens on port **18789** by default.

## Security Notes

- Gateway tokens authenticate API access
- Use `gateway.bind: "lan"` only on trusted networks
- Each agent should have a unique gateway token
- Tokens are never shared in the public protocol spec
