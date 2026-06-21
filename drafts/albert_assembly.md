# Albert Assembly — Octo as the Reaction Skeleton

**Status:** draft
**Target archival path:** (TBD — sacrarium `lamantin_ai/research/ai/` once stable)
**Created:** 2026-06-21

How the maturing LamantinAI substrates compose into Albert, with **Octo as the
Reaction layer and structural skeleton**. This page is the assembly contract: what
plugs in where, in which *shape*, and which seams are still open.

## Thesis: Octo is the spine, not the brain

Octo is the **environment in which an agent lives**, not the agent. Its job is to
be the reliable nervous system: route perception → reflex/cognition → action,
supervise long-lived connectors, hold the bus. The intelligence lives at the
**edges** — Fluxion (perception that already thinks), kaeru (memory), the
cogitator (cognition). Octo stays general; it must not assume a chat-shaped or
vision-shaped world in its core.

## Two kinds of "tool", by what the thing *is*

A clean split that decides every integration:

- **Connectors = organs of the body** (IO on the bus). Perception in, action out.
  The agent's action space is the connector set (env-as-tools, reached via the
  `octo-rig` `dispatch_to_connector` tool). Transport/IO concerns.
- **rig-tools = instruments of the mind** (held by the cogitator, *off* the bus).
  Cognition-intimate capabilities the agent invokes directly while reasoning.

Memory is a mind-instrument, not an organ — too intimate to cognition to be bus
traffic. Perception and action are organs.

## Component placement

| Component | PEMRR layer | Shape in Octo | Why this shape |
|-----------|-------------|---------------|----------------|
| **Fluxion** | Perception | **sensor connector** (emits `vision.*`) + a video **tool** | heavy GStreamer/NPU runtime on edge — a *device*, never vendored. Transport likely NATS/MQTT (CrabbyQ) when it runs on separate hardware |
| **kaeru** | Memory | **rig-tool** (`kaeru-rig` adapter) + optional cogitator-side recall | memory is a mind-instrument; agent-driven `recall`/`jot` matches kaeru's own "facilitator, not enforcer". `kaeru-core` *can* embed (RocksDB single-writer) or run as the `kaeru-mcp` daemon |
| **Octo (+CrabbyQ)** | Reaction | the skeleton itself | bus, connectors, reflex/router, cogitator, supervision, control-plane |
| GraphCogitator | (cognition) | a `Cogitator` impl over **graph-flow** | the PEMRR loop is multi-step and stateful — a graph, not a single LLM call |

The cogitator **requires a graph**: the PEMRR loop (perceive → interpret →
recall → reflect → react) is a stateful workflow. `graph-flow` (rs-graph-llm,
the LangGraph analog) is the spine of cognition; its nodes call rig-tools
(kaeru) and connectors (via octo-rig). Crucially, `Cogitator` is a **pluggable
trait** — `GraphCogitator` is just another impl; the Octo skeleton is unchanged.
octolab ran a single-shot `ReactCogitator`; Albert gets the graph.

## The layering

```
Octo skeleton        — bus, supervision, control-plane, connectors
  GraphCogitator     — graph-flow, the PEMRR loop
    rig-tools        — kaeru-rig (memory) — instruments of the mind
    connectors       — Fluxion (eyes), telegram (mouth), …   — organs of the body
```

Octo = spinal cord; mind (graph-flow) and instruments (kaeru) above; eyes
(Fluxion) below-and-beside as a sensor.

## Already aligned in Octo (built toward this)

- **`vision.*` event-kinds live in `octo-core`** (`vision.incident.fight`,
  `vision.entity.entered_zone`) — the runtime was designed for Fluxion's eyes.
- **Envelope is HTTP/NATS-shaped** → the bus can go distributed via CrabbyQ/NATS
  when nodes spread across hardware.
- **Reflex/cognition split + rule-based Router** — exactly what Fluxion needs:
  reflex reacts deterministically or escalates to cognition.
- **Two-level perception (delivery + recall), env-as-tools, outbound media path.**
- **Supervision + control-plane (`octo.control.restart_*`)** → self-restart after
  config — the OpenClaw gap that motivated Albert. Albert already does what its
  reference can't.

## Octo-side gaps Albert needs

1. **Inbound media / Fluxion event contract** — the inbound half of the media
   path; an `octo-connector-fluxion` + an agreed envelope for scene events (also
   an open question on Fluxion's side: a clean event-emission layer over its
   metadata propagation).
2. **kaeru-rig adapter** — kaeru has no Rig adapter yet; this is the memory seam.
3. **Distributed transport** — a NATS/CrabbyQ bus backend for cross-host nodes.
4. **The connectors themselves** — fluxion (in), telegram, … and the GraphCogitator.

## Out of scope (for now)

- **Skills / sandboxed code execution** (forkd & friends). Isolation belongs
  *below* a REST API — a substrate concern owned by whoever runs the skill, not a
  remote hop. Deferred; revisit as a Reaction research-node later.

## Connections

- **Assembly target:** [[albert_system_map]] — the PEMRR decomposition.
- **Reaction skeleton:** [[octo_runtime]] — Octo (tentacles + core, 5 layers).
  Code & detailed design at `~/code/personal/octo/` (separate vault).
- **Perception:** [[fluxion]] — sensor-connector source, `vision.*` emitter.
- **Memory:** [[kaeru]] — rig-tool target; `kaeru-rig` adapter is ours to write.
- **Plan:** [[roadmap]].
