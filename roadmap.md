# Albert — Assembly Roadmap

**Scope:** composing Albert from Octo (Reaction) + Fluxion (Perception) + kaeru
(Memory), with a graph-flow cogitator as the PEMRR loop.
**Status:** active

## Current phase

Octo (Reaction skeleton) is real and exercised in `octolab`: telegram connector,
env-as-tools, two-level perception, modular history, supervision + control-plane.
Albert is a **separate repo-assembly** that depends on Octo as the skeleton and
wires the substrates in. octolab was the dress rehearsal.

**Building blocks now in hand** (state sync 2026-06-21): `kaeru-rig`, `octo-rig`,
the telegram connector, and the `graph-llm` crate all exist. Three of the four
integration seams below have their tooling done; the live front is the
**GraphCogitator** — drafting it by octolab's playbook is the next move.

## Open fronts (integration seams)

1. ~~**kaeru-rig**~~ — **done.** `kaeru-core` / `kaeru-mcp` wrapped as a `rig::Tool`
   (memory as a mind-instrument). Memory seam closed.
2. **GraphCogitator** ← *Phase 2 front.* A `Cogitator` impl over `graph-llm`; nodes
   = PEMRR stages, calling rig-tools (kaeru-rig) and connectors (octo-rig
   dispatch). **Phase 1 done (2026-06-22):** a working `AlbertCogitator` (rig
   tool-loop, not yet a graph) ships in the `albert` crate (`~/code/personal/albert/`)
   with kaeru memory + the new `octo-connector-scheduler` + the reminder loop —
   builds, smoke-tested. The whole non-cognition layer (scheduler, kaeru wiring,
   reminder flow, bus `Steer`) carries into the graph version unchanged; Phase 2
   swaps only the inner cognition engine for graph-llm.
3. **Fluxion connector** — `octo-connector-fluxion` emitting `vision.*`; needs the
   inbound media/event contract (open on both sides). Blocks on Fluxion's
   event-emission layer.
4. **Distributed transport** — NATS/CrabbyQ bus backend when nodes span hardware.

## Octo-side prerequisites (verified against live `~/code/personal/octo/`)

Code review (2026-06-21) collapsed this list: the `Cogitator` trait is already a
long-running, bus-connected, supervised actor ("cognition is userland"), so the
GraphCogitator is a **userland crate**, not a kernel change. The one real gap is
now **closed** — no remaining Octo-core blockers for the GraphCogitator:

- ~~**Bus backpressure**~~ — **done** (octo `03467eb`, 2026-06-21). Lag is now
  visible + counted (`lagged_total()`); a per-subscriber shim honors
  DropOldest/DropNewest/Steer/Throttle/Block without stalling others. `Steer` +
  the new `Cogitator::subscribe_options()` hook give steering its bus primitive.
  The single true core prerequisite is closed. Brief: [[octo_bus_backpressure_fix]].
- ~~Memory exposure surface~~ — **struck.** Overstated: the cogitator holds tools
  directly (`OctoDispatchTool` is constructed in-cogitator); kaeru-rig wires in
  identically. No core surface needed.
- Inbound media path — **partially done** (Telegram photos perceived inbound as
  `Blob`); the Fluxion scene-event contract is still open, but that's the
  Perception front, not a GraphCogitator blocker.

## Suggested next step

**GraphCogitator draft, by octolab's lekala.** The three tools it composes
(kaeru-rig, octo-rig, graph-llm) are all in hand; no external blockers. Awaiting
incoming material from the user before drafting the design.

## Capabilities — next fronts (post Phase-1, brainstormed 2026-06-22)

Each is an **organ** (connector) unless noted — Albert's action space = the
connector set, so a new connector with a `description` lands in his hands with
**zero cogitator change** (proven by the scheduler).

