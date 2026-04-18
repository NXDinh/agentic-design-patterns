# How tool use works

Source: https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works

Understand the tool-use loop, where tools execute, and when to use tools instead of prose.

---

## The tool-use contract

Tool use is a contract between your application and the model. You specify what operations are available and what shape their inputs/outputs take; Claude decides when and how to call them. The model never executes anything on its own. It emits a structured request, your code (or Anthropic's servers) runs the operation, and the result flows back into the conversation.

This contract makes the model behave less like a text generator and more like a function you call. Define the schema, handle the callback, return a result. The difference vs. a classical API is that the caller is a language model choosing which function to invoke based on the conversation.

## Where tools run

### User-defined tools (client-executed)
You write the schema, you execute the code, you return the results. Vast majority of tool-use traffic.

When Claude decides to use one of your tools, the response contains a `tool_use` block with tool name + JSON args. Your app extracts args, runs the operation, sends the output back in a `tool_result` block on the next request. Claude never sees your implementation — only the schema you provided and the result you returned.

### Anthropic-schema tools (client-executed)
For common operations (shell commands, editing files, browser control, scratchpad memory), Anthropic publishes the tool schema and your application handles execution. These are: `bash`, `text_editor`, `computer`, `memory`.

Execution model = user-defined tools. The reason to use Anthropic-schema tools: **the schemas are trained-in**. Claude has been optimized on thousands of successful trajectories with these exact signatures, so it calls them more reliably and recovers from errors more gracefully than it would with a custom equivalent.

### Server-executed tools
For `web_search`, `web_fetch`, `code_execution`, and `tool_search`, Anthropic runs the code. You enable the tool; the server handles everything else. You never construct a `tool_result` for these. The response contains `server_tool_use` blocks showing what ran, but execution is already complete by the time you see them.

## The agentic loop (client tools)

Client-executed tools require your application to drive a loop. Canonical shape — a `while` loop keyed on `stop_reason`:

1. Send a request with your `tools` array and the user message.
2. Claude responds with `stop_reason: "tool_use"` and one or more `tool_use` blocks.
3. Execute each tool. Format outputs as `tool_result` blocks.
4. Send a new request containing the original messages, the assistant's response, and a user message with the `tool_result` blocks.
5. Repeat from step 2 while `stop_reason` is `"tool_use"`.

Loop exits on any other stop reason: `"end_turn"`, `"max_tokens"`, `"stop_sequence"`, or `"refusal"`.

For the SDK abstraction that handles this automatically, see `tool-runner.md`. For request mechanics, parallel calls, and result formatting, see `handle-tool-calls.md`.

## The server-side loop

Server-executed tools run their own loop inside Anthropic's infrastructure. A single request might trigger several web searches or code executions before a response comes back. The model searches, reads results, decides to search again, iterates until it has what it needs — your app doesn't participate.

Internal loop has an iteration limit. If the model is still iterating at the cap, the response comes back with `stop_reason: "pause_turn"` instead of `"end_turn"`. A paused turn means the work isn't finished; re-send the conversation (including the paused response) to let the model continue.

## When to use tools (and when not to)

**Tool use fits when:**
- **Actions with side effects.** Sending email, writing a file, updating a record.
- **Fresh or external data.** Current prices, weather, DB contents.
- **Structured, guaranteed-shape outputs.** Need a JSON object with specific fields rather than prose containing the info.
- **Calling into existing systems.** DBs, internal APIs, file systems.

The tell: **if you're writing a regex to extract a decision from model output, that decision should have been a tool call**. Parsing free-form text to recover structured intent is a sign the structure belongs in the schema.

**Tool use doesn't fit when:**
- The model can answer from training alone (summarization, translation, general-knowledge).
- One-shot Q&A with no side effects.
- Tool-calling latency would dominate a trivial response.

## Choosing between approaches

| Approach | When | What to expect |
|---|---|---|
| User-defined client tools | Custom business logic, internal APIs, proprietary data | You handle execution and the agentic loop |
| Anthropic-schema client tools | Standard dev ops (bash, file editing, browser control) | You execute; Claude calls reliably (schema trained-in) |
| Server-executed tools | Web search, code sandbox, web fetch | Anthropic handles execution; you get results directly |
