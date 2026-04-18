# Anthropic MCP docs — captured snapshot

Official Anthropic documentation about MCP, tool use, and related SDK/API features. Fetched from `platform.claude.com` and `code.claude.com`. Text-only (no images).

## Contents

### Protocol / API surface
| File | Source URL |
|---|---|
| `mcp-connector.md` | https://platform.claude.com/docs/en/agents-and-tools/mcp-connector |
| `remote-mcp-servers.md` | https://platform.claude.com/docs/en/agents-and-tools/remote-mcp-servers |
| `server-tools.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools |

### Tool use (API)
| File | Source URL |
|---|---|
| `tool-use-overview.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview |
| `how-tool-use-works.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works |
| `define-tools.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools |
| `handle-tool-calls.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls |
| `strict-tool-use.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use |
| `parallel-tool-use.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use |
| `tool-runner.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner |
| `tool-search-tool.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool |
| `tool-use-with-prompt-caching.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-use-with-prompt-caching |
| `tool-reference.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference |

### Claude Agent SDK
| File | Source URL |
|---|---|
| `agent-sdk-mcp.md` | https://code.claude.com/docs/en/agent-sdk/mcp |
| `agent-sdk-tool-search.md` | https://code.claude.com/docs/en/agent-sdk/tool-search |
| `agent-sdk-custom-tools.md` | https://code.claude.com/docs/en/agent-sdk/custom-tools |

### Claude Code
| File | Source URL |
|---|---|
| `claude-code-mcp.md` | https://code.claude.com/docs/en/mcp |
| `claude-code-llms-index.txt` | https://code.claude.com/docs/llms.txt — full Claude Code docs index for on-demand fetching |

## Fetch notes (2026-04-18)

- **`modelcontextprotocol.io`** — still 403 to bots. Protocol spec is in `inputs/mcp-official/` via the GitHub clone.
- **`platform.claude.com/docs/en/mcp`** and `/build-with-claude/mcp` — both **redirect to `modelcontextprotocol.io`**. Anthropic points readers upstream for the protocol itself.
- **`www.anthropic.com/news/*` and `www.anthropic.com/engineering/*`** — 403 to bots. Archive.org also 403.
- **MCP upstream repo** — verified already up to date (no new commits since our clone).

## Scope of Anthropic's unique MCP surfaces (vs. upstream)
1. **MCP Connector (Messages API)** — call remote MCP servers without an MCP client. Beta `mcp-client-2025-11-20`.
2. **Tool Runner SDK** — Python / TypeScript / Ruby loop automation with auto-compaction.
3. **Tool Search Tool + `defer_loading`** — scale past 30+ tools while preserving prompt cache.
4. **`mcp_toolset`** — configure allow/deny/defer per tool in the API.
5. **Strict tool use** — grammar-constrained sampling to guarantee schema conformance.
6. **Claude Code's MCP client** — `.mcp.json`, `managed-mcp.json`, allow/deny lists.
7. **Claude Agent SDK MCP** — `query()` with `mcpServers`, in-process SDK MCP servers, `ENABLE_TOOL_SEARCH` envvar.
