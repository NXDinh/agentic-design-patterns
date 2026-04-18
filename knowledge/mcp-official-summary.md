# MCP Official Docs — Summary

Source: `inputs/mcp-official/` (shallow clone of [`modelcontextprotocol/modelcontextprotocol`](https://github.com/modelcontextprotocol/modelcontextprotocol)).
Latest spec revision in this snapshot: **2025-11-25**. Full history: 2024-11-05 → 2025-03-26 → 2025-06-18 → **2025-11-25** (+ `draft/`).

## What's in the input folder
```
inputs/mcp-official/
├── README.md, AGENTS.md, GOVERNANCE.md, SECURITY.md, CONTRIBUTING.md, LICENSE
├── docs/                           # Mintlify docs site source
│   ├── docs/
│   │   ├── getting-started/intro.mdx
│   │   ├── learn/                  # architecture, server-concepts, client-concepts, versioning
│   │   ├── develop/                # build-client, build-server, connect-local-servers, connect-remote-servers, build-with-agent-skills
│   │   ├── tools/                  # inspector, debugging
│   │   ├── tutorials/security/     # authorization, security_best_practices
│   │   └── sdk.mdx
│   ├── specification/              # normative spec, per-revision
│   │   ├── 2024-11-05/ 2025-03-26/ 2025-06-18/ 2025-11-25/ draft/
│   │   └── <version>/{architecture,basic,client,server,schema.mdx,changelog.mdx}
│   ├── community/                  # governance, design-principles, SEP process
│   ├── development/roadmap.mdx
│   ├── extensions/                 # apps, auth (OAuth, enterprise IdP)
│   ├── registry/                   # MCP server registry
│   ├── seps/                       # accepted Server Enhancement Proposals
│   └── images/                     # images referenced by docs (Mintlify-absolute paths)
└── schema/                         # protocol schema per revision
    ├── 2024-11-05/ 2025-03-26/ 2025-06-18/ 2025-11-25/ draft/
    └── <version>/{schema.ts, schema.json, schema.mdx}
```
The schema is authoritative TypeScript; JSON Schema is auto-generated for compatibility.

## Mental model (from `docs/docs/learn/architecture.mdx`)
- **Host**: the AI app (Claude Code, Claude Desktop, VS Code, an IDE).
- **Client**: one per connection; managed by the host; maintains a dedicated session with one server.
- **Server**: exposes capabilities; local (stdio) or remote (HTTP).
- **Two layers**:
  - **Data layer** — JSON-RPC 2.0: lifecycle, primitives, notifications.
  - **Transport layer** — stdio or Streamable HTTP (+ auth).

Stateful protocol. Every session begins with `initialize` (capability negotiation) → `notifications/initialized`.

## Primitives

### Server → Client (what servers offer)
| Primitive | Controlled by | Purpose | Discovery | Invocation |
|---|---|---|---|---|
| **Tools** | Model | Actions/effects (queries, writes, APIs) | `tools/list` | `tools/call` |
| **Resources** | Application | Read-only context data (files, records, URIs) | `resources/list`, `resources/templates/list` | `resources/read`, `resources/subscribe` |
| **Prompts** | User | Templated workflows (slash-commands, forms) | `prompts/list` | `prompts/get` |

Resource URIs can be **direct** (`calendar://events/2024`) or **templated** (`travel://activities/{city}/{category}`) with parameter completion.

### Client → Server (what clients offer back)
| Primitive | Purpose | Example |
|---|---|---|
| **Sampling** | Server asks client's LLM for a completion | Tool analyzes 47 flight options by requesting a sampling call. |
| **Elicitation** | Server asks the user for structured input mid-task | Confirm booking details; `elicitation/requestInput` with a JSON Schema. |
| **Roots** | Client communicates filesystem scope (advisory) | `file:///Users/me/project` — not a security boundary; enforce with OS permissions. |
| **Logging** | Server sends log events to client | Debug/observability. |

### Cross-cutting utility (experimental, 2025-11-25)
- **Tasks** — durable execution wrappers: submit, poll, retrieve deferred results. For long-running jobs, batch, workflow automation.

Other utilities: cancellation, ping, progress, completion, pagination, logging.

## Transports
- **Stdio** — JSON-RPC over stdin/stdout. Local only. One client ↔ one server process.
- **Streamable HTTP** — HTTP POST + optional Server-Sent Events for streaming. Remote. Standard HTTP auth (bearer, API key, custom headers). **OAuth recommended** for token issuance.

Spec clarifies: invalid Origin on Streamable HTTP ⇒ HTTP 403. Stderr on stdio may carry all logs.

## Authorization (spec `basic/authorization.mdx`)
- OAuth 2.0 framework. Latest (2025-11-25) adds:
  - **OpenID Connect Discovery 1.0** support for authorization server discovery.
  - **OAuth 2.0 Protected Resource Metadata** (RFC 9728) — `WWW-Authenticate` optional, `.well-known` fallback.
  - **OAuth Client ID Metadata Documents** — recommended client registration mechanism.
  - **Incremental scope consent** via `WWW-Authenticate`.
- Best practices: `docs/docs/tutorials/security/security_best_practices.mdx` and `authorization.mdx`.

## Lifecycle at a glance
```
Client                              Server
  │  initialize(protocolVersion,      │
  │              capabilities,        │
  │              clientInfo)          │
  │ ────────────────────────────────▶ │
  │                                   │
  │ ◀──────  result(capabilities,     │
  │                  serverInfo)      │
  │                                   │
  │ notifications/initialized ──────▶ │
  │                                   │
  │ tools/list ─────────────────────▶ │
  │ ◀──── tool definitions            │
  │                                   │
  │ tools/call {name, arguments} ───▶ │
  │ ◀──── result.content[]            │
  │                                   │
  │ ◀──── notifications/tools/list_changed (if supported)
```
Capability-gated: a server only sends `list_changed` if it declared `{"tools": {"listChanged": true}}`.

## Security principles (from spec index)
1. **User consent & control** — explicit for every data access / tool call.
2. **Data privacy** — host needs consent before sharing user data with servers; no retransmission without consent.
3. **Tool safety** — treat tools as arbitrary code execution; tool descriptions/annotations are **untrusted** unless from a trusted server.
4. **LLM sampling controls** — user approves every sampling request; controls whether it happens, the exact prompt, and what the server sees. Protocol deliberately limits server visibility into prompts.

MCP cannot enforce these at the wire level — implementers **SHOULD** build consent UIs, access controls, and clear docs.

## Key changes in 2025-11-25 (from `docs/specification/2025-11-25/changelog.mdx`)
**Major**
1. OpenID Connect Discovery 1.0 for authorization server discovery.
2. Icons on tools/resources/resource-templates/prompts (metadata).
3. Incremental scope consent via `WWW-Authenticate`.
4. Tool naming guidance (SEP-986).
5. `ElicitResult` / `EnumSchema` — titled/untitled, single/multi-select enums.
6. URL-mode elicitation (SEP-1036).
7. Sampling supports `tools` + `toolChoice` (SEP-1577).
8. OAuth Client ID Metadata Documents for registration.
9. **Tasks** (experimental) — durable, pollable deferred execution.

**Minor**
- Input validation errors → return as tool execution errors (not protocol errors) so the model can self-correct.
- JSON Schema 2020-12 is the default dialect.
- Streamable HTTP: 403 on invalid Origin; SSE polling/resumption clarified.
- stdio: stderr for all log levels.

## Ecosystem components (from `docs/docs/learn/architecture.mdx` § Scope)
- **Spec**: normative protocol requirements.
- **SDKs**: Python, TypeScript, Kotlin, Java, C#, Swift, PHP, Rust, Ruby, Go (tiering system per SEP-1730).
- **Dev tools**: [MCP Inspector](https://github.com/modelcontextprotocol/inspector) — debugging tool.
- **Reference servers**: [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) — filesystem, git, GitHub, Slack, Postgres, etc.
- **Registry** (preview) — discover/publish servers.
- **Apps / Extensions** (in progress, 2025-11/2026): richer UIs served by MCP servers (SEP-1865 MCP Apps, SEP-2133 Extensions).

## Practical how-tos in the cloned docs
- Build a server: `inputs/mcp-official/docs/docs/develop/build-server.mdx`
- Build a client: `inputs/mcp-official/docs/docs/develop/build-client.mdx`
- Connect local/remote: `inputs/mcp-official/docs/docs/develop/connect-{local,remote}-servers.mdx`
- Debug & inspect: `inputs/mcp-official/docs/docs/tools/{debugging,inspector}.mdx`
- Authorization tutorial: `inputs/mcp-official/docs/docs/tutorials/security/authorization.mdx`
- Security best practices: `inputs/mcp-official/docs/docs/tutorials/security/security_best_practices.mdx`
- Design principles (what MCP is and isn't): `inputs/mcp-official/docs/community/design-principles.mdx`
- Full schema (types): `inputs/mcp-official/schema/2025-11-25/schema.ts`

## When the packed note `knowledge/mcp.md` isn't enough
Use this summary to navigate to the official doc. Rough map:
- **Naming a tool** → tool-naming guidance SEP-986 + `server-concepts.mdx` + changelog 2025-11-25.
- **Designing schemas** → `schema/2025-11-25/schema.ts` + JSON Schema 2020-12.
- **Handling auth** → `basic/authorization.mdx` (per version) + `tutorials/security/authorization.mdx`.
- **Writing a remote server** → `docs/docs/develop/build-server.mdx` + `basic/transports.mdx`.
- **Durable/async jobs** → `basic/utilities/tasks` (2025-11-25+ only).
- **UI surfaces for prompts/resources** → `server-concepts.mdx` user-interaction sections.

## Versioning note
Always verify which spec revision a client/server targets — capability objects and some message shapes change across 2024-11-05 / 2025-03-26 / 2025-06-18 / 2025-11-25. The `protocolVersion` field in `initialize` is the source of truth per session.
