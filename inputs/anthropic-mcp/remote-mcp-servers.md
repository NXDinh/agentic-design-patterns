# Remote MCP servers

Source: https://platform.claude.com/docs/en/agents-and-tools/remote-mcp-servers

---

Several companies have deployed remote MCP servers that developers can connect to via the Anthropic MCP connector API. These servers expand capabilities by providing remote access to services and tools through the MCP protocol.

> These are **third-party services** — not owned, operated, or endorsed by Anthropic. Review each server's security practices and terms before connecting, and only connect to servers you trust.

## Connecting to remote MCP servers

1. Review the documentation for the specific server you want to use.
2. Ensure you have the necessary authentication credentials.
3. Follow the server-specific connection instructions provided by each company.

See `mcp-connector.md` for using remote MCP servers with the Claude API.

## Finding servers

The page dynamically renders from the MCP registry (filtered to `worksWith: mcp-connector`). For the full index:

- **MCP registry API** — `https://api.anthropic.com/mcp-registry/v0/servers?version=latest&visibility=commercial`
- **Hundreds more servers on GitHub** — https://github.com/modelcontextprotocol/servers
