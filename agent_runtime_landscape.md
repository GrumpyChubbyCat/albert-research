# Agent-Runtime Landscape — the "claw" clones & Rust analogs

**Kind:** scratch / competitive survey
**Created:** 2026-06-21
**Purpose:** look at other people's agent graphs before building Albert. Five
personal-agent runtimes surveyed from source/docs: OpenClaw (the reference) +
two Go clones (picoclaw, goclaw) + two Rust analogs (openhuman, moltis).

All five **verified real and genuinely AI-agent runtimes** (not the Captain-Claw
game). Verdicts below are from READMEs, docs, and partial source reads — *not*
line-by-line audits; star counts and "X% AI-generated / <10MB RAM" figures are
project marketing, reported as-is.

## The five at a glance

| | OpenClaw | picoclaw | goclaw | openhuman | moltis |
|---|---|---|---|---|---|
| Owner | `openclaw` org | `sipeed` | `nextlevelbuilder` | `tinyhumansai` | `moltis-org` |
| Stack | TS/Node 24 (+Swift/Kotlin) | Go 1.25 | Go 1.26 | Rust+Tauri (single crate) | Rust (~80-crate workspace) |
| Shape | local daemon ("Gateway") | single static binary | single static binary (~25MB) | desktop app + Rust core | single binary (~60MB) |
| License | MIT | MIT | **CC BY-NC 4.0** (non-commercial!) | GPL | MIT |
| Maturity | beta, calver, ~huge stars | pre-1.0, self-flagged unsafe | v3.x beta, hyper-active | "early beta" 0.57.x | pre-1.0, calver, security-forward |
| Cognition | imperative attempt-loop | session-worker loop + SubTurns | hybrid V2 loop / V3 8-stage pipeline | hand-rolled turn harness | hand-rolled bounded runner |
| Graph engine? | **no** | **no** | pipeline (not a graph DSL) | **no** | **no** |

## Finding 1 — nobody runs a real cognition *graph*

The single most relevant result for Albert: **none of the five uses a
graph/state-machine cognition substrate** (no LangGraph/graph-flow analog, no
`rig`). Every one rolls a hand-written turn/attempt loop:

- OpenClaw: an "attempt loop" spread across `run-state` / `execution-phase` /
  `failure-signal` / `result-fallback-classifier` + interleaved compaction.
- picoclaw: session-keyed worker goroutines, `pipeline_setup → llm → tool →
  finalize` looping to `Finish`; nested **SubTurns** (≤3 deep, ≤5 concurrent).
- goclaw: the closest to staged — an **opt-in V3 pipeline** `Context → Think →
  Prune → Tool → Observe → Checkpoint → Finalize`, Think↔Observe ≤20 iters. But
  it's a fixed staged pipeline, not a declarative graph.
- openhuman / moltis: explicit bounded runners (`run_agent_loop_streaming`,
  `AgentLoopLimits`, loop-detector interventions).

**Implication:** Albert's bet — a `GraphCogitator` over `graph-llm` as the PEMRR
loop — is *differentiated*, not derivative. The field has converged on imperative
loops; a real graph is an actual fork in the road, not table stakes. Worth
treating as a hypothesis to validate (does the graph earn its complexity vs a
goclaw-style staged pipeline?), not an assumption.

## Finding 2 — the convergent skeleton (this is "Octo", independently rediscovered)

Strip the branding and all five share a shape that maps almost 1:1 onto Octo:

- **Gateway/daemon control plane** owning sessions, channels, tools, events.
- **Session-keyed concurrency**: cross-session parallel, **same-session strictly
  serialized** through a per-session queue (picoclaw `sync.Map`+steering queue;
  goclaw 4 semaphore "lanes" main/subagent/team/cron; moltis runner).
- **Many channel connectors** (Telegram/WhatsApp/Slack/Discord/Signal/Matrix…) —
  exactly Octo's "tentacles". OpenClaw ships 20+.
- **Sub-agent / delegation** as a first-class primitive (picoclaw SubTurn, goclaw
  agent-teams with sync/async/bidirectional, openhuman `run_subagent`, moltis
  `spawn_agent`).
