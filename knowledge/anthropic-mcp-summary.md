# Anthropic MCP — Summary

Source: `inputs/anthropic-mcp/` (captured from `platform.claude.com` and `code.claude.com`).

**Scope**: Anthropic-specific MCP / tool-use surfaces. The upstream MCP spec itself lives in `inputs/mcp-official/` (see `knowledge/mcp-official-summary.md`) — Anthropic redirects readers to `modelcontextprotocol.io` for the protocol and only documents their own products on top of it.

---

## Six Anthropic-specific surfaces

### 1. MCP Connector (Messages API)
Call remote MCP servers from the Messages API without running your own MCP client.
- **Beta header**: `anthropic-beta: mcp-client-2025-11-20` (old `mcp-client-2025-04-04` deprecated).
- **Components**:
  - `mcp_servers[]` — `{ type: "url", url, name, authorization_token }`
  - `tools[]` with `{ type: "mcp_toolset", mcp_server_name, default_config, configs }` — allowlist / denylist / defer per tool
- **Limitations**: tool calls only (no resources/prompts), HTTPS remote only (Streamable HTTP or SSE), no Bedrock/Vertex, not ZDR-eligible.
- **Response blocks**: `mcp_tool_use` + `mcp_tool_result`.
- File: `inputs/anthropic-mcp/mcp-connector.md`.

### 2. Tool Runner (SDK)
Python / TypeScript / Ruby SDK helper that drives the agentic loop automatically.
- Python: `@beta_tool` decorator infers JSON Schema from hints + docstring.
- TypeScript: `betaZodTool` (Zod-validated) or `betaTool` (JSON Schema).
- Ruby: `Anthropic::BaseTool` + `Anthropic::BaseModel`.
- Supports **automatic compaction** — summaries when tokens exceed threshold → long agentic sessions past context limits.
- Iterate `for message in runner:` or get final with `.until_done()` / `await runner`.
- **Modify results** (e.g., `cache_control: {type: "ephemeral"}`) before the runner sends them back, via `generate_tool_call_response()` + `append_messages()`.
- `ANTHROPIC_LOG=info|debug` for tool-failure stack traces.
- File: `inputs/anthropic-mcp/tool-runner.md`.

### 3. Tool Search Tool + `defer_loading`
Anthropic's answer to **tool-library scale** — the thing that makes MCP viable when you've got dozens of servers.
- Cuts context cost >85%. Loads only 3–5 tools per turn.
- Two variants: `tool_search_tool_regex_20251119` (Python `re.search`) and `tool_search_tool_bm25_20251119` (natural language).
- Mark tools (or MCP toolsets via `default_config`) `defer_loading: true`.
- **Prompt cache preserved**: deferred tools stripped from prefix before cache-key computation; discovered tools expanded inline in the conversation body.
- Limits: max 10,000 tools; 3–5 returned per search; 200-char pattern max; Sonnet 4+ / Opus 4+ / Haiku 4.5+.
- **Not supported with** `input_examples`.
- Namespace tools consistently (`github_`, `slack_`); 3–5 most-used non-deferred.
- File: `inputs/anthropic-mcp/tool-search-tool.md`.

### 4. Claude Code's MCP client
Claude Code is itself an MCP host.
- Config via `.mcp.json` (project), user settings, or `claude mcp add …`.
- **Governance**: `managed-mcp.json` for enterprise-controlled lists; `allowedMcpServers` / `deniedMcpServers` filters. Denylist beats allowlist.
- File: `inputs/anthropic-mcp/claude-code-mcp.md`.

### 5. Claude Agent SDK MCP
Build production agents with `query()` + MCP servers.
- **`mcpServers`** in options: `type: "stdio" | "http" | "sse"` or an in-process SDK server.
- **`allowedTools`** required — MCP tools need explicit permission (e.g. `mcp__github__*`).
- Naming: `mcp__{server_name}__{tool_name}`.
- **SDK MCP servers (in-process)** via `create_sdk_mcp_server` (Python) / `createSdkMcpServer` (TypeScript) + `@tool` / `tool()` — custom tools without a separate process.
- **`ENABLE_TOOL_SEARCH` envvar** — `true` (default) / `auto` / `auto:N` / `false`.
- System message at session start (`subtype: "init"`) reports per-server connection status + available tools.
- Files: `inputs/anthropic-mcp/agent-sdk-mcp.md`, `inputs/anthropic-mcp/agent-sdk-custom-tools.md`, `inputs/anthropic-mcp/agent-sdk-tool-search.md`.

