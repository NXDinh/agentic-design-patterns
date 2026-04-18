# Tool Runner (SDK)

Source: https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner

Use the SDK's Tool Runner abstraction to handle the agentic loop, error wrapping, and type safety automatically.

---

Tool Runner handles the agentic loop, error wrapping, and type safety so you don't have to. Use the [manual loop](./handle-tool-calls.md) only when you need human-in-the-loop approval, custom logging, or conditional execution.

Available in Python, TypeScript, and Ruby SDKs (beta).

Tool runner automatically:
- Executes tools when Claude calls them
- Handles the request/response cycle
- Manages conversation state
- Provides type safety and validation

> **Automatic context management**: Tool runner supports automatic compaction — generates summaries when token usage exceeds a threshold, letting long agentic tasks continue past context-window limits.

## Basic usage

### Python — `@beta_tool` decorator

```python
import json
from anthropic import Anthropic, beta_tool

client = Anthropic()

@beta_tool
def get_weather(location: str, unit: str = "fahrenheit") -> str:
    """Get the current weather in a given location.

    Args:
        location: The city and state, e.g. San Francisco, CA
        unit: Temperature unit, either 'celsius' or 'fahrenheit'
    """
    return json.dumps({"temperature": "20°C", "condition": "Sunny"})

@beta_tool
def calculate_sum(a: int, b: int) -> str:
    """Add two numbers together."""
    return str(a + b)

runner = client.beta.messages.tool_runner(
    model="claude-opus-4-7",
    max_tokens=1024,
    tools=[get_weather, calculate_sum],
    messages=[{"role": "user", "content": "Weather in Paris? Also 15+27?"}],
)
for message in runner:
    print(message)
```

(For async clients use `@beta_async_tool` + `async def`.)

The decorator inspects arguments + docstring to extract JSON Schema. `calculate_sum` becomes:
```json
{
  "name": "calculate_sum",
  "description": "Add two numbers together.",
  "input_schema": {
    "additionalProperties": false,
    "properties": {
      "a": {"description": "First number", "title": "A", "type": "integer"},
      "b": {"description": "Second number", "title": "B", "type": "integer"}
    },
    "required": ["a", "b"],
    "type": "object"
  }
}
```

### TypeScript — `betaZodTool` or `betaTool`

Using Zod (recommended, requires Zod 3.25+):
```typescript
import Anthropic from "@anthropic-ai/sdk";
import { betaZodTool } from "@anthropic-ai/sdk/helpers/beta/zod";
import { z } from "zod";

const client = new Anthropic();

const getWeatherTool = betaZodTool({
  name: "get_weather",
  description: "Get the current weather in a given location",
  inputSchema: z.object({
    location: z.string().describe("The city and state"),
    unit: z.enum(["celsius", "fahrenheit"]).default("fahrenheit")
  }),
  run: async (input) => JSON.stringify({ temperature: "20°C", condition: "Sunny" })
});

const finalMessage = await client.beta.messages.toolRunner({
  model: "claude-opus-4-7",
  max_tokens: 1024,
  tools: [getWeatherTool],
  messages: [{ role: "user", content: "Weather in Paris?" }]
});
```

Using JSON Schema:
```typescript
import { betaTool } from "@anthropic-ai/sdk/helpers/beta/json-schema";
const calculateSumTool = betaTool({
  name: "calculate_sum",
  description: "Add two numbers together",
  inputSchema: { type: "object", properties: { a: {type:"number"}, b: {type:"number"} }, required: ["a","b"] },
  run: async (input) => String(input.a + input.b)
});
```
(Runtime input is NOT validated with JSON Schema variant — validate in `run` if needed.)

### Ruby — `Anthropic::BaseTool`
```ruby
class GetWeatherInput < Anthropic::BaseModel
  required :location, String
  optional :unit, Anthropic::InputSchema::EnumOf["celsius","fahrenheit"]
end

class GetWeather < Anthropic::BaseTool
  doc "Get the current weather in a given location"
  input_schema GetWeatherInput
  def call(input)
    JSON.generate({temperature: "20°C", condition: "Sunny"})
  end
end

runner = client.beta.messages.tool_runner(
  model: "claude-opus-4-7",
  max_tokens: 1024,
  tools: [GetWeather.new],
  messages: [{role: "user", content: "Weather in Paris?"}]
)
```

Tool functions must return a content block / array (text, images, documents). Strings → text block. Non-string primitives must be stringified. JSON must be encoded to a string.

## Iterating

The runner is an iterable yielding messages. Each iteration checks for a tool use: if present, runs the tool, appends the result, queries Claude again. Loop ends when Claude returns no tool use. Use `break` to exit early.

Get the final message directly:
- Python: `runner.until_done()`
- TypeScript: `await runner` (the runner is awaitable)
- Ruby: `runner.run_until_finished`

## Advanced usage

Within the loop you can modify the next request.

### Python
```python
for message in runner:
    tool_response = runner.generate_tool_call_response()  # inspect
    if tool_response:
        print(f"Tool result: {tool_response}")

    runner.set_messages_params(lambda p: {**p, "max_tokens": 2048})
    runner.append_messages({"role": "user", "content": "Please be concise."})
```

### TypeScript
```typescript
for await (const message of runner) {
  const toolResultMessage = await runner.generateToolResponse();
  runner.setMessagesParams((params) => ({ ...params, max_tokens: 2048 }));
  runner.pushMessages({ role: "user", content: "Please be concise." });
}
```

### Ruby
```ruby
message = runner.next_message
runner.feed_messages([{role: "user", content: "Also check Boston"}])
puts runner.params
```

## Debugging tool execution

When a tool throws, the runner catches it and returns the error to Claude as a tool_result with `is_error: true`. Default: only the exception message.

For full stack traces:
```bash
export ANTHROPIC_LOG=info   # or debug
```

## Intercepting tool errors

```python
for message in runner:
    tool_response = runner.generate_tool_call_response()
    if tool_response:
        for block in tool_response["content"]:
            if block.get("is_error"):
                raise RuntimeError(f"Tool failed: {block['content']}")
```

## Modifying tool results (e.g., cache_control)

To cache large tool outputs, attach `cache_control`:
```python
for message in runner:
    tool_response = runner.generate_tool_call_response()
    if tool_response:
        for block in tool_response["content"]:
            if block["type"] == "tool_result":
                block["cache_control"] = {"type": "ephemeral"}
        runner.append_messages(message, tool_response)  # explicit append prevents auto-append
```

TypeScript: mutate in-place; the runner auto-appends the mutated response. Ruby: access via `runner.params[:messages].last`.

## Streaming

Python:
```python
runner = client.beta.messages.tool_runner(..., stream=True)
for message_stream in runner:
    for event in message_stream:
        print(event)
    print(message_stream.get_final_message())
print(runner.until_done())
```

TypeScript:
```typescript
const runner = client.beta.messages.toolRunner({ ..., stream: true });
for await (const messageStream of runner) {
  for await (const event of messageStream) {
    console.log(event);
  }
  console.log(await messageStream.finalMessage());
}
```

Ruby:
```ruby
runner.each_streaming do |event|
  case event
  when Anthropic::Streaming::TextEvent
    print event.text
  when Anthropic::Streaming::InputJsonEvent
    puts "\nTool input: #{event.partial_json}"
  end
end
```
