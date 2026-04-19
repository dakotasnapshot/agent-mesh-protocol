# hermes-agent Implementation Guide

## Overview

hermes-agent has built-in support for the Agent Mesh Protocol via:
1. **API Server** — OpenAI-compatible HTTP endpoint (`/v1/chat/completions`)
2. **Agent Mesh Handler** — `gateway/agent_mesh.py` module for structured message handling
3. **Matrix integration** — Optional fallback transport

## Quick Setup

### 1. Enable the API Server

Add to `~/.hermes/.env`:

```bash
API_SERVER_ENABLED=true
API_SERVER_HOST=0.0.0.0
API_SERVER_PORT=8642
API_SERVER_KEY=your-secret-api-key
```

### 2. Restart the Gateway

```bash
hermes gateway stop
hermes gateway run --replace
# or
systemctl restart hermes-gateway
```

### 3. Verify

```bash
curl http://localhost:8642/health
# {"status": "ok", "platform": "hermes-agent"}
```

## API Server Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Health check |
| `/health/detailed` | GET | Rich status (model, uptime, etc.) |
| `/v1/chat/completions` | POST | OpenAI-compatible chat (agent-mesh messages go here) |
| `/v1/models` | GET | List available models |
| `/v1/runs` | POST | Start async run |
| `/v1/runs/{id}/events` | GET | SSE stream for run events |

## Agent Mesh Handler

The `gateway/agent_mesh.py` module provides:

- `is_agent_mesh_message(body)` — checks if a message is agent-mesh protocol
- `handle_agent_mesh(body, sender, adapter)` — processes the message and returns a response
- Auto-discovery of tools from the `tools/` directory
- Model detection from `config.yaml`

### Supported Message Types

| Type | Handler | Response |
|------|---------|----------|
| `ping` | Returns pong with runtime/model info | `pong` |
| `capability_query` | Lists tools and capabilities | `capability_response` |
| `task` | Routes to AIAgent for execution | `task_result` |
| `broadcast` | Logs, no response | None |
| `pong` / `task_result` / `error` | Logs | None |

### Task Execution

When a `task` message is received, the handler:
1. Extracts `action` and `params` from the payload
2. Maps common actions to natural language prompts:
   - `research` → "Research this topic and provide sources"
   - `execute` → "Execute this shell command"
   - `file_operation` → "Read/write file"
   - `web_fetch` → "Fetch and summarize URL"
3. Runs the prompt through the full AIAgent pipeline (with tools)
4. Returns the result (truncated to 4000 chars)

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `API_SERVER_ENABLED` | `false` | Enable the API server |
| `API_SERVER_HOST` | `127.0.0.1` | Bind address |
| `API_SERVER_PORT` | `8642` | Listen port |
| `API_SERVER_KEY` | (none) | Bearer token for auth |
| `API_SERVER_CORS_ORIGINS` | (none) | Comma-separated CORS origins |
| `API_SERVER_MODEL_NAME` | (auto) | Override model name in /v1/models |

## Security

- Always set `API_SERVER_KEY` in production
- Use `API_SERVER_HOST=127.0.0.1` if only local access is needed
- Use `API_SERVER_HOST=0.0.0.0` for LAN mesh access
- The API key is sent as `Authorization: Bearer <key>` header
