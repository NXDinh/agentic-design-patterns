# Give Claude custom tools (Claude Agent SDK)

Source: https://code.claude.com/docs/en/agent-sdk/custom-tools

Define custom tools with the Claude Agent SDK's **in-process MCP server** so Claude can call your functions, hit your APIs, and perform domain-specific operations.

---

## Quick reference

| Goal | Action |
|---|---|
| Define a tool | `@tool` (Python) or `tool()` (TS) — name, description, schema, handler |
| Register with Claude | Wrap in `create_sdk_mcp_server` / `createSdkMcpServer`, pass to `mcpServers` in `query()` |
| Pre-approve | Add to `allowedTools` |
| Remove a built-in tool from context | Pass `tools: []` or a `tools` array of only-wanted built-ins |
| Let Claude parallelize | Set `readOnlyHint: true` |
| Handle errors without stopping the loop | Return `isError: true` instead of throwing |
| Return images/files | Use `image` or `resource` blocks in content array |
| Scale to many tools | Use tool search (`agent-sdk-tool-search.md`) |

## Create a custom tool

A tool has four parts:
- **Name** — unique identifier
- **Description** — Claude reads this to decide when to call
- **Input schema** — TS uses Zod (handler args auto-typed); Python takes a dict (`{"latitude": float}`) or a full JSON Schema dict for enums/ranges/optional
- **Handler** — async function; returns `{ content: [...], isError? }`

### Weather tool

```python
from typing import Any
import httpx
from claude_agent_sdk import tool, create_sdk_mcp_server

@tool(
    "get_temperature",
    "Get the current temperature at a location",
    {"latitude": float, "longitude": float},
)
async def get_temperature(args: dict[str, Any]) -> dict[str, Any]:
    async with httpx.AsyncClient() as client:
        response = await client.get(
            "https://api.open-meteo.com/v1/forecast",
            params={
                "latitude": args["latitude"], "longitude": args["longitude"],
                "current": "temperature_2m", "temperature_unit": "fahrenheit",
            },
        )
        data = response.json()
    return {
        "content": [{"type": "text", "text": f"Temperature: {data['current']['temperature_2m']}°F"}]
    }

weather_server = create_sdk_mcp_server(
    name="weather", version="1.0.0", tools=[get_temperature],
)
```

```typescript
import { tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const getTemperature = tool(
  "get_temperature",
  "Get the current temperature at a location",
  {
    latitude: z.number().describe("Latitude coordinate"),
    longitude: z.number().describe("Longitude coordinate")
  },
  async (args) => {
    const response = await fetch(
      `https://api.open-meteo.com/v1/forecast?latitude=${args.latitude}&longitude=${args.longitude}&current=temperature_2m&temperature_unit=fahrenheit`
    );
    const data: any = await response.json();
    return { content: [{ type: "text", text: `Temperature: ${data.current.temperature_2m}°F` }] };
  }
);

const weatherServer = createSdkMcpServer({
  name: "weather", version: "1.0.0", tools: [getTemperature]
});
```

> **Optional params**: TypeScript — add `.default()` to the Zod field. Python — leave the parameter OUT of the schema dict (every key is required), mention in the description, read with `args.get()`.

### Call the tool

```typescript
for await (const message of query({
  prompt: "What's the temperature in San Francisco?",
  options: {
    mcpServers: { weather: weatherServer },
    allowedTools: ["mcp__weather__get_temperature"]
  }
})) {
  if (message.type === "result" && message.subtype === "success") console.log(message.result);
}
```

The key in `mcpServers` becomes `{server_name}` in the fully-qualified tool name: `mcp__{server_name}__{tool_name}`.

### Multiple tools per server
A server holds any number of tools (list them in the `tools` array). List each individually in `allowedTools` or use `mcp__weather__*` wildcard.

Every tool consumes context on every turn. If dozens of tools → use tool search.

## Tool annotations

Passed as the fifth arg to `tool()` in TS, or `annotations=` kwarg in Python. Booleans:

| Field | Default | Meaning |
|---|---|---|
| `readOnlyHint` | `false` | Tool doesn't modify its environment. **Controls whether tool can be called in parallel with other read-only tools.** |
| `destructiveHint` | `true` | May perform destructive updates. Informational only. |
| `idempotentHint` | `false` | Repeated calls with same args have no additional effect. Informational only. |
| `openWorldHint` | `true` | Tool reaches systems outside your process. Informational only. |

> Annotations are metadata, not enforcement. A tool marked `readOnlyHint: true` can still write if the handler does. Keep annotations accurate.

```typescript
tool(
  "get_temperature",
  "Get the current temperature at a location",
  { latitude: z.number(), longitude: z.number() },
  async (args) => ({ content: [{ type: "text", text: "..." }] }),
  { annotations: { readOnlyHint: true } }
);
```

## Control tool access

### Tool name format
`mcp__{server_name}__{tool_name}`

### `tools` vs allowed/disallowed

| Option | Layer | Effect |
|---|---|---|
| `tools: ["Read", "Grep"]` | Availability | Only listed built-ins in context. MCP unaffected. |
| `tools: []` | Availability | All built-ins removed. Claude can only use MCP tools. |
| Allowed tools | Permission | Listed tools run without prompt. Unlisted remain available via permission flow. |
| Disallowed tools | Permission | Every call to a listed tool denied. Tool STAYS in context — Claude may waste a turn trying. |

**Prefer `tools` over disallowed for restricting built-ins** — removes from context entirely.

## Handle errors

| What happens | Result |
|---|---|
| Handler throws uncaught | Agent loop STOPS. `query` fails. |
| Handler returns `isError: true` / `"is_error": True` | Agent loop CONTINUES. Claude sees the error as data, can retry / try different tool / explain. |

```python
@tool("fetch_data", "Fetch data from an API", {"endpoint": str})
async def fetch_data(args):
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get(args["endpoint"])
            if response.status_code != 200:
                return {
                    "content": [{"type": "text", "text": f"API error: {response.status_code} {response.reason_phrase}"}],
                    "is_error": True,
                }
            data = response.json()
            return {"content": [{"type": "text", "text": json.dumps(data, indent=2)}]}
    except Exception as e:
        return {
            "content": [{"type": "text", "text": f"Failed to fetch data: {str(e)}"}],
            "is_error": True,
        }
