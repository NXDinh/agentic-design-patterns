# Define tools

Source: https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools

Specify tool schemas, write effective descriptions, and control when Claude calls your tools.

---

## Choosing a model
- Claude Opus 4.7 — complex tools, ambiguous queries; handles multiple tools better; seeks clarification when needed.
- Claude Haiku — straightforward tools; may infer missing parameters.

## Specifying client tools

Each tool in the `tools` array has:

| Parameter | Description |
|---|---|
| `name` | Must match `^[a-zA-Z0-9_-]{1,64}$` |
| `description` | Plaintext: what it does, when to use, how it behaves |
| `input_schema` | JSON Schema object for the expected parameters |
| `input_examples` | (Optional) Example input objects — see below |

For full optional properties (`cache_control`, `strict`, `defer_loading`, `allowed_callers`), see `tool-reference.md`.

### Example
```json
{
  "name": "get_weather",
  "description": "Get the current weather in a given location",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": { "type": "string", "description": "The city and state, e.g. San Francisco, CA" },
      "unit": { "type": "string", "enum": ["celsius", "fahrenheit"], "description": "Temperature unit" }
    },
    "required": ["location"]
  }
}
```

## Tool-use system prompt

When `tools` is provided, the API constructs a special system prompt:
```
In this environment you have access to a set of tools you can use to answer the user's question.
{{ FORMATTING INSTRUCTIONS }}
String and scalar parameters should be specified as is, while lists and objects should use JSON format. Note that spaces for string values are not stripped. The output is not expected to be valid XML and is parsed with regular expressions.
Here are the functions available in JSONSchema format:
{{ TOOL DEFINITIONS IN JSON SCHEMA }}
{{ USER SYSTEM PROMPT }}
{{ TOOL CONFIGURATION }}
```

## Best practices for tool definitions

- **Extremely detailed descriptions.** The single biggest factor in tool performance. Explain:
  - What the tool does
  - When it should be used (and when it shouldn't)
  - What each parameter means and how it affects behavior
  - Important caveats/limitations, including what it does NOT return
  Aim for 3–4+ sentences per tool description; more for complex tools.
- **Use `input_examples` for complex tools.** Schema-validated examples help with nested objects, format-sensitive params.
- **Consolidate related operations.** Instead of `create_pr`, `review_pr`, `merge_pr`, use a single tool with an `action` parameter. Fewer, more capable tools → less selection ambiguity.
- **Meaningful namespacing in names.** Prefix by service/resource: `github_list_prs`, `slack_send_message`. Critical at scale, especially with [tool search](./tool-search-tool.md).
- **High-signal tool responses.** Return semantic, stable IDs (slugs/UUIDs) not opaque internal refs. Include only fields Claude needs. Bloated responses waste context.

### Good description
```json
{
  "name": "get_stock_price",
  "description": "Retrieves the current stock price for a given ticker symbol. The ticker symbol must be a valid symbol for a publicly traded company on a major US stock exchange like NYSE or NASDAQ. The tool will return the latest trade price in USD. It should be used when the user asks about the current or most recent price of a specific stock. It will not provide any other information about the stock or company.",
  "input_schema": { "type": "object", "properties": { "ticker": { "type": "string", "description": "The stock ticker symbol, e.g. AAPL for Apple Inc." } }, "required": ["ticker"] }
}
```

### Bad description
```json
{
  "name": "get_stock_price",
  "description": "Gets the stock price for a ticker.",
  "input_schema": { "type": "object", "properties": { "ticker": { "type": "string" } }, "required": ["ticker"] }
}
```

## Providing tool use examples

Add `input_examples` with an array of example input objects (each must validate against `input_schema`):

```python
tools=[{
    "name": "get_weather",
    "description": "Get the current weather in a given location",
    "input_schema": { ... },
    "input_examples": [
        {"location": "San Francisco, CA", "unit": "fahrenheit"},
        {"location": "Tokyo, Japan", "unit": "celsius"},
        {"location": "New York, NY"},  # demonstrates 'unit' is optional
    ],
}]
```

### Limitations
- Each example must validate against the schema (else 400).
- Not supported for server tools (web search, code exec, etc.).
- Adds ~20–50 tokens simple, ~100–200 tokens complex.

## Controlling Claude's output

### `tool_choice` options

| Value | Behavior |
|---|---|
| `auto` (default when `tools` provided) | Claude decides whether to use any tool |
| `any` | Must use one of the provided tools |
| `tool` (with `name`) | Must use this specific tool |
| `none` (default when no `tools`) | Cannot use tools |

With `any` / `tool`, the API prefills the assistant message to force a tool use — no natural-language preamble, even if asked.

**Caveats:**
- With **extended thinking**: only `auto` and `none` are supported.
- Changes to `tool_choice` invalidate cached message blocks (tool definitions and system prompts remain cached).
- Claude Mythos Preview doesn't support forced tool use.

> **Guaranteed tool calls with strict tools**: Combine `tool_choice: {"type": "any"}` with `strict: true` on definitions to guarantee a call AND exact schema conformance.

## Model responses with tools

Claude often comments on what it's doing before invoking a tool:
```json
{
  "role": "assistant",
  "content": [
    {"type": "text", "text": "I'll help you check the current weather and time in San Francisco."},
    {"type": "tool_use", "id": "toolu_...", "name": "get_weather", "input": {"location": "San Francisco, CA"}}
  ]
}
```
Treat this text like any other assistant-generated text — don't rely on specific formatting.
