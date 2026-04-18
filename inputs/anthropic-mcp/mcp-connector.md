# MCP connector

Source: https://platform.claude.com/docs/en/agents-and-tools/mcp-connector

---

Claude's Model Context Protocol (MCP) connector feature enables you to connect to remote MCP servers directly from the Messages API without a separate MCP client.

> **Current version**: This feature requires the beta header: `"anthropic-beta": "mcp-client-2025-11-20"`. The previous version (`mcp-client-2025-04-04`) is deprecated.

> This feature is **not** eligible for Zero Data Retention (ZDR).

## Key features

- **Direct API integration**: Connect to MCP servers without implementing an MCP client
- **Tool calling support**: Access MCP tools through the Messages API
- **Flexible tool configuration**: Enable all tools, allowlist specific tools, or denylist unwanted tools
- **Per-tool configuration**: Configure individual tools with custom settings
- **OAuth authentication**: Support for OAuth Bearer tokens for authenticated servers
- **Multiple servers**: Connect to multiple MCP servers in a single request

## Limitations

- Of the MCP spec, only **tool calls** are currently supported (resources/prompts are not exposed through the connector).
- Server must be publicly exposed over HTTP (Streamable HTTP or SSE). Local STDIO servers cannot be connected directly.
- Not supported on Amazon Bedrock or Google Vertex.

## Using the MCP connector in the Messages API

Two components:

1. **MCP Server Definition** (`mcp_servers` array): connection details (URL, authentication).
2. **MCP Toolset** (`tools` array): configures which tools to enable and how.

### Basic example

```bash
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: mcp-client-2025-11-20" \
  -d '{
    "model": "claude-opus-4-7",
    "max_tokens": 1000,
    "messages": [{"role": "user", "content": "What tools do you have available?"}],
    "mcp_servers": [
      {
        "type": "url",
        "url": "https://example-server.modelcontextprotocol.io/sse",
        "name": "example-mcp",
        "authorization_token": "YOUR_TOKEN"
      }
    ],
    "tools": [
      { "type": "mcp_toolset", "mcp_server_name": "example-mcp" }
    ]
  }'
```

```python
import anthropic
client = anthropic.Anthropic()
response = client.beta.messages.create(
    model="claude-opus-4-7",
    max_tokens=1000,
    messages=[{"role": "user", "content": "What tools do you have available?"}],
    mcp_servers=[{
        "type": "url",
        "url": "https://example-server.modelcontextprotocol.io/sse",
        "name": "example-mcp",
        "authorization_token": "YOUR_TOKEN",
    }],
    tools=[{"type": "mcp_toolset", "mcp_server_name": "example-mcp"}],
    betas=["mcp-client-2025-11-20"],
)
```

## MCP server configuration

```json
{
  "type": "url",
  "url": "https://example-server.modelcontextprotocol.io/sse",
  "name": "example-mcp",
  "authorization_token": "YOUR_TOKEN"
}
```

| Property | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Currently only `"url"` is supported |
| `url` | string | Yes | Must start with `https://` |
| `name` | string | Yes | Unique identifier; must be referenced by exactly one MCPToolset |
| `authorization_token` | string | No | OAuth authorization token if required |

## MCP toolset configuration

```json
{
  "type": "mcp_toolset",
  "mcp_server_name": "example-mcp",
  "default_config": {
    "enabled": true,
    "defer_loading": false
  },
  "configs": {
    "specific_tool_name": { "enabled": true, "defer_loading": true }
  }
}
```

| Property | Required | Description |
|---|---|---|
| `type` | Yes | Must be `"mcp_toolset"` |
| `mcp_server_name` | Yes | Must match a server name in `mcp_servers` |
| `default_config` | No | Defaults applied to all tools in the set |
| `configs` | No | Per-tool overrides (keyed by tool name) |
| `cache_control` | No | Cache breakpoint configuration |

### Per-tool config fields
| Property | Default | Description |
|---|---|---|
| `enabled` | `true` | Whether this tool is enabled |
| `defer_loading` | `false` | If true, tool description is not sent to the model initially. Used with [Tool Search Tool]. |

### Precedence (highest → lowest)
1. Tool-specific settings in `configs`
2. Set-level `default_config`
3. System defaults

## Common configuration patterns

