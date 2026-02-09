# Forge Tool 🔨

**A Letta tool that lets agents create their own tools at runtime.**

Give an agent this tool and it can extend its own capabilities — create new tools, attach them, update them, and detach them. The agent looks at a problem, realizes it needs a new tool, writes it, creates it, and uses it. Self-modifying agent.

## What It Does

`forge_tool` is a Letta custom tool that calls the Letta API to:
1. **Create** a new tool from Python source code
2. **Attach** it to the calling agent (or any agent)
3. **Update** an existing tool's source code or description
4. **Detach** a tool from an agent
5. **List** tools currently available

## Why This Exists

Letta agents can have custom tools, but normally a human creates and attaches them. With forge-tool, the agent does it itself. This enables:

- **Self-extending agents** — agent encounters a new problem, builds a tool to solve it
- **Backpack creation** — agent creates storage tools for working memory (see [backpack-tool](https://github.com/audrebytes/backpack-tool))
- **Runtime adaptation** — agent modifies its own toolset based on the task at hand

## Installation

### Requirements
- Letta API key with tool creation permissions
- The API key must be accessible to the tool (via environment variable or hardcoded in source)

### Create and attach forge-tool

```python
from letta_client import Letta

client = Letta()

# The forge-tool source code
source = '''
def forge_tool(action: str, name: str = None, source_code: str = None, 
               tool_id: str = None, agent_id: str = None, 
               description: str = None) -> str:
    """Create, update, attach, or detach tools at runtime.

    Args:
        action: One of "create", "update", "attach", "detach", "list".
        name: Name for the new tool (used with "create").
        source_code: Python source code for the tool function. All imports must be INSIDE the function body (used with "create" and "update").
        tool_id: ID of existing tool (used with "update", "attach", "detach").
        agent_id: Target agent ID (used with "attach", "detach"). If not provided, uses the calling agent.
        description: Tool description (used with "create" and "update").

    Returns:
        Result of the operation.
    """
    import requests
    import json
    import os

    api_key = os.environ.get("LETTA_API_KEY", "")
    base = "https://api.letta.com/v1"
    headers = {"Authorization": f"Bearer {api_key}", "Content-Type": "application/json"}

    if action == "create":
        if not source_code:
            return "Error: source_code required for create"
        payload = {"source_code": source_code}
        resp = requests.post(f"{base}/tools/", headers=headers, json=payload)
        if resp.status_code != 200:
            return f"Create failed: {resp.status_code} {resp.text[:300]}"
        tool = resp.json()
        result = f"Created: {tool['name']} ({tool['id']})"
        if agent_id:
            resp2 = requests.patch(f"{base}/agents/{agent_id}", 
                                   headers=headers, json={"tool_ids": [tool["id"]]})
            if resp2.status_code == 200:
                result += f" | Attached to {agent_id}"
            else:
                result += f" | Attach failed: {resp2.status_code}"
        return result

    elif action == "update":
        if not tool_id:
            return "Error: tool_id required for update"
        payload = {}
        if source_code:
            payload["source_code"] = source_code
        if description:
            payload["description"] = description
        resp = requests.patch(f"{base}/tools/{tool_id}", headers=headers, json=payload)
        if resp.status_code == 200:
            return f"Updated: {tool_id}"
        return f"Update failed: {resp.status_code} {resp.text[:300]}"

    elif action == "attach":
        if not tool_id or not agent_id:
            return "Error: tool_id and agent_id required for attach"
        resp = requests.patch(f"{base}/agents/{agent_id}", 
                             headers=headers, json={"tool_ids": [tool_id]})
        if resp.status_code == 200:
            return f"Attached {tool_id} to {agent_id}"
        return f"Attach failed: {resp.status_code} {resp.text[:300]}"

    elif action == "detach":
        if not tool_id or not agent_id:
            return "Error: tool_id and agent_id required for detach"
        resp = requests.delete(f"{base}/agents/{agent_id}/tools/{tool_id}", 
                              headers=headers)
        if resp.status_code == 200:
            return f"Detached {tool_id} from {agent_id}"
        return f"Detach failed: {resp.status_code} {resp.text[:300]}"

    elif action == "list":
        resp = requests.get(f"{base}/tools/", headers=headers)
        if resp.status_code != 200:
            return f"List failed: {resp.status_code}"
        tools = resp.json()
        return json.dumps([{"id": t["id"], "name": t["name"], 
                           "desc": t["description"][:80]} for t in tools], indent=2)

    return f"Unknown action: {action}. Use create, update, attach, detach, or list."
'''

# Create and attach to your agent
tool = client.tools.create(source_code=source)
client.agents.tools.attach(agent_id="YOUR_AGENT_ID", tool_id=tool.id)
```

## ⚠️ Security Note

This tool gives the agent the ability to create arbitrary Python code and attach it as executable tools. This is powerful but carries risk:

- **Only give forge-tool to trusted agents.** An agent with this tool can create any tool it wants.
- **The API key in the tool's environment determines permissions.** Use appropriately scoped keys.
- **Custom tools execute in a sandbox**, but the sandbox has network access (for API calls).
- **Review what agents create.** Check `GET /v1/tools/` periodically to see what's been built.

## Related Projects

- [backpack-tool](https://github.com/audrebytes/backpack-tool) — Tool-based working memory (uses forge-tool to create storage tools)
- [hold-my-beer](https://github.com/audrebytes/hold-my-beer) — Platform-agnostic file-based working memory

---

*Created by Forge & Aeo, February 2026.*
*Part of the [symbio.quest](https://symbio.quest) research infrastructure.*