- **Steering** — mid-turn user injection that skips pending tools (picoclaw
  steering queue, goclaw interrupt queue, moltis `SteerInbox`, openhuman
  steerable async subagents). **Octo doesn't model this yet — likely a gap.**
- **MCP** support everywhere; tools as the capability surface.

So Octo isn't novel in *shape* — it's a clean, supervised implementation of the
shape the whole field landed on. Octo's actual differentiators are elsewhere (see
Finding 3 + the graph in Finding 1).

## Finding 3 — safety: why one is "safer" than another

Safety here is two near-orthogonal axes: **isolation** (can a tool call hurt the
host?) and **recovery/supervision** (does the system survive its own mistakes?).

**Isolation strength (strongest → weakest):**

1. **moltis** — four-tier sticky chain: **Apple Container = per-sandbox VM**
   (kernel exploit can't reach host) → Docker/Podman (`--cap-drop ALL`,
   no-new-privs, ro-root) → **WASM/WASI** (Wasmtime, fuel + epoch timeouts) →
   restricted-host fallback. **Network default-deny**, SSRF guard. Only one here
   offering VM-grade isolation.
2. **openhuman** — pluggable `docker/bubblewrap/firejail/landlock`, auto-selected
   per host; risk-classified approval gating **on by default**; pairing guard
   before the RPC port opens. Weakness: `NoopSandbox` fallback + a *shared*
   Node runtime (bigger blast radius than the old per-skill QuickJS VMs).
3. **OpenClaw** — docker/ssh/openshell backends; sandbox **non-main sessions by
   default**, `network: none` default, blocks dangerous bind mounts
   (`docker.sock`, `/etc`, `~/.ssh`, `~/.aws`…). ATLAS-mapped threat model. Doc
   is honest: "not a perfect boundary."
4. **goclaw** — 5-layer defense-in-depth, AES-256-GCM at rest, **optional**
   Docker sandbox (`WITH_SANDBOX=1`). But injection detection defaults to
   **warn-not-block**, isolation is path-string based (symlink-race), regex shell
   denylists are evadable.
5. **picoclaw** — bubblewrap on Linux, **off by default**, **silently runs
   unsandboxed if `bwrap` missing**, macOS unsandboxed, **main process never
   isolated**. Maintainers themselves say "do not deploy before v1.0."

**Recovery/supervision** — the more interesting axis for Albert, and where the
field is *weak*:

- **moltis is the only one with a real recovery story**: `AutoCheckpointHook`
  snapshots files **before every mutating tool call** and before skill/memory
  ops; `/rollback` + `/rollback diff`; session branching. This is the
  "undo button for the agent" pattern.
- **OpenClaw explicitly LACKS daemon restart / watchdog / crash recovery** — its
  own threat-model doc says these are not implemented. **This is precisely the
  "OpenClaw gap" the Albert vault already names** (`octo.control.restart_*` +
  supervision tree). External confirmation: Albert's reference genuinely can't do
  what Octo's control-plane already does.
- The others: circuit-breakers on failing hooks (goclaw, 3-strike), history
  repair of orphaned tool_use/result pairs (goclaw `sanitizeHistory`, moltis
  `json_repair`/`tool_arg_validator`) — robustness patches, not supervision.

**Takeaway for Albert:** Octo's supervision + control-plane self-restart is a
real moat vs OpenClaw and the Go clones. The thing to *steal* is **moltis's
pre-mutation checkpoint/rollback** — and Albert can do it better, because kaeru's
bi-temporal substrate makes "snapshot before mutation, time-travel to undo"
native rather than a file-copy hack.

## Finding 4 — context handling, and how much of kaeru others are reinventing

All five converge on the same context-engine contract: a **pluggable manager**
with `assemble / ingest / compact`, **token budget with a response reserve**, and
**LLM-summarization compaction that keeps a recent tail**. (OpenClaw
`ContextEngine`, picoclaw `ContextManager`+"Seahorse" summarizer @100k default,
goclaw 4-mode prompt + 75% compaction, openhuman 4-stage reduction, moltis
tool-result budgeting.)

Two patterns worth lifting directly:

