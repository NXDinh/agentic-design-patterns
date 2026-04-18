# Tool search tool

Source: https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool

Enables Claude to work with hundreds or thousands of tools by dynamically discovering and loading them on-demand. Searches your tool catalog (names, descriptions, argument names, argument descriptions) and loads only the tools Claude needs.

---

## Why it matters

- **Context bloat**: Tool definitions eat context fast. A typical multi-server setup (GitHub, Slack, Sentry, Grafana, Splunk) can consume ~55k tokens in definitions before any work. Tool search typically cuts this **>85%**, loading only 3–5 tools per request.
- **Tool selection accuracy**: Claude's ability to pick the right tool degrades once you exceed 30–50 available tools. Tool search surfaces a focused set on demand, keeping accuracy high even at thousands of tools.

On-demand loading is an instance of the broader just-in-time retrieval principle (see Anthropic's "Effective context engineering" engineering post).

You can also implement client-side tool search by returning `tool_reference` blocks from your own search.

> **ZDR eligible** for organizations with a ZDR arrangement.

> **Amazon Bedrock**: server-side tool search is available only via the invoke API, not the converse API.

## How it works

Two variants:
- **Regex** (`tool_search_tool_regex_20251119`): Claude constructs Python-regex patterns.
- **BM25** (`tool_search_tool_bm25_20251119`): Claude uses natural-language queries.

Flow:
1. Include the tool search tool in `tools`.
2. Mark tool definitions with `defer_loading: true` for tools that shouldn't load immediately.
3. Claude initially sees only the tool search tool + any non-deferred tools.
4. When Claude needs more tools, it searches.
5. The API returns 3–5 most relevant `tool_reference` blocks.
6. References auto-expand into full tool definitions.
7. Claude selects and invokes.

## Quick start

```python
import anthropic
client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=2048,
    messages=[{"role": "user", "content": "What is the weather in San Francisco?"}],
    tools=[
        {"type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex"},
        {
            "name": "get_weather",
            "description": "Get the weather at a specific location",
            "input_schema": {
                "type": "object",
                "properties": {
                    "location": {"type": "string"},
                    "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
                },
                "required": ["location"],
            },
            "defer_loading": True,
        },
        {
            "name": "search_files",
            "description": "Search through files in the workspace",
            "input_schema": {
                "type": "object",
                "properties": {
                    "query": {"type": "string"},
                    "file_types": {"type": "array", "items": {"type": "string"}},
                },
                "required": ["query"],
            },
            "defer_loading": True,
        },
    ],
)
```

## Tool definition

Two variants — pick one:
```json
{ "type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex" }
```
```json
{ "type": "tool_search_tool_bm25_20251119", "name": "tool_search_tool_bm25" }
```

### Regex variant query format

Python `re.search()` syntax — NOT natural language. Common patterns:
- `"weather"` — contains "weather"
- `"get_.*_data"` — matches `get_user_data`, `get_weather_data`
- `"database.*query|query.*database"` — OR patterns
- `"(?i)slack"` — case-insensitive

Max query length: 200 chars.

### BM25 variant query format
Natural language.

### Undated aliases
`tool_search_tool_regex` and `tool_search_tool_bm25` resolve to the latest dated version.

## Deferred tool loading

```json
{
  "name": "get_weather",
  "description": "Get current weather for a location",
  "input_schema": { ... },
  "defer_loading": true
}
```

- Tools without `defer_loading` load into context immediately.
- `defer_loading: true` → loaded only when Claude discovers via search.
- The tool search tool itself should **never** have `defer_loading: true`.
- Keep your 3–5 most frequently used tools **non-deferred** for optimal performance.

**How deferral works internally**: Deferred tools are NOT included in the system-prompt prefix. When discovered, the tool's full definition is appended inline as a `tool_reference` block in the conversation body. The prefix is untouched — **prompt caching is preserved**. The strict-mode grammar builds from the full toolset, so `defer_loading` + strict compose without grammar recompilation.

Both variants search tool names, descriptions, argument names, and argument descriptions.

## Response format

```json
{
  "role": "assistant",
  "content": [
    { "type": "text", "text": "I'll search for tools to help with the weather information." },
    { "type": "server_tool_use", "id": "srvtoolu_01ABC123", "name": "tool_search_tool_regex", "input": { "query": "weather" } },
    {
      "type": "tool_search_tool_result",
      "tool_use_id": "srvtoolu_01ABC123",
      "content": {
        "type": "tool_search_tool_search_result",
        "tool_references": [{ "type": "tool_reference", "tool_name": "get_weather" }]
      }
    },
    { "type": "text", "text": "I found a weather tool. Let me get the weather for San Francisco." },
    { "type": "tool_use", "id": "toolu_01XYZ789", "name": "get_weather", "input": { "location": "San Francisco", "unit": "fahrenheit" } }
  ],
  "stop_reason": "tool_use"
}
```

- **`server_tool_use`** — invoking the tool search tool
- **`tool_search_tool_result`** — search results with nested `tool_search_tool_search_result`
- **`tool_references`** — discovered tools
- **`tool_use`** — invoking the discovered tool

`tool_reference` blocks auto-expand into full definitions before being shown to Claude — no handling needed as long as every referenced tool has a matching definition in `tools`.

## MCP integration

`mcp_toolset` supports `defer_loading` (applied through `default_config` or per-tool `configs`). See `mcp-connector.md`.

## Custom tool search implementation

Implement your own search (e.g., embeddings/semantic) by returning `tool_reference` blocks from a custom tool:
```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_your_tool_id",
  "content": [{ "type": "tool_reference", "tool_name": "discovered_tool_name" }]
}
```
Every referenced tool must have a definition in the top-level `tools` array with `defer_loading: true`.

> The `tool_search_tool_result` format is for server-side tool search. For client-side, use the standard `tool_result` format with `tool_reference` content blocks.

See the [tool search with embeddings cookbook](https://platform.claude.com/cookbooks/tool_use).

## Error handling

> Tool search is NOT compatible with `input_examples` on tool definitions. If you need examples, use standard tool calling without tool search.

### HTTP 400 errors

**All tools deferred**:
```json
{"type":"error","error":{"type":"invalid_request_error","message":"All tools have defer_loading set. At least one tool must be non-deferred."}}
```

**Missing tool definition**:
```json
{"type":"error","error":{"type":"invalid_request_error","message":"Tool reference 'unknown_tool' has no corresponding tool definition"}}
```

### Tool result errors (HTTP 200)
```json
{
  "type": "tool_result",
  "tool_use_id": "srvtoolu_01ABC123",
  "content": { "type": "tool_search_tool_result_error", "error_code": "invalid_pattern" }
}
```
Codes: `too_many_requests`, `invalid_pattern`, `pattern_too_long`, `unavailable`.

### Common mistakes

**All tools deferred** — remove `defer_loading` from the tool search tool itself.

**Missing tool definition** — ensure every discoverable tool has a full definition in `tools`.

**Claude doesn't find expected tools**:
- Check name AND description — search hits both.
- Test your pattern with `re.search`.
- Default is case-sensitive; use `(?i)` for case-insensitive.
- Claude uses broad patterns (`".*weather.*"`) not exact matches.
- Add common keywords to descriptions for discoverability.

## Prompt caching

`defer_loading` preserves prompt caching (deferred tools stripped before cache-key computation). The system auto-expands `tool_reference` blocks throughout conversation history, so Claude can reuse discovered tools across turns without re-searching.

## Streaming

```
event: content_block_start
data: {"type":"content_block_start","index":1,"content_block":{"type":"server_tool_use","id":"srvtoolu_xyz789","name":"tool_search_tool_regex"}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"{\"query\":\"weather\"}"}}

event: content_block_start
data: {"type":"content_block_start","index":2,"content_block":{"type":"tool_search_tool_result", ...}}
```

## Batch requests
Works in the Messages Batches API. Same pricing as regular API.

## Limits and best practices

### Limits
- **Max tools**: 10,000
- **Search results**: 3–5 per search
- **Pattern length**: 200 chars
- **Models**: Claude Mythos Preview, Sonnet 4.0+, Opus 4.0+, Haiku 4.5+

### When to use
**Good:**
- 10+ tools available
- Tool definitions >10k tokens
- Tool selection accuracy issues with large sets
- MCP-powered systems with multiple servers (200+ tools)
- Growing tool library

**When NOT to use:**
- Fewer than 10 tools
- All tools used every request
- Tiny definitions (<100 tokens total)

### Optimization tips
- Keep 3–5 most-used tools non-deferred.
- Clear, descriptive names and descriptions.
- **Consistent namespacing**: `github_`, `slack_` prefixes so regex surfaces the right group.
- Semantic keywords matching how users describe tasks.
- System-prompt category hint: "You can search for tools to interact with Slack, GitHub, and Jira."
- Monitor which tools Claude discovers; refine descriptions.

## Usage tracking

```json
{
  "usage": {
    "input_tokens": 1024,
    "output_tokens": 256,
    "server_tool_use": { "tool_search_requests": 2 }
  }
}
```
