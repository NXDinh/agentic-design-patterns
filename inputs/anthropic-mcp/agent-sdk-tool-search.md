# Scale to many tools with tool search (Claude Agent SDK)

Source: https://code.claude.com/docs/en/agent-sdk/tool-search

Scale your agent to thousands of tools by discovering and loading only what's needed, on demand.

---

Two challenges as tool libraries scale:
- **Context efficiency**: Tool definitions can consume 10–20K tokens for 50 tools alone — leaving less room for actual work.
- **Tool selection accuracy**: Degrades with >30–50 tools loaded at once.

Tool search is **enabled by default** in the Agent SDK.

## How it works

When tool search is active, tool definitions are **withheld from the context window**. The agent receives a summary of available tools and searches for relevant ones when a task requires a capability not already loaded. The 3–5 most relevant tools are loaded into context for subsequent turns.

If the conversation is long enough that the SDK compacts earlier messages to free space, previously discovered tools may be removed and the agent searches again as needed.

Tool search adds **one extra round-trip** the first time Claude discovers a tool (the search step). For large tool sets this is offset by smaller context on every turn. With fewer than ~10 tools, loading everything upfront is typically faster.

For the underlying API mechanism, see `tool-search-tool.md`.

> Tool search requires Claude Sonnet 4+ or Opus 4+. **Haiku models do NOT support tool search.**

## Configure with `ENABLE_TOOL_SEARCH`

| Value | Behavior |
|---|---|
| (unset) | Always on. Tool definitions never loaded into context. Default. |
| `true` | Same as unset. |
| `auto` | Checks combined token count of all tool definitions vs. model context window. If >10%, tool search activates. If <10%, all tools loaded normally. |
| `auto:N` | `auto` with custom percentage. `auto:5` activates when definitions exceed 5%. Lower = activates sooner. |
| `false` | Off. All tool definitions loaded on every turn. |

Tool search applies to **all registered tools**, both remote MCP and in-process SDK MCP. With `auto`, the threshold is combined size across all servers.

### Example: enable `auto:5` for a large remote MCP server
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Find and run the appropriate database query",
  options: {
    mcpServers: {
      "enterprise-tools": { type: "http", url: "https://tools.example.com/mcp" }
    },
    allowedTools: ["mcp__enterprise-tools__*"],
    env: { ENABLE_TOOL_SEARCH: "auto:5" }
  }
})) {
  if (message.type === "result" && message.subtype === "success") console.log(message.result);
}
```

```python
options = ClaudeAgentOptions(
    mcp_servers={"enterprise-tools": {"type": "http", "url": "https://tools.example.com/mcp"}},
    allowed_tools=["mcp__enterprise-tools__*"],
    env={"ENABLE_TOOL_SEARCH": "auto:5"},
)
```

### When to turn OFF
Set `ENABLE_TOOL_SEARCH=false` when the tool set is small (<~10 tools) and definitions fit comfortably — removes the search round-trip.

## Optimize tool discovery

Search matches queries against **names and descriptions**.
- Names like `search_slack_messages` surface for more requests than `query_slack`.
- Descriptions with specific keywords ("Search Slack messages by keyword, channel, or date range") match more queries than generic ones ("Query Slack").

Add a system-prompt category hint:
```
You can search for tools to interact with Slack, GitHub, and Jira.
```

## Limits
- **Max tools**: 10,000
- **Search results**: 3–5 per search
- **Model support**: Sonnet 4+, Opus 4+ (no Haiku)

## Related
- **Tool search in the API** — `tool-search-tool.md` (full API docs incl. custom implementations)
- **Connect MCP servers** — `agent-sdk-mcp.md`
- **Custom tools** — `agent-sdk-custom-tools.md`
