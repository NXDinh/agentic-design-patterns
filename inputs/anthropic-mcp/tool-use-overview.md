# Tool use with Claude

Source: https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview

Connect Claude to external tools and APIs. Learn where tools execute and how the agentic loop works.

---

Tool use lets Claude call functions you define or that Anthropic provides. Claude decides when to call a tool based on the user's request and the tool's description, then returns a structured call that your application executes (client tools) or that Anthropic executes (server tools).

### Simplest example — server tool (Anthropic handles execution)

```bash
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-opus-4-7",
    "max_tokens": 1024,
    "tools": [{"type": "web_search_20260209", "name": "web_search"}],
    "messages": [{"role": "user", "content": "What'"'"'s the latest on the Mars rover?"}]
  }'
```

```python
import anthropic
client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    tools=[{"type": "web_search_20260209", "name": "web_search"}],
    messages=[{"role": "user", "content": "What's the latest on the Mars rover?"}],
)
```

## How tool use works

Tools differ primarily by where the code executes. **Client tools** (including user-defined tools and Anthropic-schema tools like `bash` and `text_editor`) run in your application: Claude responds with `stop_reason: "tool_use"` and one or more `tool_use` blocks, your code executes the operation, and you send back a `tool_result`. **Server tools** (`web_search`, `code_execution`, `web_fetch`, `tool_search`) run on Anthropic's infrastructure: you see the results directly without handling execution.

For the full conceptual model including the agentic loop and when to choose each approach, see `how-tool-use-works.md`.

For connecting to MCP servers, see `mcp-connector.md`.

> **Strict tool use**: Add `strict: true` to your tool definitions to ensure Claude's tool calls always match your schema exactly.

Tool access is one of the highest-leverage primitives you can give an agent. On benchmarks like LAB-Bench FigQA and SWE-bench, adding even basic tools produces outsized capability gains, often surpassing human expert baselines.

## What happens when Claude needs more information

If the prompt doesn't include enough information for required parameters, Claude Opus is much more likely to recognize the missing parameter and ask for it. Claude Sonnet may ask, especially when prompted to think first, but may also infer a reasonable value. Example: given a `get_weather` tool requiring `location`, if you ask "What's the weather?" without a location, Sonnet might guess "New York, NY".

## Pricing

Tool use requests are priced based on:
1. Total input tokens (including `tools` parameter)
2. Output tokens
3. For server-side tools, additional usage-based pricing

When you use `tools`, a special system prompt is automatically included. Tool-use system prompt token counts (Claude 4.x family):

| Model | `auto`, `none` | `any`, `tool` |
|---|---|---|
| Claude Opus 4.7 / 4.6 / 4.5 / 4.1 / 4 | 346 | 313 |
| Claude Sonnet 4.6 / 4.5 / 4 | 346 | 313 |
| Claude Haiku 4.5 | 346 | 313 |
| Claude Haiku 3.5 | 264 | 340 |

## Next steps
- `how-tool-use-works.md` — concepts: where tools run, the loop, when to use.
- `define-tools.md` — schema, descriptions, `tool_choice`.
- `handle-tool-calls.md` — parsing `tool_use`, formatting `tool_result`.
- `tool-runner.md` — SDK abstraction that handles the loop automatically.
- `tool-reference.md` — catalog of Anthropic-provided tools and optional properties.
