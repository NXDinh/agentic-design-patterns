# Anthropic MCP docs — captured snapshot

Official Anthropic documentation about MCP, tool use, and related SDK/API features. Fetched from `platform.claude.com` and `code.claude.com`. This folder is text-only; no images.

## Contents
| File | Source URL |
|---|---|
| `mcp-connector.md` | https://platform.claude.com/docs/en/agents-and-tools/mcp-connector |
| `remote-mcp-servers.md` | https://platform.claude.com/docs/en/agents-and-tools/remote-mcp-servers |
| `tool-use-overview.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview |
| `how-tool-use-works.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works |
| `define-tools.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools |
| `handle-tool-calls.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls |
| `tool-runner.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner |
| `tool-search-tool.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool |
| `tool-reference.md` | https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference |
| `claude-code-mcp.md` | https://code.claude.com/docs/en/mcp |

## Fetch notes
- `modelcontextprotocol.io` still returned 403 to bots — use `inputs/mcp-official/` (cloned from the upstream repo) for the protocol spec itself.
- `platform.claude.com/docs/en/mcp` redirects to `modelcontextprotocol.io` (Anthropic points readers upstream for the spec).
- Anthropic's unique contributions are in: the **MCP connector** (Messages API integration), **tool-use SDK ergonomics**, the **tool search tool** + `defer_loading` (at-scale MCP), and **Claude Code's MCP client**.