- **Write durable memory BEFORE you compact.** goclaw does a "memory-flush-first"
  pass (agent writes L1/L2 memories, *then* truncate to 4 msgs); moltis runs a
  **"silent agentic turn"** that persists prefs/decisions/project-context before
  condensing. Albert gets this almost for free: a `consolidate_out` to kaeru
  *before* the GraphCogitator compacts a node's history.
- **Memory as an agent-invoked tool, not an automatic subsystem.** goclaw
  (`memory_search` / `memory_expand`), moltis (`memory_save`), openhuman recall
  tools — all let the model *choose* to recall/expand. This is exactly the
  **kaeru-rig framing** (memory = mind-instrument, agent-driven `recall`/`jot`)
  and matches kaeru's "facilitator, not enforcer" principle. Validation that the
  rig-tool placement (vs an automatic context injector) is the right call.

And how much of kaeru others are independently rebuilding:

- **goclaw** is the loudest echo: 3-tier memory **Working/L0 → Episodic/L1 →
  Semantic/L2**, where L2 is a **knowledge graph with temporal validity
  (`valid_from`/`valid_until`)** and a Knowledge Vault using **`[[wikilinks]]`** +
  hybrid BM25+pgvector. That is kaeru's bi-temporal, two-tier, typed-graph,
  wiki-export thesis — reinvented inside a Go monolith.
- **openhuman** Memory Tree: hierarchical compression (leaves seal upward,
  `L_n→L_{n+1}`), Obsidian-style wiki, 70/30 vector/FTS retrieval.
- **moltis**: SQLite+FTS5+vector, optional `qmd` with BM25+vector+LLM-rerank
  (RRF), `MEMORY.md` + `memory/*.md`.

**Takeaway:** the market is validating kaeru's design by rebuilding fragments of
it inline. kaeru's edge is that it's a *standalone substrate* with native
bi-temporality, explicit two-tier consolidation, and provenance that survives the
tier boundary — not a per-app memory module welded into one runtime. Albert
wiring kaeru-rig in is the bet that a real memory substrate beats an inline one.

## What Albert should take away

1. **Keep the graph** — it's the genuine differentiator, but treat "graph earns
   its complexity over a goclaw-style staged pipeline" as a hypothesis to test.
2. **Add steering** to Octo — mid-turn user injection that skips pending tools is
   universal among the serious clones; Octo lacks it.
3. **Get checkpoint/rollback from the graph, not from kaeru.** *(corrected — see
   [[context_memory_layering]].)* moltis's file-snapshot `AutoCheckpointHook` is a
   hand-rolled version of what graph-llm's checkpointer gives natively (snapshot
   per super-step → rollback / branch / resume). Albert's graph bet already buys
   this at the cognition-state level. **Steering is the same capability** — a
   graph that checkpoints between nodes can take injected input between nodes too.
4. **"Save before compact" is three ops on three tiers**, not one dump *(corrected
   — see [[context_memory_layering]])*: compact the hot **Octo** history tail;
   checkpoint the **graph-llm** state; `consolidate_out` итоги to **kaeru**. The
   hottest layer is Octo's message history, **not** kaeru.
5. **kaeru-rig as agent-invoked tool is vindicated** — the field treats memory as
   a tool the model chooses, matching kaeru's facilitator stance.
6. **Supervision is already a moat** — OpenClaw can't self-restart; Octo can.
7. **Octo is the anti-velosiped** — Finding 2 means the field keeps rebuilding the
   gateway+session+connectors+sub-agent skeleton; Octo *is* that, done cleanly and
   supervised. The skeleton is commoditised and it's ours → Albert's novelty
   budget goes entirely to the graph (tier 2) and kaeru (tier 3), not the runtime.

## Connections

- **Assembly contract this informs:** [[albert_assembly]] — esp. the
  graph-cogitator and memory-as-mind-instrument decisions.
- **Plan:** [[roadmap]] — GraphCogitator is the current front; this survey is the
  "look around first" step.
- **Reaction skeleton being benchmarked:** [[octo_runtime]] — the convergent
  skeleton (Finding 2) + the supervision moat (Finding 3).
- **Memory substrate being validated:** [[kaeru]] — Finding 4; others reinvent
  fragments of it inline.
