# inputs/ — Source documents

Raw, unedited source material we reference. Each source lives in its own subfolder. Photos/images that belong to a source live under `../assets/<source-name>/` so the top-level layout stays: **one folder for text inputs, one for photos**.

## Current sources
| Folder | What | Origin |
|---|---|---|
| `book/` | *Agentic Design Patterns* by Gulli & Sauco — full text | Original repo (unchanged chapter content) |
| `mcp-official/` | Model Context Protocol spec + docs + schema | Shallow clone of [modelcontextprotocol/modelcontextprotocol](https://github.com/modelcontextprotocol/modelcontextprotocol) |

## Adding a new source — convention
1. Create `inputs/<slug>/` — put text docs, code, spec files here.
2. Create `assets/<slug>/` — put all its images/photos here.
3. Update `knowledge/` with a packed summary (`knowledge/<slug>-summary.md` or similar) and/or an action-oriented reference.
4. Add a row to the table above and a pointer in the root `CLAUDE.md` so Claude knows when to consult it.

## Why this layout
- **Text vs photos separated** → easy to scan, easy to add, easy to archive.
- **Per-source subfolders** → no naming collisions, clean boundaries.
- **`knowledge/` holds derivatives** → sources stay pristine; distillations are versioned separately.
