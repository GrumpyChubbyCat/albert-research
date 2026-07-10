# OpenClaw — how it works with files & code (feeds phases 2-3)

**Kind:** research note / source summary
**Informs:** [[roadmap]] (phase 2 file workspace, phase 3 forkd), [[declarative_skills]]
**Status:** settled finding (verified by sub-agent against repos/docs, 2026-07-11)

Research trigger: the claim that "OpenClaw uses *picode* for the filesystem."

## Headline: there is no "picode"

It is a mishearing of **PI** — the TypeScript agent toolkit by Mario Zechner
(`github.com/badlogic/pi-mono`, npm `@mariozechner/pi-coding-agent`; "pi coding
agent" → "picode"). OpenClaw does **not** write its own file/code layer — it
**embeds PI in-process** (`createAgentSession()` / `AgentSession`, wired in
`src/agents/`) and wraps it. So: **PI = the tools; OpenClaw = the confinement +
sandbox around them.** Do not grep for a `picode` symbol.

## The tool surface (from PI)

- Default model-facing tools: **`read`, `write`, `edit`, `bash`**. Opt-in:
  `grep`, `find`, `ls` (via `--tools`).
- **`edit` is string-replacement, NOT unified-diff / apply_patch.** (read = file/
  image contents; write = create/overwrite.) A proven *minimal quartet*.
- Custom tools register via `registerTool({ name, description, parameters:
  Type.Object(...), execute })` (TypeBox schemas). Tools "participate in a file
  mutation queue" to avoid parallel-write conflicts.

## Confinement (OpenClaw's own layer — the part we most want)

On top of PI, `src/agents/workspace.ts` + `src/agents/sandbox/fs-bridge*.ts`:

- Path resolution through `resolveUserPath` / `openRootFile`, validating that a
  resolved path can't escape the **workspace root**; `walkWorkspaceFiles` rejects
  entries where `rel.startsWith("..") || path.isAbsolute(rel)`.
- Writes: `replaceFileAtomic` with `O_EXCL` / `O_NOFOLLOW` and `0o600` (anti-
  symlink, anti-race).

## Code execution & isolation

- PI's `bash` by default **runs on the host with the user's full permissions —
  unsandboxed.** (Optional opt-in extension uses `@anthropic-ai/sandbox-runtime` →
  `sandbox-exec` on macOS, **bubblewrap** on Linux.)
- OpenClaw swaps `bash` for **`exec` / `process`** and runs it in a sandbox backend
  (`src/agents/sandbox/`): **docker** (default) or **ssh**. Verified defaults in
  `sandbox/config.ts`:
  - `network: "none"` (only the browser container overrides),
  - `readOnlyRoot: true` + writable **tmpfs** at `/tmp`, `/var/tmp`, `/run`,
  - `workspaceAccess: "none"` — you must opt a workspace in (ro/rw),
  - `capDrop: ["ALL"]`, seccomp/AppArmor, PID/mem/CPU limits, env sanitisation,
  - escape hatches all off by default, `dangerously*`-prefixed.
- **Trust split:** single-user "main" sessions run tools **directly on the host**;
  group/channel sessions run **inside the sandbox** (`tool-policy.ts`,
  `validate-sandbox-security.ts`).

## Skills

PI implements the **Agent Skills** standard: `SKILL.md` files, invoked `/skill:name`,
instruct the model and may bundle/point to scripts it then runs via bash/exec.
Confirms our declarative→executable split ([[declarative_skills]]): the SKILL.md is
the instruction bundle; execution is a separate concern (our forkd).

**Permission-gate pattern (borrowable):** subscribe to the `tool_call` event,
inspect/mutate `event.input`, return `{ block: true, reason }` to veto (e.g. block
`rm -rf`, `sudo`); or override a built-in by **registering a tool with the same
name**. This is exactly our "deterministic reflex before a dangerous action" idea,
applied to tool calls.

## What this means for Albert

- **Phase 2 (file workspace)** — mirror PI's `read` / `write` / `edit`
  (string-replace) confined to ONE workspace root, reusing OpenClaw's path-safety
  primitives (reject `..`/absolute, atomic `O_EXCL`/`O_NOFOLLOW`/`0o600`).
  Language-agnostic — reimplement in Rust as an `fs` organ. **No container needed
  here**; path confinement is the safety for read/write/edit.
- **Phase 3 (forkd)** — only `bash`/`exec` needs the real sandbox. Albert runs as
  **root on a small VM with no docker to lean on**, so the **fs-bridge path-safety
  layer matters more for us than the container layer**; add bubblewrap/landlock
  (Linux) or go WASM-first. The posture defaults above (`network:none`,
  read-only root, workspace opt-in, cap-drop-all, `dangerously*` escapes) are a
  ready-made template.
- **Borrow:** the read/write/edit/bash quartet; the `tool_call` gate + same-name
  override; the path-confinement primitives; the config-posture defaults; SKILL.md.
- **Skip:** the Gateway / multi-channel messaging stack, ClawHub, Live Canvas,
  voice, companion apps; and — for a single root VM — the docker/ssh backend
  machinery itself (keep only the *policy* it encodes); PI's extension-SDK breadth
  (sub-agents, plan mode, TUI, compaction).

## Caveats

Exact JSON arg shapes for PI's read/write/edit/bash aren't published in the docs
read (behaviour + string-replace nature of `edit` confirmed; field-level shapes
would need PI's tool source). `src/agents/pi-tools.ts` referenced but 404'd on raw
fetch. "OpenShell" appears in marketing; only `docker` + `ssh` verified in code.
