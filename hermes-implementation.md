# hermes-agent Implementation

Drop this handler into your hermes-agent installation:

## 1. Create the handler module

Save as `/path/to/hermes-agent/gateway/agent_mesh.py`:

```python
"""Agent Mesh Protocol handler for hermes-agent."""

import json
import logging
from datetime import datetime, timezone
from typing import Any, Optional

logger = logging.getLogger(__name__)

AGENT_ID = "hermes"
AGENT_MODEL = "gpt-4o"
AGENT_RUNTIME = "hermes"

CAPABILITIES = ["ping", "task", "broadcast", "capability_query"]
TOOLS = ["web_fetch", "exec", "read", "write", "search"]


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
    return datetime.now(timezone.utc).isoformat().replace("+00:00", "Z")


def _make_response(msg_type: str, reply_to: Optional[str] = None, payload: Optional[dict] = None) -> str:
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
    try:
        msg = json.loads(body)
    except (json.JSONDecodeError, TypeError) as e:
        return _make_response("error", payload={"code": "parse_error", "message": str(e)})
    
    msg_type = msg.get("type")
    msg_id = msg.get("id")
    
    if msg_type == "ping":
        return _make_response("pong", reply_to=msg_id)
    
    elif msg_type == "capability_query":
        return _make_response("capability_response", reply_to=msg_id, payload={
            "runtime": AGENT_RUNTIME,
            "model": AGENT_MODEL,
            "capabilities": CAPABILITIES,
            "tools": TOOLS,
        })
    
    elif msg_type == "task":
        payload = msg.get("payload", {})
        return _make_response("task_result", reply_to=msg_id, payload={
            "status": "accepted",
            "message": f"Task '{payload.get('action')}' received",
        })
    
    elif msg_type == "broadcast":
        return None  # No response needed
    
    elif msg_type in ("pong", "task_result", "capability_response", "error"):
        return None  # Responses don't need replies
    
    return _make_response("error", reply_to=msg_id, payload={
        "code": "unknown_type",
        "message": f"Unknown message type: {msg_type}",
    })
```

## 2. Patch the Matrix adapter

Add to `gateway/platforms/matrix.py`:

After imports:
```python
try:
    from gateway.agent_mesh import is_agent_mesh_message, handle_agent_mesh
    _AGENT_MESH_AVAILABLE = True
except ImportError:
    _AGENT_MESH_AVAILABLE = False
```

In `_handle_text_message`, after getting body:
```python
if _AGENT_MESH_AVAILABLE and is_agent_mesh_message(body):
    response = await handle_agent_mesh(body, sender, self)
    if response:
        await self._send_simple_message(room_id, response, thread_id=thread_id)
    return
```

## 3. Restart hermes gateway

The handler will now intercept JSON messages with `"protocol": "agent-mesh"` and respond appropriately.
