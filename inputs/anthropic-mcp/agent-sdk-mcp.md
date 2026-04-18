# Connect to external tools with MCP (Claude Agent SDK)

Source: https://code.claude.com/docs/en/agent-sdk/mcp

Configure MCP servers to extend your agent with external tools. Covers transport types, tool search, authentication, and error handling.

---

MCP servers can run as **local processes**, connect over **HTTP**, or execute **in-process** within your SDK application.

## Quickstart — connect to Claude Code's docs MCP server

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Use the docs MCP server to explain what hooks are in Claude Code",
  options: {
    mcpServers: {
      "claude-code-docs": { type: "http", url: "https://code.claude.com/docs/mcp" }
    },
    allowedTools: ["mcp__claude-code-docs__*"]
  }
})) {
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

async def main():
    options = ClaudeAgentOptions(
        mcp_servers={"claude-code-docs": {"type": "http", "url": "https://code.claude.com/docs/mcp"}},
        allowed_tools=["mcp__claude-code-docs__*"],
    )
    async for message in query(prompt="Use the docs MCP server to explain what hooks are in Claude Code", options=options):
        if isinstance(message, ResultMessage) and message.subtype == "success":
            print(message.result)

asyncio.run(main())
```

## Add an MCP server

### In code
Pass MCP servers via `mcpServers` in `query()` options:
```typescript
options: {
  mcpServers: {
    filesystem: {
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
    }
  },
  allowedTools: ["mcp__filesystem__*"]
}
```

### From a config file — `.mcp.json`
Project-root `.mcp.json`, picked up when the `project` setting source is enabled (default for `query()`):
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
    }
  }
}
```
If you set `settingSources` explicitly, include `"project"` for this file to load.

## Allow MCP tools

MCP tools require **explicit permission**. Without it, Claude sees tools are available but can't call them.

### Tool naming convention
`mcp__<server-name>__<tool-name>`. E.g. GitHub server "github" with `list_issues` → `mcp__github__list_issues`.

### Grant access with `allowedTools`
```typescript
allowedTools: [
  "mcp__github__*",           // All tools from github server
  "mcp__db__query",           // Only query tool from db server
  "mcp__slack__send_message"  // Specific slack tool
]
```

> **Prefer `allowedTools` over permission modes for MCP access.** `permissionMode: "acceptEdits"` does NOT auto-approve MCP tools (only file edits and filesystem Bash). `permissionMode: "bypassPermissions"` DOES auto-approve MCP tools but also disables all other safety prompts — too broad. A wildcard in `allowedTools` grants exactly the server you want.

### Discover available tools
```typescript
for await (const message of query({ prompt: "...", options })) {
  if (message.type === "system" && message.subtype === "init") {
    console.log("Available MCP tools:", message.mcp_servers);
  }
}
```

## Transport types

Pick based on the server's docs:
- Command to run → **stdio**
- URL → **HTTP or SSE**
- Building your own in code → **SDK MCP server** (in-process)

### stdio servers
Local processes via stdin/stdout.
```typescript
options: {
  mcpServers: {
    github: {
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-github"],
      env: { GITHUB_TOKEN: process.env.GITHUB_TOKEN }
    }
  },
  allowedTools: ["mcp__github__list_issues", "mcp__github__search_issues"]
}
```

### HTTP/SSE servers
Cloud-hosted MCP + remote APIs:
```typescript
options: {
  mcpServers: {
    "remote-api": {
      type: "sse",                            // or "http" for non-streaming
      url: "https://api.example.com/mcp/sse",
      headers: { Authorization: `Bearer ${process.env.API_TOKEN}` }
    }
  },
  allowedTools: ["mcp__remote-api__*"]
}
```

### SDK MCP servers (in-process)
Define custom tools in your code — no separate process. See `agent-sdk-custom-tools.md`.

## MCP tool search

With many tools configured, definitions eat context. Tool search withholds definitions and loads only what Claude needs per turn. **Enabled by default.** See `agent-sdk-tool-search.md`.

## Authentication

### Environment variables
```typescript
mcpServers: {
  github: {
    command: "npx",
    args: ["-y", "@modelcontextprotocol/server-github"],
    env: { GITHUB_TOKEN: process.env.GITHUB_TOKEN }
  }
}
```
`.mcp.json` uses `${GITHUB_TOKEN}` expansion at runtime.

### HTTP headers (remote servers)
```typescript
mcpServers: {
  "secure-api": {
    type: "http",
    url: "https://api.example.com/mcp",
    headers: { Authorization: `Bearer ${process.env.API_TOKEN}` }
  }
}
```

### OAuth 2.1
MCP spec supports [OAuth 2.1](https://modelcontextprotocol.io/specification/2025-03-26/basic/authorization). The SDK doesn't handle the OAuth flow automatically — complete it in your app, then pass the access token as a bearer header:
```typescript
const accessToken = await getAccessTokenFromOAuthFlow();
const options = {
  mcpServers: {
    "oauth-api": {
      type: "http",
      url: "https://api.example.com/mcp",
      headers: { Authorization: `Bearer ${accessToken}` }
    }
  },
  allowedTools: ["mcp__oauth-api__*"]
};
```

## Examples

### List GitHub issues
```bash
export GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
```
```typescript
for await (const message of query({
  prompt: "List the 3 most recent issues in anthropics/claude-code",
  options: {
    mcpServers: {
      github: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-github"],
        env: { GITHUB_TOKEN: process.env.GITHUB_TOKEN }
      }
    },
    allowedTools: ["mcp__github__list_issues"]
  }
})) {
  if (message.type === "system" && message.subtype === "init") console.log("MCP servers:", message.mcp_servers);
  if (message.type === "assistant") {
    for (const block of message.message.content) {
      if (block.type === "tool_use" && block.name.startsWith("mcp__")) console.log("MCP tool called:", block.name);
    }
  }
  if (message.type === "result" && message.subtype === "success") console.log(message.result);
}
```

### Query a Postgres database
```typescript
const connectionString = process.env.DATABASE_URL;
for await (const message of query({
  prompt: "How many users signed up last week? Break it down by day.",
  options: {
    mcpServers: {
      postgres: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-postgres", connectionString]
      }
    },
    allowedTools: ["mcp__postgres__query"]
  }
})) {
  if (message.type === "result" && message.subtype === "success") console.log(message.result);
}
```

## Error handling

The SDK emits a `system` message with `subtype: "init"` at the start of each query containing connection status for each MCP server:
```typescript
for await (const message of query({ prompt: "Process data", options })) {
  if (message.type === "system" && message.subtype === "init") {
    const failedServers = message.mcp_servers.filter(s => s.status !== "connected");
    if (failedServers.length > 0) console.warn("Failed to connect:", failedServers);
  }
  if (message.type === "result" && message.subtype === "error_during_execution") {
    console.error("Execution failed");
  }
}
```

## Troubleshooting

### Server shows "failed" status
Common causes:
- **Missing env vars** — check `env` field matches the server's expectations.
- **Server not installed** — for `npx`, ensure the package exists and Node is on PATH.
- **Invalid connection string** — check format and DB reachability.
- **Network issues** — URL reachable? Firewalls?

### Tools not being called
Add them to `allowedTools` — `mcp__servername__*`.

### Connection timeouts
Default MCP SDK timeout is 60s. Lighter-weight server, pre-warm, or inspect server logs.

## Related
- **Custom tools** — `agent-sdk-custom-tools.md`
- **Tool search** — `agent-sdk-tool-search.md`
- **MCP server directory** — https://github.com/modelcontextprotocol/servers
