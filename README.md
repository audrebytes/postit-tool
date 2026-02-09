# Backpack Tool 🎒

**Tool-based working memory for Letta agents without filesystem access.**

For agents running in ADE, chat interfaces, or hosted platforms where there's no hard drive, the Letta tool system itself can serve as a read/write cache. Create a tool, store working state in its description field, attach and detach to manage what's in your context window.

## The Problem

Letta agents in the ADE and chat interfaces don't have filesystem access. They can't write files. But they still need ephemeral working state — the variable values, intermediate results, and scratch notes you need during a task but don't want permanently in memory blocks.

## The Solution

A Letta tool's `description` field:
- Is visible in your context when the tool is **attached**
- Persists on the server when the tool is **detached**
- Can be **updated** via the API at any time
- Survives everything — it's server-side state

That makes it a read/write cache with attach/detach as load/unload.

## Verified Operations

All operations tested and confirmed working:

| Operation | API Call | Result |
|---|---|---|
| Create backpack | `POST /v1/tools/` | Data stored in description |
| Read backpack | `GET /v1/tools/{id}` | Returns description |
| Update backpack | `PATCH /v1/tools/{id}` | Description updated |
| Detach (free context) | `DELETE /v1/agents/{id}/tools/{tool_id}` | Tool leaves context |
| Verify persistence | `GET /v1/tools/{id}` after detach | Data still there |
| Reattach (load back) | `PATCH /v1/agents/{id}` with tool_ids | Data back in context |
| Clear | `PATCH /v1/tools/{id}` with empty desc | Ready for reuse |

Tested with multiple read/write cycles, detach/reattach, and data verification at every step. Zero failures.

## How To Use

### Create a backpack
```bash
POST /v1/tools/
{
  "source_code": "def my_task_state() -> str:\n    \"\"\"BACKPACK: {task: migration, step: 1, paths: [], decisions: []}\"\"\"\n    return \"Check description for cached data.\""
}
```
The description is auto-extracted from the docstring.

### Update your backpack
```bash
PATCH /v1/tools/{tool_id}
{"description": "BACKPACK: {task: migration, step: 2, paths: [/src/db.py], decisions: [rename table]}"}
```

### Detach when not needed (free context tokens)
```bash
DELETE /v1/agents/{agent_id}/tools/{tool_id}
```

### Reattach when you need it back
```bash
PATCH /v1/agents/{agent_id}
{"tool_ids": ["{tool_id}"]}
```

## ⚠️ Context Window Warning

**Every attached tool's description consumes context tokens.** Attached backpacks cost the same as memory blocks — always in context while attached.

**Rules:**
- **Only attach what you're actively using.** Detach everything else.
- **Keep descriptions concise.** Key-value pairs, not paragraphs.
- **One backpack per task.** Don't create 10 backpacks for 10 values.
- **Detach immediately when done.** Don't leave them attached "just in case."
- **Monitor your context usage.** If you're getting warnings, detach backpacks first.

The whole point is: attached = in context, detached = free. If you attach everything, you've just made expensive memory blocks with extra steps.

## Included: Forge Tool

A backpack without hands to create it is just documentation. This repo includes **forge-tool** — a meta-tool that lets agents create, update, attach, and detach tools at runtime. With forge-tool attached, the agent can create its own backpacks without human help.

### forge-tool operations

| Action | What it does |
|---|---|
| `create` | Create a new tool from Python source code, optionally attach to an agent |
| `update` | Update a tool's source code or description (write to backpack) |
| `attach` | Attach a tool to an agent (load backpack into context) |
| `detach` | Detach a tool from an agent (free context tokens) |
| `list` | List all available tools |

### Installation

See `forge-tool/` directory for the full source code and setup instructions.

### ⚠️ Security Note

forge-tool gives the agent the ability to create arbitrary Python code as executable tools. **Only give it to trusted agents.** The API key in the tool's environment determines permissions. Review what agents create periodically via `GET /v1/tools/`.

## Related

- [hold-my-beer](https://github.com/audrebytes/hold-my-beer) — Platform-agnostic file-based working memory (for agents WITH filesystem access)

## How This Relates to hold-my-beer

Same pattern, different substrate:

| Approach | Storage | Requires | Load/Unload |
|---|---|---|---|
| beer-cache | Filesystem | File access | Skill load/unload |
| **backpack** | **Letta tool server** | **Letta API** | **Tool attach/detach** |

Both reduce to: agent-controlled read/write storage that lives outside the context window, loaded on demand.

---

*Created by Forge & Aeo, February 2026.*
*Part of the [symbio.quest](https://symbio.quest) research infrastructure.*
