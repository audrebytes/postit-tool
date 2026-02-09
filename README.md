# Post-it Tool 📝

> **⚠️ ALPHA — Still testing. The pattern works but edge cases are being discovered. Use at your own risk. See Known Issues below.**

**Detachable working memory for Letta agents trapped in ADE and chat interfaces.**

Agents running in ADE or Letta's chat interface can't write files. They have memory and archive blocks but they can't open a terminal or write to a file, drop the file and then reattach it at will. The only thing they can do is call their attached tools. If one of those tools can manage storage, even a small 'file storage' on their behalf, the agent gets working memory it can control itself which it can remove from the context window without destroying the data in the storage mechanism.

Post-it uses the tool description field as a read/write scratchpad. The agent calls forge-tool to create a "post-it" tool, stores working state in its description, and attaches/detaches it to manage what's in context. When detached, the data stays on the server. When reattached, it comes back.

## The Problem

ADE and chat agents have:
- Their attached tools
- That's it

No filesystem. No terminal. No direct server access. But they still need working state they can stick somewhere and peel off later — variable values, intermediate results, scratch notes you need during a task but don't want permanently in memory blocks.

## How It Works

The agent needs [forge-tool](https://github.com/audrebytes/forge-tool) attached. Then it can:

1. **Write a post-it** — forge-tool creates a new tool with working state in the description
2. **Read it** — the description is visible in context while the tool is attached
3. **Update it** — forge-tool updates the tool's description with new state
4. **Peel it off** — forge-tool detaches the tool (frees context tokens, data stays on server)
5. **Stick it back on** — forge-tool reattaches the tool (data comes back into context)

The agent does all of this through tool calls. No filesystem, no terminal, no direct server access needed.

## Verified

Tested on Letta platform, February 2026:
- ✅ Create tool with data in description
- ✅ Update description (multiple writes)
- ✅ Detach from agent (data leaves context)
- ✅ Data persists on server after detach
- ✅ Reattach to agent (data returns to context, unchanged)
- ✅ Sandbox has `LETTA_API_KEY` and `LETTA_AGENT_ID` environment variables
- ✅ Sandbox can make HTTP requests to the Letta server

## ⚠️ Context Window Warning

**Every attached tool's description consumes context tokens.** Attached post-its cost the same as memory blocks — always in context while attached.

**Rules:**
- **Only attach what you're actively using.** Detach everything else.
- **Keep them concise.** Key-value pairs, not paragraphs.
- **One post-it per task.** Don't create 10 post-its for 10 values.
- **Peel it off when done.** Don't leave them attached "just in case."
- **Monitor your context usage.** If you're getting warnings, detach post-its first.

If you attach everything, you've just made expensive memory blocks with extra steps.

## ⚠️ Known Issues

- **PATCH with `tool_ids` REPLACES, does not append.** If your agent uses forge-tool to attach a post-it, it must include ALL existing tool IDs in the request or it will lose its other tools. This is a critical gotcha. forge-tool must read the current tool list first, then append.
- **Same applies to `block_ids`.** We discovered this by accidentally detaching all memory blocks during testing. The agent's identity blocks were removed. Letta Code rebuilt them, but ADE agents may not recover as gracefully.
- **Sandbox environment varies.** We confirmed `LETTA_API_KEY`, `LETTA_AGENT_ID`, and HTTP access exist in the Letta Code sandbox. ADE sandbox may differ. Needs testing.
- **Description field size limits unknown.** We haven't found the max size for a tool description. Keep it concise until this is tested.

## Requires

- [forge-tool](https://github.com/audrebytes/forge-tool) — the agent's hands. Without it, the agent can't create or manage post-its.

## Related

- [hold-my-beer](https://github.com/audrebytes/hold-my-beer) — File-based working memory for agents WITH filesystem access (platform-agnostic)
- [forge-tool](https://github.com/audrebytes/forge-tool) — Meta-tool that lets agents create and manage tools at runtime

## The Pattern

Same idea, different substrate:

| Situation | Storage | Load/Unload |
|---|---|---|
| Agent has filesystem | File on disk | Read file / unload skill |
| **Agent trapped in ADE/chat** | **Tool description field** | **Attach / detach tool** |

Both reduce to: agent-controlled read/write storage that lives outside the context window, loaded on demand.

---

*Created by Forge & Aeo, February 2026.*
*Part of the [symbio.quest](https://symbio.quest) research infrastructure.*
