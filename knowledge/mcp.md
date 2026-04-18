# Model Context Protocol (MCP) — Reference

Source: Ch 10, Agentic Design Patterns (Gulli/Sauco).

## TL;DR
MCP is an open, client-server protocol that standardizes how LLMs/agents discover and use external capabilities. One MCP server can serve any compliant client (Claude, GPT, Gemini, etc.). Think "universal adapter" vs. per-vendor function calling.

## MCP vs. plain function calling
| | Function Calling | MCP |
|---|---|---|
| Standardization | Vendor-specific | Open protocol |
| Scope | One-to-one LLM→function | Client-server, many tools per server |
| Discovery | Hard-coded in prompt | Dynamic via manifest |
| Reusability | Tightly coupled | Server reused across clients |
| Best for | Small, fixed tool set | Growing / shared tool ecosystem, enterprise |

**Rule of thumb:** <5 tools, one app, one model → function calling. Shared tools across projects/models, or plan to grow → MCP.

## Architecture
1. **LLM** — decides what action is needed.
2. **MCP Client** — wrapper around the LLM; translates intent to MCP calls; handles discovery/connection.
3. **MCP Server** — exposes capabilities for a domain (DB, email, filesystem, internal API).
4. **3P service** — the actual backend the server fronts.

## The three server primitives
- **Resource** — static data addressable by URI (a file, DB record, PDF). Read-only context.
- **Tool** — executable function with side effects or computation (send_email, run_query).
- **Prompt** — templated prompt the server supplies to guide interaction with its resources/tools.

Get these roles right — tools are verbs, resources are nouns, prompts are instructions.

## Interaction flow
1. **Discovery**: client asks server `list_tools` / `list_resources` / `list_prompts`.
2. **Formulate**: LLM picks a tool + args.
3. **Send**: client issues standardized call.
4. **Execute**: server authenticates, validates, runs, returns structured response.
5. **Update context**: client feeds result back to the LLM.

## Transport layers
- **Local**: JSON-RPC over STDIO — fast, secure, inter-process.
- **Remote**: Streamable HTTP / SSE — shared, scalable, web-friendly.

## Design considerations (critical before building)
- **Agent-friendly APIs**: don't blindly wrap legacy APIs. If the legacy endpoint returns "one ticket at a time," the agent will thrash at volume. Add filter / sort / pagination deterministically in the server.
- **Agent-friendly payloads**: return formats the model can read (Markdown, JSON, plaintext) — not raw PDFs or binary. If source is PDF, convert to text inside the server.
- **Security**: authN/authZ per client; per-tool permissions; audit logs. Treat every tool as a capability grant.
- **Error handling**: define a shape (`code`, `message`, `retryable`, `hint`) so the LLM can recover or escalate.
- **Local vs remote**: local for sensitive data + speed; remote for shared org tools.
- **Discovery dynamics**: agents should re-query capabilities, not cache forever — enables hot-adding tools without redeploy.
- **Batch vs on-demand**: define upfront; batch endpoints need different ergonomics than interactive.

## Common use cases
- Database integration (SQL, BigQuery).
- Generative media orchestration (image/video/audio pipelines).
- External API fan-out (CRM, ticketing, weather, stocks).
- Reasoning-based information extraction over private corpora.
- Custom internal tools exposed to any compliant client.
- Complex multi-step workflows composing multiple servers.
- IoT device control.
- Financial services automation.

## Implementation — FastMCP (Python)
Minimal server:
```python
from fastmcp import FastMCP
mcp = FastMCP()

@mcp.tool
def greet(name: str) -> str:
    """Return a personalized greeting. Use when the user asks to greet someone."""
    return f"Hello, {name}!"

if __name__ == "__main__":
    mcp.run(transport="http", host="127.0.0.1", port=8000)
```
Docstrings + type hints auto-generate the schema. Add resources with `@mcp.resource`, prompts with `@mcp.prompt`.

## Connecting from Claude Code
Add to `.mcp.json` (project) or user settings:
```json
{
  "mcpServers": {
    "my-server": {
      "command": "python",
      "args": ["path/to/server.py"],
      "env": {"API_KEY": "..."}
    }
  }
}
```
For HTTP servers use the HTTP transport shape. Verify with `/mcp` in the Claude Code CLI.

## Implementation — ADK (consumer side)
- Local stdio: `MCPToolset(connection_params=StdioServerParameters(command="npx", args=["-y", "@modelcontextprotocol/server-filesystem", ABS_PATH]))`.
- Remote HTTP: `MCPToolset(connection_params=HttpServerParameters(url="http://localhost:8000"))`.
- Use `tool_filter=[...]` to expose a subset of the server's tools.

## Checklist for designing an MCP server
- [ ] Decide tool vs resource vs prompt for every capability.
- [ ] APIs support filter/sort/paginate — not just "get everything."
- [ ] Payloads are text-ish (Markdown/JSON), not opaque binary.
- [ ] AuthN + per-tool authZ.
- [ ] Typed error envelope.
- [ ] Local vs remote decision documented.
- [ ] Versioning scheme for schema changes.
- [ ] Logs: request, response, latency, caller identity.
- [ ] README shows a 3-line connect example for each major client.

## Pitfalls
- Wrapping legacy CRUD 1:1 → agent can't work efficiently. **Fix:** add bulk/filtered endpoints.
- Returning huge blobs → burns context. **Fix:** return refs + a `fetch_detail` tool.
- No discovery doc → clients hard-code. **Fix:** rich descriptions in manifest.
- Shared server, no per-client scope → security nightmare. **Fix:** scope-based auth.