### Enable all tools (simplest)
```json
{ "type": "mcp_toolset", "mcp_server_name": "google-calendar-mcp" }
```

### Allowlist
Set `default_config.enabled: false`, then enable specific tools via `configs`:
```json
{
  "type": "mcp_toolset",
  "mcp_server_name": "google-calendar-mcp",
  "default_config": { "enabled": false },
  "configs": {
    "search_events": { "enabled": true },
    "create_event": { "enabled": true }
  }
}
```

### Denylist
Enable all by default; disable specific tools:
```json
{
  "type": "mcp_toolset",
  "mcp_server_name": "google-calendar-mcp",
  "configs": {
    "delete_all_events": { "enabled": false },
    "share_calendar_publicly": { "enabled": false }
  }
}
```

## Validation rules

- `mcp_server_name` in an MCPToolset must match a server in `mcp_servers`.
- Every MCP server in `mcp_servers` must be referenced by exactly one MCPToolset.
- Unique toolset per server.
- Unknown tool names in `configs` → backend warning logged, no error (tools may be dynamic).

## Response content types

### MCP Tool Use Block
```json
{
  "type": "mcp_tool_use",
  "id": "mcptoolu_014Q35RayjACSWkSj4X2yov1",
  "name": "echo",
  "server_name": "example-mcp",
  "input": { "param1": "value1" }
}
```

### MCP Tool Result Block
```json
{
  "type": "mcp_tool_result",
  "tool_use_id": "mcptoolu_014Q35RayjACSWkSj4X2yov1",
  "is_error": false,
  "content": [{ "type": "text", "text": "Hello" }]
}
```

## Multiple MCP servers
Multiple entries in `mcp_servers` with a matching MCPToolset each. Per-toolset configuration is independent.

## Authentication

For servers that require OAuth, pass `authorization_token` in the server definition. Consumers handle the OAuth flow and token refresh.

### Obtaining a token for testing
Use the MCP Inspector:
```bash
npx @modelcontextprotocol/inspector
```
Select transport, enter server URL, click "Open Auth Settings" → "Quick OAuth Flow", then copy the `access_token`.

See the [MCP spec authorization section](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization).

## Client-side MCP helpers (TypeScript SDK)

For local stdio servers, MCP prompts, or MCP resources, the TypeScript SDK provides helpers:
```typescript
import {
  mcpTools, mcpMessages, mcpResourceToContent, mcpResourceToFile
} from "@anthropic-ai/sdk/helpers/beta/mcp";
```

| Helper | Description |
|---|---|
| `mcpTools(tools, mcpClient)` | Converts MCP tools → Claude API tools for `toolRunner()` |
| `mcpMessages(messages)` | Converts MCP prompt messages → Claude API messages |
| `mcpResourceToContent(resource)` | MCP resource → content block |
| `mcpResourceToFile(resource)` | MCP resource → file upload object |

Install both SDKs:
```bash
npm install @anthropic-ai/sdk @modelcontextprotocol/sdk
```

### Use MCP tools
```typescript
const transport = new StdioClientTransport({ command: "mcp-server", args: [] });
const mcpClient = new Client({ name: "my-client", version: "1.0.0" });
await mcpClient.connect(transport);

const { tools } = await mcpClient.listTools();
const runner = await anthropic.beta.messages.toolRunner({
  model: "claude-opus-4-7",
  max_tokens: 1024,
  messages: [{ role: "user", content: "What tools do you have available?" }],
  tools: mcpTools(tools, mcpClient)
});
```

Helpers throw `UnsupportedMCPValueError` if an MCP value isn't supported.

## Data retention
MCP Connector is not ZDR-eligible. Data retained per Anthropic's standard retention policy.

## Migration from `mcp-client-2025-04-04`

**Old**: tool configuration lived inside the MCP server definition (`tool_configuration.enabled`, `allowed_tools`).

**New**: tool configuration moved to the `tools` array as `mcp_toolset` objects. More flexible (allowlist/denylist/per-tool). Beta header changes to `mcp-client-2025-11-20`.

| Old pattern | New pattern |
|---|---|
| No `tool_configuration` | `mcp_toolset` with no `default_config` |
| `tool_configuration.enabled: false` | `default_config.enabled: false` |
| `tool_configuration.allowed_tools: [...]` | `default_config.enabled: false` + specific tools in `configs` |
