# Anthropic MCP — Summary

Source: `inputs/anthropic-mcp/` (captured from `platform.claude.com` and `code.claude.com`).

**What lives here vs. `knowledge/mcp-official-summary.md`:** the upstream MCP spec belongs to the protocol; this file covers **Anthropic-specific MCP surfaces** — how Claude products consume MCP and what features Anthropic built on top. Anthropic points readers to `modelcontextprotocol.io` for the spec itself.

## The four Anthropic-specific MCP surfaces

### 1. MCP Connector (Messages API)
Call remote MCP servers from the Claude Messages API without running your own MCP client.
- **Beta header**: `anthropic-beta: mcp-client-2025-11-20` (old `mcp-client-2025-04-04` deprecated).
- Two API components:
  - `mcp_servers[]` — server connection (`type`, `url`, `name`, `authorization_token`).
  - `tools[]` with `{ type: "mcp_toolset", mcp_server_name, default_config, configs }` — which tools, enabled/disabled, `defer_loading`.
- **Limitations**: tool calls only (no resources/prompts); HTTPS remote only (Streamable HTTP or SSE); no Bedrock/Vertex; not ZDR-eligible.
- **Response blocks**: `mcp_tool_use` + `mcp_tool_result` (mirror standard tool_use/tool_result).
- **Patterns**: enable-all (no config), allowlist (`default_config.enabled: false` + per-tool enables), denylist (disable specific).
- OAuth tokens obtained via the [MCP Inspector](https://github.com/modelcontextprotocol/inspector) for testing.
- File: `inputs/anthropic-mcp/mcp-connector.md`.

### 2. Tool Runner (SDK)
Python / TypeScript / Ruby SDK helper that drives the agentic loop automatically.
- Python: `@beta_tool` decorator infers JSON Schema from type hints + docstring.
- TypeScript: `betaZodTool` (Zod-validated) or `betaTool` (JSON Schema, runtime-unvalidated).
- Ruby: `Anthropic::BaseTool` + `Anthropic::BaseModel` input schemas.
- Supports automatic **compaction** — summaries when tokens exceed a threshold → long agentic sessions past context limits.
- Iteration: `for message in runner:` / `for await (const m of runner)` / `.until_done()` / `await runner`.
- Advanced: `generate_tool_call_response()` inspect → `set_messages_params()` / `append_messages()` modify next request.
- Error flow: tool exceptions caught → returned to Claude as `tool_result` with `is_error: true`. Set `ANTHROPIC_LOG=info|debug` for stack traces.
- Modify results (e.g., `cache_control: {type: "ephemeral"}` for prompt caching) before runner sends them back.
- Streaming: `stream=True` → yields `BetaMessageStream` per iteration.
- File: `inputs/anthropic-mcp/tool-runner.md`.

### 3. Tool Search Tool + `defer_loading`
Anthropic's answer to **tool-library scale** — the thing that makes MCP viable when you've got dozens of servers.
- **Problem it solves**: 55k+ tokens of tool definitions in a multi-server setup; Claude's tool-selection accuracy degrades past 30–50 tools.
- **Result**: >85% context reduction. Loads only 3–5 tools per turn.
- Two variants:
  - `tool_search_tool_regex_20251119` — Claude emits Python `re.search()` patterns.
  - `tool_search_tool_bm25_20251119` — natural-language queries.
- Mark individual tools (or MCP toolsets via `default_config`) `defer_loading: true`.
- **Prompt-cache preserved**: deferred tools are stripped from the prefix before cache-key computation; discovered tools are expanded inline in the conversation body.
- Limits: max **10,000 tools**; 3–5 returned per search; 200-char pattern max.
- Requires: Sonnet 4.0+, Opus 4.0+, Haiku 4.5+.
- **When to use**: 10+ tools; definitions >10k tokens; multi-server MCP setups; growing tool libraries.
- **Not supported with**: tool-use examples (`input_examples`).
- Optimization: namespace tools consistently (`github_`, `slack_`); put common keywords in descriptions; keep 3–5 most-used tools non-deferred.
- File: `inputs/anthropic-mcp/tool-search-tool.md`.

### 4. Claude Code's MCP client
The CLI/IDE Claude Code is itself an MCP host.
- Connects to local stdio, remote HTTP, SSE servers.
- Config: `.mcp.json` in project, user settings, or the MCP Registry via `claude mcp add …`.
- **Governance**: `managed-mcp.json` for enterprise-controlled server lists; `allowedMcpServers` / `deniedMcpServers` filters. Denylist beats allowlist.
- Full doc: `inputs/anthropic-mcp/claude-code-mcp.md`.

## Anthropic's take on tool use (mental model)

From `inputs/anthropic-mcp/how-tool-use-works.md`:
> Tool use is a contract between your application and the model. You specify what operations are available and what shape their inputs and outputs take; Claude decides when and how to call them.

Where tools run — three buckets:
| Bucket | Examples | Who executes |
|---|---|---|
| **User-defined client tools** | Your domain APIs | You — the majority of traffic |
| **Anthropic-schema client tools** | `bash`, `text_editor`, `computer`, `memory` | You, but the **schema is trained-in** so Claude calls them more reliably |
| **Server-executed tools** | `web_search`, `web_fetch`, `code_execution`, `tool_search` | Anthropic |

**The agentic loop** (client tools): `while stop_reason == "tool_use": execute, send tool_result, continue`. Exits on `end_turn`, `max_tokens`, `stop_sequence`, `refusal`. For server tools, iteration happens inside Anthropic — `pause_turn` means "re-send to continue".

**Tell that you should be using tools**: *if you're writing a regex to extract a decision from model output, that decision should have been a tool call.*

## Tool definition best practices (from `define-tools.md`)

1. **Descriptions are the #1 performance factor**. 3–4+ sentences. What, when, what-not, params, caveats, limitations.
2. **Consolidate related ops** — one tool with an `action` parameter beats three tools (`create_pr`, `review_pr`, `merge_pr` → `pr_action`).
3. **Namespace by service**: `github_list_prs`, `slack_send_message`. Critical at scale.
4. **High-signal responses**. Return stable IDs (slugs/UUIDs), only fields Claude needs. Bloat wastes context.
5. **`input_examples`** for complex/nested tools (not server tools).

## Tool-use HTTP shape (strict)

From `handle-tool-calls.md`:
- **`tool_result` must immediately follow its matching `tool_use`** — no messages in between.
- In a `user` message with tool results, **tool_result blocks come FIRST** in `content`; any text comes AFTER. Violation → 400.

## Errors

- Client-tool failure → return `tool_result` with `is_error: true` and an **instructive message** (`"Rate limit exceeded. Retry after 60 seconds."` — not `"failed"`).
- Invalid tool call → Claude retries **2–3 times** before giving up.
- **Eliminate invalid calls entirely** with `strict: true` on definitions.
- Server-tool errors handled transparently by Claude; you don't handle them.

## Tool reference — optional properties that compose

(From `tool-reference.md`.) All of these can be set on the same tool:

| Property | Purpose | Not on |
|---|---|---|
| `cache_control` | Prompt-cache breakpoint | — |
| `strict` | Schema validation on names + inputs | `mcp_toolset` |
| `defer_loading` | Exclude from initial prompt; load on-demand via tool search | (on `mcp_toolset` use `default_config`) |
| `allowed_callers` | Restrict who can call (`"direct"` vs `"code_execution_20260120"`) | `mcp_toolset` |
| `input_examples` | Example inputs for complex tools | server tools |
| `eager_input_streaming` | Fine-grained input streaming | — (user-defined only) |

Anthropic-provided tool catalog (major ones):
- **Server**: web_search, web_fetch, code_execution, tool_search, advisor, mcp_toolset.
- **Client**: memory, bash, text_editor, computer.

## `tool_choice` reference
| Value | Behavior |
|---|---|
| `auto` (default with `tools`) | Claude decides |
| `any` | Must use some tool |
| `tool` + `name` | Must use this tool |
| `none` (default without `tools`) | No tools |

Forced modes (`any`/`tool`) prefill the assistant message → no natural-language preamble. **Extended thinking** supports only `auto` / `none`.

## How this connects to the book

- **Ch 5 Tool Use** → `inputs/anthropic-mcp/tool-use-overview.md`, `how-tool-use-works.md`, `define-tools.md`, `handle-tool-calls.md`.
- **Ch 10 MCP** → `inputs/anthropic-mcp/mcp-connector.md` (+ `inputs/mcp-official/` for the protocol spec).
- **Ch 16 Resource-Aware Optimization** → `tool-search-tool.md` + `defer_loading` + prompt caching.
- **Ch 18 Guardrails** → `strict: true`, `is_error` messages, `allowed_callers`, denylist patterns.

## Key takeaways
1. For **remote MCP from your own app**, use the MCP Connector in the Messages API — no MCP client required.
2. For **local MCP or any prompts/resources**, use the TypeScript SDK helpers alongside `@modelcontextprotocol/sdk`.
3. To **scale past ~30 tools**, turn on the tool search tool and `defer_loading`. Namespace your tool names.
4. The SDK **Tool Runner** handles the loop + error wrapping + compaction — use the manual loop only when you need human-in-the-loop, custom logging, or conditional execution.
5. **Anthropic does not host its own MCP spec page** — they redirect to `modelcontextprotocol.io`. For protocol details, use `inputs/mcp-official/` and `knowledge/mcp-official-summary.md`.
