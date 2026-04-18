# Tool Use (Function Calling) — Reference

Source: Ch 5, Agentic Design Patterns (Gulli/Sauco).

## TL;DR
An LLM decides when to call an external function, emits a structured call (JSON), the runtime executes it, and the result is fed back. Use when the model needs real-time data, calculations, actions, or access to private systems.

## When to use
- Real-time/external data (weather, prices, inventory).
- Private/proprietary data (internal DB, company docs).
- Deterministic computation the model is unreliable at (math, code execution).
- Side effects (send email, update record, control device).

## The 6-step loop
1. **Define** — name, purpose, typed params, docstring.
2. **LLM decides** — given tools + request, picks one (or none).
3. **Call generated** — structured JSON `{tool, args}`.
4. **Execute** — runtime calls the function, captures output or error.
5. **Observe** — result returned to the LLM.
6. **Process** — LLM uses result to answer or call the next tool.

## Tool definition — what makes a good one
- **Clear name + docstring**: the docstring IS the spec the LLM reads. Start with the purpose, then params, then return shape. Include example triggers ("use this when the user asks X").
- **Typed params** with descriptions. Prefer primitives + small enums over free-form strings.
- **Return clean data, not strings**: raw floats, dicts, or raise specific exceptions. Let the agent layer format the message. (CrewAI example, Ch 5: `get_stock_price` returns `float` or raises `ValueError`.)
- **Single responsibility**: one tool = one verb. Don't overload.
- **Deterministic helpers on top of APIs**: if the underlying API returns a list of 10k items, wrap it with filter/sort/pagination before exposing — agents are bad at scanning long results.

## "Tool" is broader than "function"
A tool can be: a Python function, an API call, a DB query, a code interpreter, a search index, or a call to **another agent**. Design the mental model as "tool calling" so delegation patterns (specialist sub-agent as a tool) fall out naturally.

## Common tool categories
| Category | Example | Notes |
|---|---|---|
| Retrieval | web search, vector search, DB query | Often the first tool you need. See `rag.md`. |
| Compute | code interpreter, calculator | Prefer deterministic execution over LLM arithmetic. |
| Action | send email, create ticket, commit code | Gate behind confirmation/guardrails. |
| Delegation | call sub-agent / sub-skill | Treat the sub-agent as just another tool. |
| Control | IoT, browser automation | Highest blast radius — log everything. |

## Pitfalls & mitigations
- **LLM invents params** → tighten type hints and add `enum` / constrained values; validate server-side.
- **Silent failures** → raise typed errors; return error shape with `{error_code, message, retry_hint}`.
- **Tool sprawl** → group related tools behind an MCP server (see `mcp.md`); use a `tool_filter` to expose only what's needed per task.
- **Long outputs blow the context** → paginate, summarize, or return IDs the agent can fetch on demand.
- **Non-idempotent actions** → require a confirmation token arg or run in a dry-run mode first.

## Framework cheat sheet
- **LangChain**: `@tool` decorator → bind to model via `create_tool_calling_agent` → run with `AgentExecutor`.
- **CrewAI**: `@tool("Name")` → attach to `Agent(tools=[...])`.
- **Google ADK**: `LlmAgent(tools=[google_search, BuiltInCodeExecutor(), ...])`. Prebuilt: `google_search`, `built_in_code_execution`, `VSearchAgent` (Vertex AI Search).
- **Vertex Extensions**: managed, auto-executed tool wrappers with enterprise auth — contrast with function calling where the client executes.
- **Anthropic/Claude**: tool_use blocks with JSONSchema input; same loop shape.

## Minimal pattern (language-agnostic pseudocode)
```
def my_tool(arg1: str, arg2: int) -> dict:
    """One-line purpose. When to call. Param semantics."""
    result = do_work(arg1, arg2)         # real work
    return {"ok": True, "data": result}  # structured, parseable
```

## Checklist before shipping a tool
- [ ] Docstring explains purpose + trigger conditions.
- [ ] Params are typed; enums used where possible.
- [ ] Returns structured data or raises typed errors.
- [ ] Handles empty / not-found / rate-limit cases explicitly.
- [ ] Idempotent OR requires explicit confirmation.
- [ ] Logged (inputs, outputs, duration, errors).
- [ ] Has at least one unit test for the happy path and one for failure.