### 6. Strict tool use
Grammar-constrained sampling to guarantee Claude's tool inputs exactly match the JSON Schema.
- Set `strict: true` on the tool definition.
- Required: `additionalProperties: false` in the schema.
- Composes with `defer_loading` and `cache_control`.
- Pair with `tool_choice: {"type": "any"}` to guarantee BOTH some tool runs AND inputs are valid.
- **HIPAA gotcha**: schemas cached 24h independently of message content → **no PHI in schema** (property names, enum/const values, pattern regexes).
- File: `inputs/anthropic-mcp/strict-tool-use.md`.

---

## Anthropic's tool-use mental model

From `how-tool-use-works.md`:
> Tool use is a contract between your application and the model. You specify what operations are available; Claude decides when to call them.

**Three execution buckets:**
| Bucket | Examples | Who executes |
|---|---|---|
| User-defined client tools | Your domain APIs | You |
| Anthropic-schema client tools | `bash`, `text_editor`, `computer`, `memory` | You — but the schema is **trained-in**, so Claude calls them more reliably |
| Server-executed tools | `web_search`, `web_fetch`, `code_execution`, `tool_search` | Anthropic |

**The agentic loop (client tools)**: `while stop_reason == "tool_use": execute, return tool_result, continue`. Exits on `end_turn`, `max_tokens`, `stop_sequence`, or `refusal`.

**Server-side loop**: happens inside Anthropic. `stop_reason: "pause_turn"` means "re-send the conversation to continue" — see `server-tools.md`.

**The tell that you should be using tools**: *if you're writing a regex to extract a decision from model output, that decision should have been a tool call.*

---

## Message-format rules (critical)

From `handle-tool-calls.md`:
1. **`tool_result` must immediately follow its matching `tool_use`** — no messages in between.
2. In a `user` message with tool results, **tool_result blocks come FIRST** in `content`; any text comes AFTER. Violation → 400.
3. **Parallel tool use requires all tool_result blocks in a single user message.** Splitting into multiple user messages teaches Claude to stop parallelizing.

---

## Tool definition best practices (`define-tools.md`)

1. **Descriptions are the #1 performance factor**. 3–4+ sentences. What/when/params/caveats.
2. **Consolidate related ops**. One tool with an `action` parameter beats three.
3. **Namespace by service**: `github_list_prs`, `slack_send_message`. Critical for tool search.
4. **High-signal responses**. Stable IDs; only fields Claude needs.
5. **`input_examples`** for complex/nested tools (not supported for server tools; not supported with tool search).

---

## Parallel tool use (`parallel-tool-use.md`)

Default-on for Claude 4 models. Disable with `disable_parallel_tool_use=true`.

**Strong system-prompt trigger:**
```
<use_parallel_tool_calls>
For maximum efficiency, whenever you perform multiple independent operations, invoke all relevant tools simultaneously rather than sequentially. Prioritize calling tools in parallel whenever possible. For example, when reading 3 files, run 3 tool calls in parallel to read all 3 files into context at the same time. When running multiple read-only commands like `ls` or `list_dir`, always run all of the commands in parallel. Err on the side of maximizing parallel tool calls rather than running too many tools sequentially.
</use_parallel_tool_calls>
```

---

## Prompt caching + tools (`tool-use-with-prompt-caching.md`)

- Put `cache_control: {"type": "ephemeral"}` on the **last tool** in your `tools` array to cache the entire tool-definitions prefix.
- For `mcp_toolset`, place `cache_control` on the `mcp_toolset` entry itself (API applies it to the final expanded tool).
- `defer_loading: true` **preserves the cache** — deferred tools aren't in the prefix; discovered tools are added to the conversation body.
- **Invalidation hierarchy** (`tools` → `system` → `messages`):
  - Modifying tool definitions → invalidates everything
  - Toggling web search/citations → system + messages
  - Changing `tool_choice` / `disable_parallel_tool_use` / thinking params → messages

---

## Server tools (`server-tools.md`)

