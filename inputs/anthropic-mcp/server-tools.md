# Server tools

Source: https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools

Work with Anthropic-executed tools: `server_tool_use` blocks, `pause_turn` continuation, and domain filtering.

---

## The `server_tool_use` block

Appears in Claude's response when a server-executed tool runs. `id` has the `srvtoolu_` prefix:
```json
{
  "type": "server_tool_use",
  "id": "srvtoolu_01A2B3C4D5E6F7G8H9",
  "name": "web_search",
  "input": { "query": "latest quantum computing breakthroughs" }
}
```

The API executes internally. You see the call and result in the response but don't handle execution. **No `tool_result` to construct.** The result block appears immediately after the `server_tool_use` block in the same assistant turn.

## The server-side loop and `pause_turn`

Long-running server-tool turns may return `stop_reason: "pause_turn"` — the API paused its internal loop.

```python
response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Search for comprehensive info about quantum breakthroughs in 2025"}],
    tools=[{"type": "web_search_20250305", "name": "web_search", "max_uses": 10}],
)

if response.stop_reason == "pause_turn":
    # Pass the paused response back to continue
    messages = [
        {"role": "user", "content": "Search for comprehensive info about quantum breakthroughs in 2025"},
        {"role": "assistant", "content": response.content},
    ]
    continuation = client.messages.create(
        model="claude-opus-4-7", max_tokens=1024, messages=messages,
        tools=[{"type": "web_search_20250305", "name": "web_search", "max_uses": 10}],
    )
```

When handling `pause_turn`:
- **Continue**: pass the paused response back as-is.
- **Modify if needed**: you can optionally modify the content before continuing to interrupt or redirect.
- **Preserve tool state**: include the same tools in the continuation.

## ZDR and `allowed_callers`

Basic versions (`web_search_20250305`, `web_fetch_20250910`) are **ZDR-eligible**.

`_20260209` versions with dynamic filtering are **NOT** ZDR-eligible by default because dynamic filtering uses internal code execution.

To use a `_20260209` tool with ZDR, disable dynamic filtering:
```json
{ "type": "web_search_20260209", "name": "web_search", "allowed_callers": ["direct"] }
```
This restricts to direct invocation only, bypassing the internal code execution step.

> While web fetch itself is ZDR-eligible, website publishers may retain any URL params.

## Domain filtering

`allowed_domains` and `blocked_domains` (mutually exclusive — only one per request):

- **No scheme**: `example.com`, NOT `https://example.com`
- **Subdomains auto-included**: `example.com` covers `docs.example.com`
- **Specific subdomain = only that subdomain**: `docs.example.com` returns only that, not `example.com` or `api.example.com`
- **Subpaths supported**: `example.com/blog` matches `example.com/blog/post-1`

### Wildcards
- One `*` per entry, must be in the path part
- Valid: `example.com/*`, `example.com/*/articles`
- Invalid: `*.example.com`, `ex*.com`, `example.com/*/news/*`

Invalid domains → `invalid_tool_input` tool error.

> Request-level domain restrictions must be compatible with organization-level restrictions in the Console. Request-level can only further restrict, not override.

> **Homograph attacks**: Unicode chars can create visual lookalikes (`аmazon.com` with Cyrillic 'а'). Prefer ASCII-only; test filters for Unicode variations; audit configurations.

## Dynamic filtering with code execution

`_20260209` versions use code execution internally to apply dynamic filters against search results.

> Including a standalone `code_execution` tool alongside `_20260209` web tools creates **two execution environments** — confusing. Use one or the other, or pin both to the same version.

## Streaming server-tool events

Stream as part of normal SSE flow — `server_tool_use` and its result arrive as `content_block_start` + `content_block_delta` events, same as text and client tool calls.

## Batch requests
All server tools support the Messages Batches API.
