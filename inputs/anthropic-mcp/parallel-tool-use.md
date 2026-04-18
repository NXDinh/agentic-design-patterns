# Parallel tool use

Source: https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use

Enable and format parallel tool calls, with message-history guidance and troubleshooting.

---

By default, Claude may use **multiple tools in one turn**. Disable with:
- `disable_parallel_tool_use=true` + `tool_choice: auto` → Claude uses **at most one** tool
- `disable_parallel_tool_use=true` + `tool_choice: any` or `tool` → Claude uses **exactly one** tool

## Worked example

```python
tools = [
    {
        "name": "get_weather",
        "description": "Get the current weather in a given location",
        "input_schema": {
            "type": "object",
            "properties": {"location": {"type": "string"}},
            "required": ["location"],
        },
    },
    {
        "name": "get_time",
        "description": "Get the current time in a given timezone",
        "input_schema": {
            "type": "object",
            "properties": {"timezone": {"type": "string"}},
            "required": ["timezone"],
        },
    },
]

response = client.messages.create(
    model="claude-opus-4-7", max_tokens=1024,
    messages=[{"role": "user", "content": "What's the weather in SF and NYC, and what time is it there?"}],
    tools=tools,
)

tool_uses = [b for b in response.content if b.type == "tool_use"]
print(f"Claude made {len(tool_uses)} tool calls")

# Execute all tools, format results correctly
tool_results = []
for tool_use in tool_uses:
    result = "..."  # Actually run the tool
    tool_results.append({"type": "tool_result", "tool_use_id": tool_use.id, "content": result})

# Continue — ALL RESULTS IN A SINGLE USER MESSAGE
messages = [
    {"role": "user", "content": "What's the weather in SF and NYC, and what time is it there?"},
    {"role": "assistant", "content": response.content},
    {"role": "user", "content": tool_results},  # single message with all results
]
```

## Maximizing parallel tool use

### System prompts (Claude 4 models)

Basic:
```text
For maximum efficiency, whenever you need to perform multiple independent operations, invoke all relevant tools simultaneously rather than sequentially.
```

Stronger (if default isn't enough):
```text
<use_parallel_tool_calls>
For maximum efficiency, whenever you perform multiple independent operations, invoke all relevant tools simultaneously rather than sequentially. Prioritize calling tools in parallel whenever possible. For example, when reading 3 files, run 3 tool calls in parallel to read all 3 files into context at the same time. When running multiple read-only commands like `ls` or `list_dir`, always run all of the commands in parallel. Err on the side of maximizing parallel tool calls rather than running too many tools sequentially.
</use_parallel_tool_calls>
```

### User-message prompting
- Instead of: "What's the weather in Paris? Also check London."
- Use: "Check the weather in Paris and London simultaneously."
- Explicit: "Please use parallel tool calls to get the weather for Paris, London, and Tokyo at the same time."

## Troubleshooting

### 1. Incorrect tool result formatting (most common)

Sending separate user messages for each tool result **teaches Claude to avoid parallelism**.

**Wrong — reduces parallel tool use:**
```json
[
  {"role": "assistant", "content": [tool_use_1, tool_use_2]},
  {"role": "user", "content": [tool_result_1]},
  {"role": "user", "content": [tool_result_2]}
]
```

**Right — all results in a single user message:**
```json
[
  {"role": "assistant", "content": [tool_use_1, tool_use_2]},
  {"role": "user", "content": [tool_result_1, tool_result_2]}
]
```

### 2. Weak prompting
Use the stronger `<use_parallel_tool_calls>` system prompt.

### 3. Measuring
```python
tool_call_messages = [m for m in messages if any(b.type == "tool_use" for b in m.content)]
total_tool_calls = sum(len([b for b in m.content if b.type == "tool_use"]) for m in tool_call_messages)
avg_tools_per_message = total_tool_calls / len(tool_call_messages) if tool_call_messages else 0.0
# Should be > 1.0 if parallel calls are working
```

## Related
- **Single-tool flow + formatting rules** — `handle-tool-calls.md`
- **SDK abstraction that handles parallel automatically** — `tool-runner.md`
