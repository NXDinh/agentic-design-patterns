# Handle tool calls

Source: https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls

Parse `tool_use` blocks, format `tool_result` responses, and handle errors with `is_error`.

---

> Simpler with Tool Runner — the manual handling here is automated by `tool-runner.md`. Use this when you need custom control over tool execution.

## Handling results from client tools

Response has `stop_reason: "tool_use"` and one or more `tool_use` blocks with:
- `id` — unique identifier for this block (used to match results later)
- `name` — tool name
- `input` — object matching the tool's `input_schema`

### Example response
```json
{
  "id": "msg_01Aq9w938a90dw8q",
  "model": "claude-opus-4-7",
  "stop_reason": "tool_use",
  "role": "assistant",
  "content": [
    { "type": "text", "text": "I'll check the current weather in San Francisco for you." },
    { "type": "tool_use", "id": "toolu_01A09q90qw90lq917835lq9", "name": "get_weather", "input": { "location": "San Francisco, CA", "unit": "celsius" } }
  ]
}
```

### Steps when you receive a tool use for a client tool
1. Extract `name`, `id`, `input` from the `tool_use` block.
2. Run your tool with the `input`.
3. Continue the conversation with a `user` message containing a `tool_result` block:
   - `tool_use_id` — matches the tool_use `id`.
   - `content` — string, or array of nested blocks (text, image, document).
   - `is_error` (optional) — `true` if the tool errored.

### Important formatting rules

- **Tool result blocks must immediately follow** their corresponding tool_use blocks. No messages in between.
- In the user message with tool results, **tool_result blocks must come FIRST** in `content`. Any text comes AFTER.

**Wrong (400 error):**
```json
{
  "role": "user",
  "content": [
    { "type": "text", "text": "Here are the results:" },
    { "type": "tool_result", "tool_use_id": "toolu_01" }
  ]
}
```
**Right:**
```json
{
  "role": "user",
  "content": [
    { "type": "tool_result", "tool_use_id": "toolu_01" },
    { "type": "text", "text": "What should I do next?" }
  ]
}
```

### Examples

Successful:
```json
{ "role": "user", "content": [{ "type": "tool_result", "tool_use_id": "toolu_01...", "content": "15 degrees" }] }
```

With images:
```json
{
  "role": "user",
  "content": [{
    "type": "tool_result",
    "tool_use_id": "toolu_01...",
    "content": [
      { "type": "text", "text": "15 degrees" },
      { "type": "image", "source": { "type": "base64", "media_type": "image/jpeg", "data": "..." } }
    ]
  }]
}
```

Empty:
```json
{ "role": "user", "content": [{ "type": "tool_result", "tool_use_id": "toolu_01..." }] }
```

With documents:
```json
{
  "role": "user",
  "content": [{
    "type": "tool_result",
    "tool_use_id": "toolu_01...",
    "content": [
      { "type": "text", "text": "The weather is" },
      { "type": "document", "source": { "type": "text", "media_type": "text/plain", "data": "15 degrees" } }
    ]
  }]
}
```

## Handling results from server tools

Claude executes the tool internally and incorporates results into its response — no additional interaction required.

> **API design note**: Unlike APIs that use special `tool` / `function` roles, the Claude API keeps everything in `user` / `assistant` messages. `user` includes client content and `tool_result`; `assistant` contains AI-generated content and `tool_use`.

## Handling errors with `is_error`

### Tool execution error
```json
{
  "role": "user",
  "content": [{
    "type": "tool_result",
    "tool_use_id": "toolu_01...",
    "content": "ConnectionError: the weather service API is not available (HTTP 500)",
    "is_error": true
  }]
}
```
Claude incorporates the error into its response.

> **Write instructive error messages.** Not `"failed"` — say what went wrong and what Claude should try next: `"Rate limit exceeded. Retry after 60 seconds."`

### Invalid tool name / missing parameters

If Claude's tool call is invalid, return a tool_result with `is_error: true`:
```json
{
  "role": "user",
  "content": [{
    "type": "tool_result",
    "tool_use_id": "toolu_01...",
    "content": "Error: Missing required 'location' parameter",
    "is_error": true
  }]
}
```
Claude will retry 2-3 times with corrections before apologizing.

> **Eliminate invalid calls entirely** with `strict: true` on definitions — guarantees inputs always match the schema.

### Server tool errors

Claude handles server tool errors transparently. You don't need to handle `is_error` results for server tools.

For `web_search`: `too_many_requests`, `invalid_input`, `max_uses_exceeded`, `query_too_long`, `unavailable`.