- Result comes back as `server_tool_use` + inline result blocks in the assistant turn — **no `tool_result` to construct**.
- **`pause_turn`** = re-send the conversation (including the paused response) with the same tools to continue.
- **ZDR + dynamic-filtering**: `_20260209` tools aren't ZDR-eligible by default (internal code execution). Set `"allowed_callers": ["direct"]` to disable dynamic filtering → ZDR-compatible.
- **Domain filtering**: no scheme; subdomains auto-included; one `*` per entry in path only. `*.example.com` is invalid.
- **Homograph gotcha**: Unicode lookalikes (`а` Cyrillic) can bypass ASCII filters.

---

## `tool_choice` reference (`define-tools.md`)
| Value | Behavior |
|---|---|
| `auto` (default with `tools`) | Claude decides |
| `any` | Must use some tool |
| `tool` + `name` | Must use this tool |
| `none` (default without `tools`) | No tools |

Forced modes (`any` / `tool`) prefill the assistant message → no natural-language preamble. **Extended thinking** supports only `auto` / `none`.

---

## Tool reference — composable optional properties (`tool-reference.md`)

All of these stack on the same tool:

| Property | Purpose | Not on |
|---|---|---|
| `cache_control` | Prompt-cache breakpoint | — |
| `strict` | Schema validation on names + inputs | `mcp_toolset` |
| `defer_loading` | Exclude from initial prompt; load on-demand via tool search | (on `mcp_toolset` use `default_config`) |
| `allowed_callers` | Restrict who calls (`"direct"` vs `"code_execution_20260120"`) | `mcp_toolset` |
| `input_examples` | Example inputs for complex tools | server tools; tools with search |
| `eager_input_streaming` | Fine-grained input streaming | user-defined only |

Anthropic-provided tools:
- **Server**: web_search, web_fetch, code_execution, tool_search, advisor, mcp_toolset.
- **Client** (trained-in schemas): memory, bash, text_editor, computer.

---

## How this connects to the book

- **Ch 5 Tool Use** → `tool-use-overview.md`, `how-tool-use-works.md`, `define-tools.md`, `handle-tool-calls.md`, `parallel-tool-use.md`, `strict-tool-use.md`.
- **Ch 10 MCP** → `mcp-connector.md`, `agent-sdk-mcp.md`, `claude-code-mcp.md` + `inputs/mcp-official/` for the protocol spec.
- **Ch 12 Exception Handling** → `handle-tool-calls.md` (`is_error`), `agent-sdk-custom-tools.md` error pattern (return `isError: true` to keep loop alive vs throw to stop it).
- **Ch 13 Human-in-the-loop** → `allowedTools`, permission modes, `managed-mcp.json`.
- **Ch 16 Resource-Aware Optimization** → `tool-search-tool.md`, `tool-use-with-prompt-caching.md`, `defer_loading`, Tool Runner auto-compaction.
- **Ch 18 Guardrails** → `strict: true`, `allowed_callers`, denylist patterns, domain filtering in server tools, homograph-attack note.

---

## Key takeaways
1. **Remote MCP from the API**: use MCP Connector (beta `mcp-client-2025-11-20`) — no MCP client required.
2. **Local MCP or prompts/resources from the API**: use the TypeScript SDK helpers (`mcpTools`, `mcpMessages`, `mcpResourceToContent`) alongside `@modelcontextprotocol/sdk`.
3. **Building an agent**: use the **Claude Agent SDK** with `query()` + `mcpServers` + in-process SDK MCP servers for custom tools.
4. **Scale past 30 tools**: turn on tool search (on by default in the SDK; `ENABLE_TOOL_SEARCH` envvar to tune) + `defer_loading`. Namespace tool names.
5. **Type-safe inputs**: set `strict: true` on tool definitions. Pair with `tool_choice: "any"` to guarantee a valid call.
6. **Keep the cache**: `cache_control` on the last tool; `defer_loading` is cache-safe.
7. **Parallel tool use**: all results in a single user message; use the `<use_parallel_tool_calls>` system-prompt block for aggressive parallelism.
8. **Anthropic doesn't host its own MCP spec** — for protocol details, use `inputs/mcp-official/` and `knowledge/mcp-official-summary.md`.

## Fetch status (2026-04-18)
- MCP upstream repo: already up to date.
- Anthropic docs: 10 tool-use pages + 3 Agent SDK MCP pages captured.
- `anthropic.com/engineering/*` articles + archive.org: still 403 to our fetcher.
- Full Claude Code docs index in `inputs/anthropic-mcp/claude-code-llms-index.txt` — use it to fetch specific pages on demand.
