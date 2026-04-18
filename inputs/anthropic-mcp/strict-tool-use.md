# Strict tool use

Source: https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use

Enforce JSON Schema compliance on Claude's tool inputs with **grammar-constrained sampling**.

---

Setting `strict: true` on a tool definition uses grammar-constrained sampling to guarantee Claude's tool inputs match your JSON Schema. Use when you need to:
- Validate tool parameters
- Build agentic workflows
- Ensure type-safe function calls
- Handle complex tools with nested properties

## Why it matters

Without strict mode, Claude might return incompatible types (`"2"` instead of `2`) or omit required fields, breaking your functions.

With `strict: true`:
- Functions receive correctly-typed args every time
- No retry logic needed
- Production-ready agents that work consistently at scale

Example — booking system needs `passengers: int`. Without strict: Claude might send `passengers: "two"` or `passengers: "2"`. With strict: always `passengers: 2`.

## Quick start

```python
from anthropic import Anthropic

client = Anthropic()
response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    messages=[{"role": "user", "content": "What's the weather in San Francisco?"}],
    tools=[{
        "name": "get_weather",
        "description": "Get the current weather in a given location",
        "strict": True,  # Enable strict mode
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "The city and state"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
            },
            "required": ["location"],
            "additionalProperties": False,  # REQUIRED for strict
        },
    }],
)
```

```typescript
await client.messages.create({
  model: "claude-opus-4-7",
  max_tokens: 1024,
  messages: [{ role: "user", content: "What's the weather like in San Francisco?" }],
  tools: [{
    name: "get_weather",
    description: "Get the current weather in a given location",
    strict: true,
    input_schema: {
      type: "object",
      properties: {
        location: { type: "string" },
        unit: { type: "string", enum: ["celsius", "fahrenheit"] }
      },
      required: ["location"],
      additionalProperties: false
    }
  }]
});
```

### Response format
```json
{
  "type": "tool_use",
  "name": "get_weather",
  "input": { "location": "San Francisco, CA" }
}
```

**Guarantees**:
- Tool `input` strictly follows `input_schema`
- Tool `name` is always valid

## How it works
1. Define your JSON Schema in `input_schema`.
2. Add `"strict": true` as a top-level property alongside `name`, `description`, `input_schema`.
3. When Claude uses the tool, `input` will strictly follow the schema and `name` will always be valid.

## Common patterns

### Constrained integer via enum
```json
{
  "passengers": { "type": "integer", "enum": [1, 2, 3, 4, 5, 6, 7, 8, 9, 10] }
}
```

### Date format
```json
{ "departure_date": { "type": "string", "format": "date" } }
```

### Multi-tool agentic workflow — all strict
Helps guarantee that each step's tool inputs are valid, reducing retries across long agent runs.

## Force strict + force tool use

Combine with `tool_choice`:
- `tool_choice: {"type": "any"}` + `strict: true` on all tools → **guaranteed** that SOME tool runs AND inputs match schema.
- `tool_choice: {"type": "tool", "name": "X"}` + `strict: true` → guaranteed that tool X runs with valid inputs.

## Data retention

Strict tool use compiles `input_schema` definitions into grammars using the same pipeline as structured outputs. **Tool schemas are temporarily cached for up to 24 hours since last use.** Prompts and responses are not retained beyond the API response.

### HIPAA
Strict tool use is HIPAA eligible, **but PHI must not be included in tool schema definitions**. The API caches compiled schemas separately from message content; these cached schemas do not receive the same PHI protections as prompts/responses.

**Do not include PHI** in:
- `input_schema` property names
- `enum` values
- `const` values
- `pattern` regular expressions

PHI should only appear in message content (prompts and responses).