- **Web search + parse** — *user is building it.* One `octo-connector-web` organ:
  `web.search { engine: yandex|google, query }` + `web.fetch { url, render:
  auto|http|browser }`. **The agent chooses Playwright via the `render` param**
  (`auto` = cheap HTTP+readability first, escalate to browser on JS/block; force
  `browser`/`http` to override) — the latency tradeoff is taught in the connector's
  `description`. Playwright (Node) runs as a supervised **sidecar** the connector
  drives. Yandex first, Google later. Octo-side work = the connector wrapper
  (scheduler-shaped); the parser core is the user's.
  - **webclaw rejected** (`0xMassi/webclaw`): technically a clean Octo-connector
    fit (MCP/axum sidecar), but **AGPL-3.0** (copyleft gate for a productized
    LamantinAI) and it offloads exactly the JS/captcha cases to a **paid cloud** —
    the cases Playwright owns. Kept only as a *reference* for the browserless
    fast-path layering (TLS-fingerprint + readability), not a dependency.
- **Loop scratchpad** — **built 2026-06-28** (`albert/src/scratchpad.rs`). A
  super-operational per-task working object the agent authors (`scratchpad_goal/
  step/mark/note/clear`) and sees each turn; makes multi-step tasks verifiable
  (done = all steps `verified`). Distinct from chat (log) and kaeru (durable, tool).
  Design: [[loop_scratchpad]]. This is the realization of the "cycle window" idea —
  delivered on the plain loop, no graph needed.
- **Skills** — two things, don't conflate: *declarative* (SKILL.md registry the
  cogitator equips; mind-side, no exec, metadata can live in kaeru) vs *executable*
  (code → needs the sandbox substrate below). **Declarative: design worked out**
  ([[declarative_skills]] — storage/infra/visibility/accessibility, progressive
  disclosure); **implementation is the next build.**
- **Code (minimal) + executable skills** — converge on one **sandboxed execution
  substrate** = the reactivated **forkd** (see Deferred). Even "minimal" code needs
  real isolation (landscape lesson; OpenClaw's picode runs in a docker sandbox).
  Tiers: WASM-first (Wasmtime/WASI, minimal-safe, à la moltis `wasm-calc`) → full
  runner connector (bubblewrap/landlock/container, picode-level) as an organ
  owning isolation below REST.
- **Proactivity via scheduler + memory routines** — **built 2026-07-09.** Reuses
  the scheduler; no new substrate. A *routine* = a recurring system alarm keyed by
  payload `{ routine: "memory_reflection" }` (silent, no user message) vs a user
  reminder `{ task, channel, reply_via }`. `on_alarm` routes routines to
  `run_routine`; Albert **idempotently seeds** the base reflection routine on
  startup (retry until the scheduler is up; `ALBERT_REFLECTION_SECS` override,
  default 6h). The handler runs a silent reflection agent turn using
  `kaeru_reflect` (shipped in kaeru v0.4.1) + `link`/`synthesise`. Verified live
  (seed → fire → route → reflect). *Open:* seeding is "add if absent" — changing the
  period needs clearing the alarm; a per-routine upsert is a later nicety.
- **Cycle / Phase-2 graph** — the GraphCogitator splits today's single working-first
  turn into explicit PEMRR nodes. Value confirmed (user, 2026-06-28): **between
  stages the agent must have its accumulated context in front of it** — the graph's
  threaded, checkpointable state ([[context_memory_layering]] tier 2) gives exactly
  that (perceived → interpreted → recalled → reflected → react), vs one opaque
  rig-tool-loop turn. Resolves the open "does the graph earn its complexity?"
  hypothesis: **yes** — the explicit inter-stage context is the payoff.

## Deferred (not now)

- ~~Skills / sandboxed execution (forkd)~~ — **reactivated** 2026-06-22: skills are
  now due (user request). Promoted to *Capabilities* above as the sandboxed-
  execution substrate (WASM-first → full forkd runner). Still a Reaction
  research-node, just no longer deferred.
