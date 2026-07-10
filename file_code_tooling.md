# rig for file/code tools — plumbing only; build octo-code (phase 2)

**Kind:** research note / decision
**Informs:** [[roadmap]] (phase 2 file workspace, phase 3 forkd)
**Builds on:** [[openclaw_code_fs]]
**Status:** settled (verified against rig-core 0.35.0 local source + docs.rs 0.39.0 +
the rig monorepo `crates/` + a resolver spike, 2026-07-11)

Question: does **rig** already have file/code tools — or do we build `octo-code`?

## Finding: rig gives the plumbing, not the tools

- **rig-core ships the tool machinery + exactly one tool.** The `Tool` trait,
  `ToolDyn`, `ToolSet`, `.tool()`, the native tool-loop (`max_turns` — default **0**,
  must be set), MCP wiring (`tool/rmcp.rs`), and the `rig_tool`/`tool_macro` attribute
  (from `rig-derive`). The whole `tools` module is one built-in: **`ThinkTool`** (a
  no-I/O scratchpad). No read/write/edit/list/bash/exec anywhere. `loaders/` are RAG
  *document* loaders, not agent actions.
- **You implement every tool's `call` body; rig never touches the FS/shell.**
- **No typed computer-use.** The Anthropic provider `ToolDefinition` has no `type`
  field, so `bash_2025*` / `text_editor_*` / `computer_*` server-defined tools can't
  go through the typed `.tool()` path (escape hatch only: `anthropic_beta` + raw
  `additional_params.tools`, unrun/untyped). Moot for us anyway: OpenRouter/DeepSeek,
  not Anthropic-direct; OpenAI hosted computer-use is OpenAI-side + Operator-style.
- **`crates/` monorepo audit** (19 crates): `rig-core` (machinery), `rig-derive`
  (macros), `rig-memory` (conversation-history filtering — we have octo-history +
  kaeru), and **16 vector-store/embeddings integrations**. Rig's weight is
  **RAG + tool-calling**; zero agent action-tools.

So "rig has everything for computer-use" is **half true**: everything to *wire* tools
(we already do — kaeru/scratchpad/skills), nothing that *is* a file/code tool.
Computer-use = our code + our safety.

## References — adapt, don't depend

- **`llm-coding-tools-rig`** v0.1.0 (Apache-2.0) — ready rig `Tool` impls: `ReadTool`,
  `WriteTool`, `EditTool`, `GlobTool`, `GrepTool`, `BashTool`, `WebFetchTool`,
  `TodoRead/WriteTool` (sandboxed/unrestricted modes). **Pins `rig-core ^0.28`** → a
  resolver spike confirmed it pulls a *second* rig-core (0.28 beside our 0.35); its
  0.28 `Tool` type can't install on our 0.35 agent. **Not a dependency** — but under
  Apache-2.0 it stands as a **permissively-licensed reference implementation** of
  exactly the tool surface we need. The plan: **port it to rig 0.35** (a version it
  predates) and reshape it to our conventions and safety model, with attribution.
  Deriving from a proven reference is sounder than starting from a blank page.
- **`rig_tool` / `tool_macro`** (rig-derive, re-exported by rig-core) — attribute
  macro turning a **free function** into a `Tool`. **Stateless only** (unit-like
  struct; cannot capture `Arc`/handles). Use it for the **new stateless coding tools**
  (workspace root from const/env, not per-instance state); keep **trait impls for
  stateful tools** (scratchpad/skills/kaeru/dispatch hold store handles).
- **`rust-bash`** — a sandboxed bash interpreter over a virtual FS; a backend
  reference for a later exec tool.
- **`maki`** (tontinton/maki) — a token-efficient Rust coding agent (code index, bash
  permission system, multi-agent delegation; `code_execution` = an in-process
  time/mem-limited interpreter sandbox, not containers/WASM). Like OpenClaw: a whole
  agent to mine for patterns, not a library to adopt.
- **"Rewriting Claude Code in Rust"** (0xPlaygrounds/joshmo, DEV.to) — the canonical
  `ReadFile`/`WriteFile`/`Bash` rig-`Tool` pattern over `std::fs`/`tokio::process`.

## Decision (2026-07-11)

- **`octo-code` = its own crate** in the octo workspace, wired into `octo-rig` behind
  a **`code` feature** — `octo-rig = { features = ["code"] }` pulls `octo-code` as an
  optional dependency and exposes its tools. Not a bus connector. The coding tools
  (file `read`/`write`/`edit`/`list`, later glob/grep) live in octo-code; octo-rig
  keeps its dispatch bridge and opts them in. Built by **porting `llm-coding-tools-rig`**
  (Apache-2.0) to rig 0.35, `rig_tool` macro where the tool is stateless.
- **Safety = folder-level only; forkd NOT needed yet.** The current need is a jailed
  working folder, not a code sandbox. Albert generates working artifacts in a **/tmp
  scratch workspace**, path-confined (reject `..`/absolute, atomic writes per
  [[openclaw_code_fs]]) — that confinement *is* the safety. Revisit forkd/exec-sandbox
  only when we actually run untrusted code.
- **Durable artifacts → a `storage` connector (an octo organ), not raw FS.** Finished
  outputs are handed to a storage connector reached via dispatch (env-as-tools),
  backend **swappable: a local dir now, an S3-compatible store later**. The split:
  ephemeral work in /tmp (octo-code tools) vs. durable results in storage (an organ),
  cloud-portable from day one.

- **House style.** The reference is not copied verbatim — it is **ported to rig 0.35
  and refactored to our conventions**: files ≤~500 lines, brace-grouped leaf imports
  (no module paths in bodies), **no `anyhow`** (octo `OctoError`/thiserror), no emoji,
  `rig_tool` macro for the stateless tools. Apache-2.0 attribution kept.

**Two deliverables:** (1) **octo-code** (its own crate; coding tools, /tmp-jailed;
wired via octo-rig's `code` feature); (2) a **storage connector** (swappable
local↔S3). **forkd is parked.**
