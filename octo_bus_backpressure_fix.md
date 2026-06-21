# Octo bus backpressure — fix brief

**Status:** done
**Created:** 2026-06-21
**Closed:** 2026-06-21
**Kind:** task (Octo-side prerequisite for the GraphCogitator)
**Part of:** [[roadmap]] — Octo-side prerequisites
**Blocks:** the GraphCogitator under wide perception + steering ([[context_memory_layering]], [[agent_runtime_landscape]] Finding 2)

Implementation-ready context for fixing the one real Octo-core gap that the
GraphCogitator needs. Everything else in the assembly is userland (build the
cogitator crate); this is the kernel change. Paths are in `~/code/personal/octo/octo/`.

## Outcome

**Done — landed in `~/code/personal/octo/` (commit `03467eb`, 2026-06-21).**

- **Phase 1 (visibility + knobs):** `Subscription::next()` surfaces broadcast lag
  (warn + counter), exposed via `lagged_total()` — the silent-drop site is now
  observable. `subscribe_sync` takes `SubscribeOptions`, threaded through the
  runtime (cogitator / router / control). New `Cogitator::subscribe_options()`
  hook → a cogitator can request a non-default subscription.
- **Phase 2 (per-subscriber shim):** a forwarder task drains the broadcast into
  the subscriber's own bounded queue under its strategy (DropOldest / DropNewest /
  Steer / Throttle / best-effort Block); a slow consumer never stalls the others.
  `Steer` supersedes by channel/correlation = the steering primitive.
- Default `SubscribeOptions` → `DropOldest` (the fan-out native, = what the ignored
  opts already did) → octolab unchanged.
- **Phase 3 (never-drop control lane):** deferred, as planned.
- **Tests (acceptance):** `bus_lag_is_surfaced_not_silent`,
  `bus_shim_deep_buffer_no_drops`, `bus_steer_supersedes_pending` — 59 pass.

The one true Octo-core prerequisite is closed → the GraphCogitator has no kernel
blocker. Steering now has its bus primitive (`Steer` + the `subscribe_options`
hook); wiring it into the GraphCogitator's subscriptions is userland.

## Goal

Make the in-process bus **not silently drop envelopes** for a slow/high-rate
subscriber, and give the cogitator's subscription a real backpressure policy.
Today drops are silent and unconfigurable; a graph cogitator under Fluxion-rate
perception (or while steering) will lose perception and steer envelopes invisibly.

## Evidence (current state, with locations)

- `octo-core/src/bus.rs`
  - `InProcessBus` is a raw `tokio::sync::broadcast::Sender`.
  - `EventBus::subscribe(filter, _opts)` — **`_opts` is ignored** (explicit
    `// TODO: opts ignored`).
  - `InProcessBus::subscribe_sync(filter)` — used by the runtime for the
    **cogitator, router, and control listener** — takes *no* `SubscribeOptions`
    at all; raw `self.sender.subscribe()`.
  - `Subscription::next()` — on `Err(Lagged(_skipped))` does `continue`: the
    skipped count is **discarded** (`// TODO: surface lag count via tracing /
    metrics`). This is the silent-drop site.
  - Struct-level TODO already names the intended fix: *"Block / Throttle / Steer
    need a shim layer with per-subscriber mpsc fan-out."*
- `octo-core/src/connector/subscription.rs`
  - `SubscribeOptions { backpressure, buffer: 256, concurrency, panic_policy }`,
    default `backpressure = Block` — **but nothing honors it**.
  - `BackpressureStrategy { DropOldest(default), DropNewest, Block, Throttle{rate_per_sec}, Steer }`.
    `Steer` is documented as *"Merge incoming into the in-flight item… Borrowed
    from OpenClaw's steer mode"* — i.e. steering is already specced here.
- `octo-core/src/runtime.rs` — `self.bus.subscribe_sync(cog_filter)` for the
  cogitator (and router, and `CONTROL_GLOB`): no opts path exists.
- `octolab/src/main.rs` — `Octo::builder()` with default `bus_capacity 1024`,
  no opts. This is why octolab never trips the bug (low human rate + 1024 ring +
  dispatch uses its own `by_correlation` subscription). Latent, not solved.

## The core tension the implementer must resolve

