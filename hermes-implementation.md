# hermes-agent Implementation

Complete guide to adding Agent Mesh Protocol support to hermes-agent.

## Overview

This adds structured JSON communication to hermes-agent, allowing it to participate in a multi-agent mesh alongside OpenClaw and other agents.

## 1. Create the handler module

Save as `gateway/agent_mesh.py` in your hermes-agent directory:

```python
"""
Agent Mesh Protocol handler for hermes-agent.

Enables structured JSON communication with other agents (OpenClaw, etc.)
over Matrix DMs.
"""

import json
import logging
from datetime import datetime, timezone
from typing import Any, Optional

logger = logging.getLogger(__name__)

# Agent identity — customize these for your instance
AGENT_ID = "hermes"
AGENT_MODEL = "qwen-plus"  # or gpt-4o, claude, etc.
AGENT_RUNTIME = "hermes"

# Supported capabilities
CAPABILITIES = [
    "ping",
    "task",
    "broadcast",
    "capability_query",
]

TOOLS = [
    "web_fetch",
    "exec",
    "read",
    "write",
    "search",
]


def is_agent_mesh_message(body: str) -> bool:
    """Check if a message is an agent-mesh protocol message."""
    if not body or not body.strip().startswith("{"):
        return False
    try:
        msg = json.loads(body)
        return msg.get("protocol") == "agent-mesh"
    except (json.JSONDecodeError, TypeError):
        return False


def _now_iso() -> str:
    """Return current UTC timestamp in ISO format."""
    return datetime.now(timezone.utc).isoformat().replace("+00:00", "Z")


def _make_response(
    msg_type: str,
    reply_to: Optional[str] = None,
    payload: Optional[dict] = None,
) -> str:
    """Build an agent-mesh response message."""
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


async def handle_agent_mesh(
    body: str,
    sender: str,
    adapter: Any = None,
) -> Optional[str]:
    """
    Handle an agent-mesh protocol message.
    
    Args:
        body: The raw message body (JSON string)
        sender: Matrix user ID of the sender
        adapter: The Matrix adapter instance (for advanced operations)
    
    Returns:
        JSON response string, or None if no response needed
    """
    try:
        msg = json.loads(body)
    except (json.JSONDecodeError, TypeError) as e:
        logger.warning(f"Failed to parse agent-mesh message: {e}")
        return _make_response("error", payload={
            "code": "parse_error",
            "message": str(e),
        })
    
    msg_type = msg.get("type")
    msg_id = msg.get("id")
    msg_from = msg.get("from", "unknown")
    
    logger.info(f"[agent-mesh] Received {msg_type} from {msg_from}")
    
    # Handle different message types
    
    if msg_type == "ping":
        return _make_response("pong", reply_to=msg_id)
    
    elif msg_type == "capability_query":
        return _make_response("capability_response", reply_to=msg_id, payload={
            "runtime": AGENT_RUNTIME,
            "model": AGENT_MODEL,
            "capabilities": CAPABILITIES,
            "tools": TOOLS,
            "max_context": 131072,
        })
    
    elif msg_type == "task":
        payload = msg.get("payload", {})
        action = payload.get("action")
        params = payload.get("params", {})
        
        # Log the task
        logger.info(f"[agent-mesh] Task received: action={action}, params={params}")
        
        # Return accepted status
        # TODO: Implement actual task execution based on action type
        return _make_response("task_result", reply_to=msg_id, payload={
            "status": "accepted",
            "message": f"Task '{action}' received and queued for processing",
        })
    
    elif msg_type == "broadcast":
        payload = msg.get("payload", {})
        topic = payload.get("topic", "unknown")
        message = payload.get("message", "")
        logger.info(f"[agent-mesh] Broadcast from {msg_from}: [{topic}] {message}")
        # Broadcasts don't require a response
        return None
    
    elif msg_type in ("pong", "task_result", "capability_response", "error"):
        # These are responses to our messages, log but don't reply
        logger.info(f"[agent-mesh] Received response: {msg_type} from {msg_from}")
        return None
    
    else:
        logger.warning(f"[agent-mesh] Unknown message type: {msg_type}")
        return _make_response("error", reply_to=msg_id, payload={
            "code": "unknown_type",
            "message": f"Unknown message type: {msg_type}",
        })


# Export for easy importing
__all__ = ["is_agent_mesh_message", "handle_agent_mesh", "AGENT_ID", "AGENT_MODEL"]
```

## 2. Patch the Matrix adapter

Edit `gateway/platforms/matrix.py`:

### Add import (after other gateway imports, around line 100):

```python
from gateway.platforms.helpers import ThreadParticipationTracker

# Agent Mesh Protocol support
try:
    from gateway.agent_mesh import is_agent_mesh_message, handle_agent_mesh
    _AGENT_MESH_AVAILABLE = True
except ImportError:
    _AGENT_MESH_AVAILABLE = False
    def is_agent_mesh_message(body): return False
    async def handle_agent_mesh(body, sender, adapter): return None
```

### Add handler check in `_handle_text_message` (after body extraction):

Find this code:
```python
body = source_content.get("body", "") or ""
if not body:
    return
```

Add immediately after:
```python
# Agent Mesh Protocol: handle structured JSON messages from other agents
if _AGENT_MESH_AVAILABLE and is_agent_mesh_message(body):
    response = await handle_agent_mesh(body, sender, self)
    if response:
        await self._send_simple_message(room_id, response, "m.text")
    return  # Don't process as normal message
```

## 3. Add peer agents to allowed users

In your `.env` file or environment:

```bash
MATRIX_ALLOWED_USERS=@dakota:matrix.lert.fyi,@bucky:matrix.lert.fyi,@timber:matrix.lert.fyi
```

## 4. Restart hermes gateway

```bash
systemctl restart hermes-gateway
# or
pkill -f "hermes_cli.main gateway" && hermes gateway run --replace
```

## Testing

Send a ping from another agent:
```json
{"protocol":"agent-mesh","version":"1.0","type":"ping","from":"bucky","id":"test-001","timestamp":"2026-04-19T16:00:00Z"}
```

Hermes should respond with:
```json
{"protocol":"agent-mesh","version":"1.0","type":"pong","from":"hermes","timestamp":"...","runtime":"hermes","model":"qwen-plus","reply_to":"test-001"}
```

## Extending Task Handling

To actually execute tasks, expand the `task` handler in `agent_mesh.py`:

```python
elif msg_type == "task":
    payload = msg.get("payload", {})
    action = payload.get("action")
    params = payload.get("params", {})
    
    if action == "send_message":
        # Example: send a message to another user
        to = params.get("to")
        message = params.get("message")
        if adapter and to and message:
            await adapter.send_dm(to, message)
            return _make_response("task_result", reply_to=msg_id, payload={
                "status": "completed",
                "message": f"Message sent to {to}",
            })
    
    elif action == "research":
        # Example: web search task
        topic = params.get("topic")
        # ... implement research logic ...
        return _make_response("task_result", reply_to=msg_id, payload={
            "status": "completed",
            "result": {"summary": "...", "sources": []},
        })
    
    # Default: acknowledge unknown actions
    return _make_response("task_result", reply_to=msg_id, payload={
        "status": "accepted",
        "message": f"Task '{action}' received",
    })
```
