# Agentic Patterns — Packed Reference

Distilled, action-oriented notes from *Agentic Design Patterns* (Gulli & Sauco), packed so Claude can consume them efficiently in future sessions.

## Files
- [`tool-use.md`](./tool-use.md) — Ch 5. When/how to expose a function to an LLM. Tool-design checklist.
- [`mcp.md`](./mcp.md) — Ch 10. Model Context Protocol: client-server architecture, primitives, design rules.
- [`rag.md`](./rag.md) — Ch 14. Retrieval-Augmented Generation: pipeline, strategies, Agentic RAG, GraphRAG, pitfalls.

## How to use with Claude Code
Two usage modes:

### Mode A — On-demand (recommended)
Keep this folder in the repo. Add pointers in the project's `CLAUDE.md` so Claude knows when to read each file. Context stays lean; Claude loads only what's relevant.

Example `CLAUDE.md` snippet:
```
## Agent design references
- When designing a new tool or function-calling interface → read `knowledge/tool-use.md`.
- When building or connecting to an MCP server → read `knowledge/mcp.md`.
- When designing retrieval over docs (RAG, vector search, citations) → read `knowledge/rag.md`.
```

### Mode B — Always-loaded
If the project is agent-heavy, inline the files with `@`-imports in `CLAUDE.md`:
```
@knowledge/tool-use.md
@knowledge/mcp.md
@knowledge/rag.md
```
Trade-off: more context cost every turn.

## Packing principles (why these files are short)
- **Decision rules first** — when to use, when not to.
- **Tables over prose** — compare options at a glance.
- **Checklists, not paragraphs** — actionable before shipping.
- **Skeleton code** — patterns, not full runnable demos (the book has those).
- **Pitfalls + fixes** — what fails in production and how to address.

Extend the folder with additional chapters as needed, following the same shape:
`TL;DR → When to use → Core concepts → Pattern → Pitfalls → Checklist`.
