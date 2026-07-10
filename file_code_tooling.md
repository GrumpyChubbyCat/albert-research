# rig for file/code tools — plumbing only (phase 2 decision)

**Kind:** research note / decision
**Informs:** [[roadmap]] (phase 2 file workspace, phase 3 forkd)
**Builds on:** [[openclaw_code_fs]]
**Status:** settled (verified against rig-core 0.35.0 local source + docs.rs 0.39.0 +
the rig monorepo `crates/`, 2026-07-11)

Question: does **rig** already have file/code tools — or do we build `octo-code`?

## Finding: rig gives the plumbing, not the tools

- **rig-core ships the tool machinery + exactly one tool.** The `Tool` trait,
  `ToolDyn`, `ToolSet`, `.tool()`, the native tool-loop (`max_turns` — default **0**,
  must be set), MCP wiring (`tool/rmcp.rs`), and a `#[rig_tool]`/`tool_macro`
  attribute (from `rig-derive`) that turns a plain fn into a `Tool`. The whole
  `tools` module is one built-in: **`ThinkTool`** (a no-I/O scratchpad). No
  read/write/edit/list/bash/exec anywhere. `loaders/` are RAG *document* loaders,
  not agent actions.
- **You implement every tool's `call` body; rig never touches the FS/shell.**
- **No typed computer-use.** The Anthropic provider `ToolDefinition` has no `type`
  field, so `bash_2025*` / `text_editor_*` / `computer_*` server-defined tools can't
  go through the typed `.tool()` path. Escape hatch only: `client.anthropic_beta(..)`
  + raw `additional_params.tools` passthrough — rig won't type/validate/run it. And
  it's moot for us: we're on **OpenRouter/DeepSeek**, not Anthropic-direct; OpenAI's
  hosted `computer_use`/`file_search` (Responses API) is OpenAI-side + Operator-style
  and equally N/A.
- **The `crates/` monorepo audit** (19 crates): `rig-core` (machinery), `rig-derive`
  (macros), `rig-memory` (conversation-history filtering — we have octo-history +
  kaeru), and **16 vector-store/embeddings integrations** (bedrock, fastembed,
  lancedb, milvus, mongodb, neo4j, postgres, qdrant, s3vectors, scylladb, sqlite,
  surrealdb, vertexai, gemini-grpc, vectorize). Rig's weight is **RAG + tool-calling**;
  zero agent action-tools.

So the intuition "rig has everything for computer-use" is **half true**: it has
everything to *wire* tools (we already do — kaeru/scratchpad/skills), nothing that
*is* a file/code tool. Computer-use = our code + our safety.

## External references (mine, don't blind-adopt)

- **`llm-coding-tools-rig`** (community crate) — ready rig `Tool` impls: `ReadTool`,
  `WriteTool`, `EditTool`, `GlobTool`, `GrepTool`, `BashTool`, `WebFetchTool`,
  `TodoRead/WriteTool`, with sandboxed/unrestricted modes (`.tool(ReadTool::<true>::
  new())`). **Targets rig-core ^0.28** — compat with our 0.35 unverified; its
  "sandbox" unaudited. A strong *reference*, a risky *dependency*.
- **`rust-bash`** — a sandboxed bash interpreter over a virtual FS; a possible
  backend reference for phase-3 exec.
- **"Rewriting Claude Code in Rust"** (0xPlaygrounds/joshmo, DEV.to) — defines
  `ReadFile`/`WriteFile`/`Bash` as rig `Tool`s over `std::fs` + `tokio::process`. The
  canonical "define your own, rig runs the loop" pattern — exactly our path.

## Decision: build `octo-code` (rig-tools crate), informed by the above

- **A crate of rig `Tool` impls, sibling to `octo-rig`** — not a bus connector.
  File/code are **synchronous, local faculties** the agent invokes (like
  scratchpad/kaeru), not async external organs that push events (telegram/calendar/
  scheduler). A connector would add a bus round-trip per file op for no gain. Install
  it on the agent via `.tool(..)` like `kaeru_rig`/`octo_rig`.
- **Own the safety layer.** Albert runs as **root on the VM** with no docker to lean
  on → the workspace-jail + path-safety (from [[openclaw_code_fs]]: reject
  `..`/absolute, atomic `O_EXCL`/`O_NOFOLLOW`/`0o600`, one workspace root) is
  *the* safety for phase 2 and must be ours, audited — not a third-party crate's
  unverified "sandbox mode."
- **Phase 2 = files only** (`read`/`write`/`edit` string-replace/`list`, workspace-
  jailed). **Phase 3 = exec** (`bash`) behind the real sandbox (forkd: bubblewrap/
  landlock or WASM) — possibly the one place an organ/connector fits, since the
  sandbox is a heavy supervised substrate.
- **Lean: B (build) informed by A (mine `llm-coding-tools-rig` + joshmo as
  starting points).** Blind-adopting a third-party FS/exec layer for a root agent is
  the one place not to outsource. Cheap first signal worth taking: check whether
  `llm-coding-tools-rig` even compiles against rig 0.35 — if it does, it's a faster
  reference to read.

## Caveat

`llm-coding-tools-rig` 0.35 compatibility and the realness of its "sandbox" are
**unverified**; treat as reference until checked.
