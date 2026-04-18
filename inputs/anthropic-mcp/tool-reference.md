# Tool reference

Source: https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference

Directory of Anthropic-provided tools and reference for optional properties you can set on any tool definition.

---

## Anthropic-provided tools

Two kinds: **server tools** (execute on Anthropic infrastructure) and **client tools** (Anthropic defines the schema but your app executes). Both appear in `tools` alongside your user-defined tools.

| Tool | `type` | Execution | Status |
|---|---|---|---|
| Web search | `web_search_20260209` / `web_search_20250305` | Server | GA |
| Web fetch | `web_fetch_20260209` / `web_fetch_20250910` | Server | GA |
| Code execution | `code_execution_20260120` / `code_execution_20250825` | Server | GA |
| Advisor | `advisor_20260301` | Server | Beta: `advisor-tool-2026-03-01` |
| Tool search | `tool_search_tool_regex_20251119` / `tool_search_tool_bm25_20251119` | Server | GA |
| MCP connector | `mcp_toolset` | Server | Beta: `mcp-client-2025-11-20` |
| Memory | `memory_20250818` | Client | GA |
| Bash | `bash_20250124` | Client | GA |
| Text editor | `text_editor_20250728` / `text_editor_20250124` | Client | GA |
| Computer use | `computer_20251124` / `computer_20250124` | Client | Beta |

Tool search `type` also accepts undated aliases: `tool_search_tool_regex`, `tool_search_tool_bm25` (resolve to latest).

## Tool versioning

Most Anthropic tools carry `_YYYYMMDD` suffixes. A new version is released when behavior, schema, or model support changes. Older versions remain available.

Relationship types:
- **Capability-keyed** — `web_search_20260209` / `web_fetch_20260209` add dynamic content filtering; `code_execution_20260120` adds programmatic tool calling from within the sandbox. Both new/old current; pick based on needed capability.
- **Model-keyed** — `text_editor_20250728` is for Claude 4 models; `text_editor_20250124` is for earlier. Choose by model.
- **Variants** — `tool_search_tool_regex_20251119` and `tool_search_tool_bm25_20251119` are two algorithms released together; neither supersedes.
- **Legacy** — `code_execution_20250522` (Python only) vs `code_execution_20250825` (adds Bash + file ops).

`mcp_toolset` is not date-versioned — versioning is carried in the `anthropic-beta` header.

## Tool definition properties

Every tool in `tools` (including user-defined) accepts optional properties. These **compose** — you can set `defer_loading` + `cache_control` + `strict` on the same tool.

| Property | Purpose | Available on |
|---|---|---|
| `cache_control` | Set a prompt-cache breakpoint at this tool definition | All tools |
| `strict` | Guarantee schema validation on names and inputs | All tools except `mcp_toolset` |
| `defer_loading` | Exclude from initial system prompt; load on-demand via tool search | All tools (for `mcp_toolset`, use `default_config`) |
| `allowed_callers` | Restrict which callers can call the tool | All tools except `mcp_toolset` |
| `input_examples` | Example input objects to help Claude | User-defined + Anthropic-schema client tools (not server tools) |
| `eager_input_streaming` | Fine-grained input streaming (`true`) vs buffered (`false`) | User-defined tools only |

### `allowed_callers` values

Array accepting:
| Value | Meaning |
|---|---|
| `"direct"` | Model can call directly in a `tool_use` block (default if omitted) |
| `"code_execution_20260120"` | Code running inside a code_execution sandbox can call this tool |

Omitting `"direct"` (e.g. `["code_execution_20260120"]`) means the tool is callable only from within code execution. The response's `tool_use` block includes a `caller` field.

### `defer_loading` and prompt caching

Tools with `defer_loading: true` are **stripped from the rendered tools section before the cache key is computed**. They don't appear in the system-prompt prefix. When tool search discovers a deferred tool, the full definition is expanded inline at that point in the conversation body, **not** in the prefix.

→ **`defer_loading: true` preserves your prompt cache**. You can add deferred tools without invalidating an existing cache entry. The cache remains valid across the turn where the tool is discovered and the turn where it's called.
