# Project Memory

Knowledge base for agentic-design work: the *Agentic Design Patterns* book (Gulli & Sauco) plus the official MCP spec + docs.

## Layout
- `inputs/` — raw source documents, one subfolder per source (`book/`, `mcp-official/`, ...).
- `assets/` — photos/images, grouped by source (`book/`, `mcp-official/`, ...).
- `knowledge/` — packed/distilled references optimized for Claude.
- See `inputs/README.md` for the convention to add new sources.

## Packed references — read on demand
When the task matches, open the packed note first; fall back to `inputs/...` for the full source.

| When the task is about... | Read first | Deep source |
|---|---|---|
| Designing a tool / function-calling interface | `knowledge/tool-use.md` | `inputs/book/01-Part_One/Chapter_5-*.md` |
| Building or integrating an MCP server/client | `knowledge/mcp.md`, `knowledge/mcp-official-summary.md` | `inputs/mcp-official/docs/` + `inputs/mcp-official/schema/2025-11-25/` |
| RAG, vector search, or retrieval over docs | `knowledge/rag.md` | `inputs/book/03-Part_Three/Chapter_14-*.md` |
| Any other agentic pattern (routing, memory, guardrails, eval, ...) | — | `inputs/book/0{1-4}-Part_*/Chapter_*.md` + `inputs/book/05-Appendix/` |

## Notes
- Latest MCP spec revision in `inputs/mcp-official/`: **2025-11-25**. The `schema/2025-11-25/schema.ts` is authoritative.
- Book image references use `../../../assets/book/...` after the reorg.
