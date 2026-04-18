# Tool use with prompt caching

Source: https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-use-with-prompt-caching

Cache tool definitions across turns and understand what invalidates your cache.

---

## `cache_control` on tool definitions

Place `cache_control: {"type": "ephemeral"}` on the **last tool in your `tools` array**. This caches the entire tool-definitions prefix, from the first tool through the marked breakpoint:

```json
{
  "tools": [
    {
      "name": "get_weather",
      "description": "Get the current weather in a given location",
      "input_schema": { "type": "object", "properties": { "location": {"type": "string"} }, "required": ["location"] }
    },
    {
      "name": "get_time",
      "description": "Get the current time in a given time zone",
      "input_schema": { "type": "object", "properties": { "timezone": {"type": "string"} }, "required": ["timezone"] },
      "cache_control": { "type": "ephemeral" }
    }
  ]
}
```

For `mcp_toolset`, the `cache_control` breakpoint lands on the **last tool in the set**. You don't control tool order within an MCP toolset, so place the breakpoint on the `mcp_toolset` entry itself — the API applies it to the final expanded tool.

## `defer_loading` and cache preservation

Deferred tools are NOT included in the system-prompt prefix. When the model discovers a deferred tool through tool search, the definition is appended inline as a `tool_reference` block in the conversation history. **The prefix is untouched, so prompt caching is preserved.**

Adding tools dynamically through tool search does not break your cache. Start a conversation with a small set of always-loaded tools (cached), let the model discover additional tools as needed, and keep the same cache hit across every turn.

`defer_loading` also acts independently of grammar construction for strict mode. The grammar builds from the full toolset regardless of which tools are deferred, so prompt caching and grammar caching are both preserved when tools load dynamically.

## What invalidates your cache

The cache follows a prefix hierarchy (`tools` → `system` → `messages`). A change at one level invalidates that level and everything after it:

| Change | Invalidates |
|---|---|
| Modifying tool definitions | Entire cache (tools, system, messages) |
| Toggling web search or citations | System + messages caches |
| Changing `tool_choice` | Messages cache |
| Changing `disable_parallel_tool_use` | Messages cache |
| Toggling images present/absent | Messages cache |
| Changing thinking parameters | Messages cache |

> If you need to vary `tool_choice` mid-conversation, consider placing cache breakpoints before the variation point.

## Per-tool interaction table

| Tool | Caching considerations |
|---|---|
| Web search | Enabling/disabling invalidates system + messages caches |
| Web fetch | Enabling/disabling invalidates system + messages caches |
| Code execution | Container state is independent of prompt cache |
| Tool search | Discovered tools load as `tool_reference` blocks → prefix cache preserved |
| Computer use | Screenshot presence affects messages cache |
| Text editor | Standard client tool, no special caching interaction |
| Bash | Standard client tool, no special caching interaction |
| Memory | Standard client tool, no special caching interaction |

## Related
- **Tool search** — `tool-search-tool.md` — load tools on demand without breaking cache
- **Tool reference** — `tool-reference.md` — all options (`cache_control`, `strict`, `defer_loading`, `allowed_callers`)