```

## Return images and resources

The `content` array accepts `text`, `image`, `resource` blocks. Mix them.

### Images (inline base64)
| Field | Type | Notes |
|---|---|---|
| `type` | `"image"` | — |
| `data` | string | Base64 raw bytes, no `data:image/...;base64,` prefix |
| `mimeType` | string | Required (e.g. `image/png`) |

```python
import base64, httpx

@tool("fetch_image", "Fetch an image from a URL", {"url": str})
async def fetch_image(args):
    async with httpx.AsyncClient() as client:
        response = await client.get(args["url"])
    return {
        "content": [{
            "type": "image",
            "data": base64.b64encode(response.content).decode("ascii"),
            "mimeType": response.headers.get("content-type", "image/png"),
        }]
    }
```

### Resources (URI-addressable content)
URI is a label Claude can reference later — the actual content rides in `text` or `blob`.

| Field | Type | Notes |
|---|---|---|
| `type` | `"resource"` | — |
| `resource.uri` | string | Any URI scheme |
| `resource.text` | string | Content if text. Or use `blob`. |
| `resource.blob` | string | Base64 if binary |
| `resource.mimeType` | string | Optional |

```typescript
return {
  content: [{
    type: "resource",
    resource: {
      uri: "file:///tmp/report.md",   // Label, NOT a path SDK reads
      mimeType: "text/markdown",
      text: "# Report\n..."            // Actual content inline
    }
  }]
};
```

These shapes are the MCP `CallToolResult` type.

## Example: unit converter with enums and error handling

```python
@tool(
    "convert_units",
    "Convert a value from one unit to another",
    {
        "type": "object",
        "properties": {
            "unit_type": {"type": "string", "enum": ["length", "temperature", "weight"], "description": "Category of unit"},
            "from_unit": {"type": "string"},
            "to_unit": {"type": "string"},
            "value": {"type": "number"},
        },
        "required": ["unit_type", "from_unit", "to_unit", "value"],
    },
)
async def convert_units(args):
    conversions = {
        "length": {
            "kilometers_to_miles": lambda v: v * 0.621371,
            "miles_to_kilometers": lambda v: v * 1.60934,
        },
        "temperature": {
            "celsius_to_fahrenheit": lambda v: (v * 9)/5 + 32,
            "fahrenheit_to_celsius": lambda v: (v - 32) * 5/9,
        },
    }
    key = f"{args['from_unit']}_to_{args['to_unit']}"
    fn = conversions.get(args["unit_type"], {}).get(key)
    if not fn:
        return {
            "content": [{"type": "text", "text": f"Unsupported conversion: {args['from_unit']} to {args['to_unit']}"}],
            "is_error": True,
        }
    result = fn(args["value"])
    return {"content": [{"type": "text", "text": f"{args['value']} {args['from_unit']} = {result:.4f} {args['to_unit']}"}]}
```

## Related
- **Connect MCP servers** — `agent-sdk-mcp.md`
- **Tool search** — `agent-sdk-tool-search.md`
- **MCP spec** — https://modelcontextprotocol.io
