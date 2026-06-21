# Context / memory layering — three tiers, three owners

**Kind:** scratch / design note
**Created:** 2026-06-21
**Status:** working model (crystallised from the landscape survey + user correction)

The competing runtimes ([[agent_runtime_landscape]]) collapse "memory" into one
pluggable context-engine subsystem. Albert should **not** — it has three distinct
substrates and the concerns split cleanly across them. Conflating them (as the
first draft of the survey did) leads to bad calls like "checkpoint via kaeru".

## The three tiers

| Tier | What it is | Owner | Operations | Lifetime |
|------|-----------|-------|-----------|----------|
| **Hot context** | the live working window — message history | **Octo** (modular history) | append, **compact the tail** | per-turn / session |
| **Staged cognition state** | the PEMRR-graph state threaded between nodes | **graph-llm** (checkpointer) | snapshot per super-step → **rollback / branch / resume / steering-interrupt** | per-run |
| **Deliberate memory** | the typed graph the agent thinks *in* + long-term итоги | **kaeru** | `recall` / `jot` / `consolidate_out` | cross-session, durable |

## Why this split

- **Hot ≠ kaeru.** The hottest layer is *not* kaeru — it's Octo's message
  history. That's the live window that gets compacted under token pressure. kaeru
  is one tier up: *active memory for thinking*, agent-invoked (`recall`/`jot`),
  not an automatic conversation buffer.
- **Checkpoint belongs to the graph, not to memory.** graph-llm holds typed state
  threaded through nodes; a checkpointer persists that state after each super-step.
  From that you get resume, **rollback** (rewind to a prior checkpoint),
  **branching** (fork from one), and human-in-the-loop **interrupt** — natively.
  moltis's `AutoCheckpointHook` (file snapshot before each mutating tool call) is a
  hand-rolled imitation of what a graph framework gives for free. So Albert's
  graph bet *also* buys checkpoint/rollback at the cognition-state level — no
  bespoke store, no abuse of kaeru's bi-temporal substrate for it.
- **Steering and checkpoint are the same graph capability.** A graph that can
  snapshot state *between* nodes can also accept injected input *between* nodes.
  Steering (mid-turn user correction that skips pending steps — picoclaw steering
  queue, moltis `SteerInbox`) is just the interrupt side of the checkpointer.
  Choosing graph-llm gets both at once.

## "Save before you compact" — routed correctly

The borrowed pattern from goclaw/moltis ("write durable memory before condensing
history") is **three operations on three tiers**, not one dump:

1. compact the **hot tail** — Octo history level.
2. checkpoint the **graph state** — graph-llm level (so a bad compaction is
   reversible).
3. `consolidate_out` the **итоги** — kaeru level (durable, provenance survives).

## Open questions

- Does the GraphCogitator read hot context *from* Octo history, or own its own
  node-local message state and sync back? (Where does the working window live when
  cognition is a graph?)
- kaeru recall as a graph node vs as a rig-tool the model calls ad hoc — both? The
  PEMRR "recall" stage suggests a node; the facilitator principle suggests a tool.
- Checkpoint granularity: per PEMRR node, or per tool call (moltis does per
  mutation)?

## Connections

- **Survey that motivated this:** [[agent_runtime_landscape]] — Findings 3 & 4.
- **Assembly contract:** [[albert_assembly]] — refines the GraphCogitator and
  kaeru-as-mind-instrument rows.
- **Memory substrate:** [[kaeru]] — the deliberate tier only.
- **Reaction skeleton:** [[octo_runtime]] — owner of the hot tier (modular history).
- **Plan:** [[roadmap]] — GraphCogitator is where tiers 2 & the steering/checkpoint
  story get built.