**`Block` is fundamentally at odds with a fan-out broadcast.** Truly blocking the
publisher until a slow subscriber catches up = head-of-line blocking that freezes
*every* subscriber (the whole nervous system stalls on one slow consumer). With
`broadcast::Sender::send` being non-blocking, end-to-end Block isn't directly
expressible. So:

- **Per-subscriber independent bounded queue + lossy-but-visible policy** is the
  realistic model (a slow consumer drops/steers on its *own* queue; others
  unaffected).
- **`Block` should be best-effort / deferred**, documented as incompatible with
  fan-out — not silently pretended.

This is the key design call; the rest is mechanical.

## Proposed fix (phased — phase 1 alone removes the silent-loss bug)

### Phase 1 — visibility + knobs (cheap, high value, do first)
1. `Subscription::next()`: on `Lagged(n)`, `tracing::warn!` with subscriber
   identity + `n`, and bump a per-subscription counter. Expose `lagged_total()`.
   → converts a silent correctness bug into an observable one.
2. Make `subscribe_sync` accept `SubscribeOptions` (so the runtime can hand the
   cogitator a non-default subscription). Thread opts through `runtime.rs`.

### Phase 2 — per-subscriber mpsc shim (the real fix)
3. `subscribe`/`subscribe_sync` spawn a forwarder task:
   `broadcast::Receiver → filter → apply BackpressureStrategy → bounded mpsc(buffer)`.
   `Subscription` wraps the mpsc `Receiver`.
4. Implement strategies at the mpsc:
   - `DropOldest` — ring; drop front on overflow (count it).
   - `DropNewest` — drop arrival on full (count it).
   - `Throttle{rate_per_sec}` — rate-limit.
   - `Steer` — bounded-supersede: keep the latest per key (channel or
     correlation), merging a follow-up into the pending item. **This is the
     steering primitive** — a mid-flight follow-up supersedes the queued one.
   - `Block` — best-effort (forwarder awaits mpsc capacity); document that the
     upstream broadcast can still lag → not true end-to-end Block. Recommend
     `Steer`/deep-buffer for the cogitator instead.

### Phase 3 — optional: criticality lanes
5. Consider a non-lossy guarantee for **delivery-critical** kinds —
   `octo.control.**` and steering messages must never drop, while `vision.**`
   may drop-with-count. Either a deep/Steer buffer for the cogitator + a separate
   never-drop control subscription, or kind-aware policy. Note as an option, not
   a hard requirement; decide when wiring the GraphCogitator's subscriptions.

## Recommended cogitator subscription (once Phase 2 lands)

Not part of the kernel fix, but the consumer-side intent that justifies it: the
GraphCogitator should subscribe with a **deep buffer + `Steer`** for chat (so a
follow-up supersedes a pending one = steering), and treat high-rate perception
(`vision.**`) as `DropOldest`-with-visible-counts. Control-plane stays non-lossy.

## Acceptance criteria

- Burst test: a fast producer (e.g. 10k-envelope burst) + a deliberately slow
  subscriber → with `DropOldest`, drops are **logged with counts** (no silent
  loss); with a deep buffer, **no drops**; a test asserts lag is surfaced.
- `Steer` test: a subscriber with `Steer` gets only the latest of a superseding
  sequence (follow-up on the same channel supersedes the pending item).
- `subscribe_sync` accepts opts; the runtime gives the cogitator a non-default
  subscription.
- Regression: octolab still runs unchanged on defaults.

## Scope boundaries (do NOT)

- Not the distributed / NATS-backed bus (separate, later).
- Not changing the `Cogitator` trait — it's already sufficient (cognition is
  userland).
- Not true global `Block` — documented as deferred / incompatible with fan-out.

## Files

- `octo-core/src/bus.rs` — `Subscription`, `InProcessBus::{subscribe, subscribe_sync}`, `next()`.
- `octo-core/src/connector/subscription.rs` — strategy helpers if needed.
- `octo-core/src/runtime.rs` — thread `SubscribeOptions` into the cogitator /
  router / control `subscribe_sync` calls.
- `octo-core/src/lib.rs` — bus tests live here; add burst + Steer tests.

## Connections

- **Why it matters now:** [[context_memory_layering]] — staged cognition + steering.
- **Survey that surfaced steering/backpressure:** [[agent_runtime_landscape]].
- **Plan:** [[roadmap]] — the only true Octo-core prerequisite; the GraphCogitator
  itself is userland.
