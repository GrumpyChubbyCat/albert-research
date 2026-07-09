# Log — Albert Vault

Append-only. Every non-trivial change. Header: `## [YYYY-MM-DD HH:MM] <op> | <title>`.

## [2026-06-21 19:49] note | Vault initialised

Created the Albert operational vault (`~/code/personal/albert/research/`), mirroring
the Octo vault layout. Generic schema (`CLAUDE.md`, `STRUCTURE.md`) cloned from the
Octo vault; `README.md`, `index.md`, this `log.md` authored fresh.

## [2026-06-21 19:49] ingest | Seeded raw sources (no forkd)

Copied four immutable snapshots into `sources/sacrarium_snapshot/` from the Octo
vault: `albert_system_map.md` (PEMRR map), `octo_runtime.md` (Reaction runtime),
`fluxion.md` (Perception), `kaeru.md` (Memory). **forkd deliberately excluded** —
skills/sandbox is out of scope for now (per directive).

## [2026-06-21 19:49] note | Assembly contract drafted

Wrote `drafts/albert_assembly.md` — the integration design worked out in session:
Octo as the Reaction skeleton ("spine, not brain"); the connectors-as-organs vs
rig-tools-as-mind-instruments split; component placement (Fluxion = sensor
connector, kaeru = rig-tool via a `kaeru-rig` adapter, GraphCogitator over
graph-flow as the PEMRR loop on the pluggable `Cogitator` trait); the layering;
what's already aligned in Octo (`vision.*` kinds, NATS-shaped envelope,
reflex/cognition split, supervision + control-plane self-restart = the OpenClaw
gap); and the Octo-side gaps (inbound media/Fluxion event contract, kaeru-rig,
distributed transport). Plus `roadmap.md` — first step = kaeru-rig.

## [2026-06-21 22:47] note | State sync — three integration seams already closed

User reports reality has moved ahead of the vault. Of the four open fronts in
`roadmap.md`, the tooling for three is **already built**:

- **kaeru-rig** — done (was the "suggested first step"; memory seam closed).
- **octo-rig** — done (connector dispatch; action space as organs).
- **telegram connector** — done (the "mouth").
- **graph-llm** crate — exists (substrate for the GraphCogitator / PEMRR loop).

So the next concrete move is the **GraphCogitator draft, by octolab's playbook**
(octolab ran a single-shot `ReactCogitator`; Albert gets the graph). Roadmap
updated to reflect the shift. Awaiting incoming material from the user before
drafting — recording state now, design to follow.

## [2026-06-21 22:56] note | Surveyed 5 competing agent runtimes ("claw" landscape)

"Look around before building Albert." Analyzed five personal-agent runtimes from
source/docs (fan-out, one researcher per repo): **OpenClaw** (TS/Node reference),
**picoclaw** (Go, sipeed), **goclaw** (Go, nextlevelbuilder), **openhuman** (Rust,
tinyhumansai), **moltis** (Rust, moltis-org). All five verified real AI-agent
runtimes (not the Captain-Claw game). Wrote `agent_runtime_landscape.md`.

Headline findings:
1. **None runs a real cognition graph** — all hand-rolled imperative loops (goclaw
   has the closest: an opt-in staged pipeline, still not a graph DSL). Albert's
   `GraphCogitator`-over-`graph-llm` bet is differentiated, not derivative.
2. **Convergent skeleton ≈ Octo, independently** — gateway control-plane,
   session-keyed (same-session serialized) concurrency, many channel connectors,
   sub-agent delegation, **steering** (mid-turn user injection — Octo lacks this).
3. **Safety:** isolation strongest→weakest moltis (Apple-Container VM) > openhuman
   > OpenClaw > goclaw > picoclaw (opt-in, degradable). Recovery axis is weak
   field-wide; **moltis alone** has pre-mutation checkpoint+rollback. **OpenClaw
   explicitly lacks self-restart/watchdog → external confirmation of the
   "OpenClaw gap"** that motivated Octo's control-plane.
4. **Context:** all converge on pluggable assemble/ingest/compact + token budget +
   summarize-keeping-tail. goclaw (L0/L1/L2 + temporal-validity KG + `[[wikilinks]]`
   + BM25/pgvector) and openhuman (Memory Tree) are **reinventing fragments of
   kaeru inline** — validates kaeru as a standalone substrate. Memory-as-
   agent-tool everywhere → vindicates kaeru-rig framing.

Albert takeaways recorded on the page: keep+validate the graph; add steering to
Octo; steal checkpoint/rollback (via kaeru bi-temporal, not file snapshots);
consolidate_out-before-compact; supervision is already a moat vs OpenClaw.

## [2026-06-21 23:11] note | Three-tier context model; corrected checkpoint call

User feedback sharpened the landscape takeaways into a clean layering. Wrote
`context_memory_layering.md` and corrected two muddled takeaways on
`agent_runtime_landscape.md`.

- **Octo = anti-velosiped.** Finding 2 reframed: the field keeps rebuilding the
  gateway+session+connectors+sub-agent skeleton; Octo is that, done & supervised.
  Novelty budget goes to graph + kaeru, not the runtime. (Added as takeaway 7.)
- **Three tiers, three owners:** *hot context* = Octo modular history (the hottest
  layer — NOT kaeru); *staged cognition state* = graph-llm checkpointer; *delib-
  erate memory* = kaeru (`recall`/`jot`/`consolidate`).
- **Checkpoint correction:** my "checkpoint via kaeru bi-temporal" was wrong.
  Checkpoint/rollback/branch/resume is **native to graph-llm's checkpointer** over
  graph state; moltis's file-snapshot hook is a hand-rolled version of that. The
  graph bet buys it for free. **Steering = the interrupt side of the same
  checkpointer** (inject between nodes ⇔ snapshot between nodes).
- **"Save before compact" = three ops on three tiers**, not one dump.

Open questions parked on the layering page: where the working window lives when
cognition is a graph; kaeru recall as graph-node vs rig-tool; checkpoint
granularity (per node vs per tool call).

## [2026-06-21 23:40] note | Read live octo code; A–E reassessed; bus-fix brief

Read `~/code/personal/octo/octo/` (cogitator, bus, runtime, octo-rig, octolab,
octo-history, subscription). Snapshot had under-sold Octo — big correction:

- **`Cogitator` trait is already long-running/supervised/bus-connected** with a
  rich `CogitatorContext` (publish / bus / subscribe / publish_and_await_response
  / connectors / shutdown), pre-subscribed before connectors start. README:
  "cognition is userland — you bring the brain." So of last turn's asks A–E, only
  **D (backpressure) is a real core change**; A/B/C/E + Q1 are all userland (build
  the GraphCogitator crate). My earlier "single-shot cogitator" assumption was
  wrong — octolab's `ReactCogitator` is long-running; only its per-message
  cognition is a synchronous rig `multi_turn(5)` loop (the bit the graph replaces).
- **octo-history** = the hot-context tier, already a crate (`HistoryStore`,
  per-channel transcript, "NOT agentic memory"). Q1 closed: read window at run
  start, append canonical turns on commit. No core change.
- **Steering is pre-designed**: `BackpressureStrategy::Steer` ("borrowed from
  OpenClaw's steer mode") exists but is unwired.
- **Bus backpressure bug confirmed real but latent in octolab** — masked by low
  human rate + default 1024 broadcast capacity + dispatch's own by-correlation
  subscription. NOT solved, just not provoked. Will bite the graph: wide/high-rate
  perception (Fluxion) + slower cognition + steering-needs-undrained-loop ⇒ lag >
  capacity ⇒ silent drop (`Subscription::next` discards the `Lagged` count).
- Wrote `octo_bus_backpressure_fix.md` — implementation-ready brief (evidence w/
  file locations, the Block-vs-fan-out tension, phased fix, acceptance criteria,
  files). Updated `roadmap.md` Octo-side prereqs: backpressure = only true core
  gap; memory-exposure struck; inbound-media partially done. User will return
  with the fix implemented.

## [2026-06-22 00:01] note | Bus fix landed; convergence insight consolidated to Octo vault

- **Bus backpressure fixed** by the user (Octo commit `03467eb`). `octo_bus_backpressure_fix.md`
  → `done`: Phase 1 (lag surfaced via `lagged_total()`; `subscribe_options()`
  cogitator hook) + Phase 2 (per-subscriber shim with DropOldest/DropNewest/Steer/
  Throttle/best-effort Block; `Steer` = the steering primitive). 59 tests pass.
  The only true Octo-core prerequisite for the GraphCogitator is closed — no
  kernel blocker remains; wiring `Steer` into the cogitator's subscriptions is
  userland.
- **Convergence insight consolidated cross-vault.** Per user, the survey's
  positioning takeaway (the whole personal-agent-runtime field independently
  rebuilds Octo's connector+channel skeleton → Octo's value as a runtime *is*
  that layer) belongs in Octo's own vault. Wrote it there:
  `~/code/personal/octo/research/drafts/field_converges_on_octo_shape.md` (+ its
  log/index), cross-referencing Octo's `novelty` / `octo_as_agent_runtime_eval` /
  `connector_channel_split`, with provenance back to our
  `agent_runtime_landscape.md`. Operational→operational cross-vault, path refs
  (not wikilinks).

## [2026-06-22 00:36] task | Albert Phase 1 built — talk + memory + reminders (working-first)

"Поехали." User chose **working-first** (extend octolab's rig tool-loop; graph is
Phase 2). Albert now exists as a runnable assembly.

- **New crate `octo-connector-scheduler`** (in octo, realises the `notifications.md`
  design): bidirectional managed connector; emits `alarm.fired`, accepts
  `octo.scheduler.{add,cancel,list}_alarm`, OneShot + Interval triggers, JSON
  persistence (atomic), 1s tick. Advertises a `description` → reachable via the
  cogitator's env-as-tools dispatch (no new tool). Test
  `interval_alarm_fires_and_cancels` (add→fire→cancel over the bus) green.
- **New `albert` crate** — own Cargo workspace at `~/code/personal/albert/` (own
  `[patch.crates-io]` for graph_builder, since the patch is per-workspace-root and
  kaeru pulls cozo). Path deps to sibling octo + kaeru checkouts (switch to git
  once pushed). `AlbertCogitator` = octolab's `ReactCogitator` grown up: perceives
  `chat.message` + `alarm.fired`; tools = `dispatch_to_connector` (scheduler) +
  kaeru verbs (awake/recall/read/remember/task/done/recent). Hot context =
  octo-history; deliberate memory = kaeru (initiative "albert").
- **Reminder loop:** ask → kaeru task + dispatch add_alarm (carries task + channel
  + reply_via in the alarm payload) → `alarm.fired` → recall + remind on the stored
  channel → user "done" → kaeru_done + dispatch cancel_alarm. Active alarms are
  front-loaded into each chat turn (queried from the scheduler) so the model
  cancels the right one by matching the task — no id threading.
- **Builds clean** (1m20s incl. cozo). Smoke test (console, `/start` reflex) round-
  trips end-to-end. LLM reminder dance not yet exercised (needs the user's
  OpenRouter key). Caveat: kaeru is single-writer — don't run Albert while
  kaeru-mcp holds the same vault.

## [2026-06-22 00:46] reorg | Code moved under `albert/`; env prefix OCTO→ALBERT

Mirror octo's layout: code now lives at `~/code/personal/albert/albert/` (crate),
parallel to `research/`, with `.env` at the repo root. Path deps deepened
`../…` → `../../…`; `DOTENV_PATH` → `…/../.env`. Config now reads the **`ALBERT_`**
env prefix (`ALBERT_OPENAI_KEY` / `ALBERT_LLM_MODEL` / `ALBERT_LLM_BASE_URL` /
`ALBERT_HISTORY`). The **Telegram token keeps its own name `OCTO_TELEGRAM_TOKEN`**
(it's the connector's, not Albert's app config) — read separately, `#[serde(skip)]`
on the field. Rebuilds clean; smoke test confirms ALBERT_ vars load + console
fallback when the token is empty.

## [2026-06-22 00:59] note | Phase 1 verified LIVE in Telegram — the reminder loop works end-to-end

First real conversation with Albert on a Telegram bot (LamantinAI Assistant,
model deepseek/deepseek-v4-flash via OpenRouter). User: "Прошло просто
великолепно!" The full loop ran end-to-end with a real LLM:
perceive request → kaeru task + scheduler add_alarm → `alarm.fired` → recall +
remind in-chat → user confirms → kaeru_done + cancel_alarm. The open worry from
the build entry (does the model thread channel/reply_via into the alarm payload?)
is resolved in practice — reminders landed in the chat. Albert is a real, talking,
remembering, reminding assistant. Octo (Reaction) + kaeru (Memory) + the scheduler
connector compose as designed. Next: Phase 2 (graph-llm GraphCogitator) when the
working-first version has earned its keep.

## [2026-06-22 01:25] note | Capabilities brainstorm; webclaw rejected; web-parser is a self-built organ

Discussed what Albert lacks. Recorded as a Capabilities section in `roadmap.md`.

- **webclaw** (`0xMassi/webclaw`) researched (cross-vault survey style). Real, Rust,
  clean Octo-connector fit (MCP/axum sidecar), TLS-fingerprint HTTP + readability +
  QuickJS data-islands; search = Serper API. **Rejected for the product:** AGPL-3.0
  (copyleft gate) + offloads JS/captcha to a **paid cloud** — exactly the cases
  Playwright owns. Kept as reference for browserless fast-path layering only.
- **Web search + parse → user builds it himself** (Yandex + Playwright, Google
  later). Octo seam settled: one `octo-connector-web` organ, `web.search` +
  `web.fetch { render: auto|http|browser }`; **the agent chooses Playwright via the
  `render` param** (auto = cheap-HTTP-first, escalate on JS/block), the tradeoff
  taught in the connector `description`. Playwright runs as a supervised Node
  sidecar. Zero cogitator change (env-as-tools). I scaffold the connector wrapper
  when the parser core is ready.
- **Skills** split into declarative (SKILL.md registry, mind-side) vs executable
  (code → sandbox). **Code + executable skills converge on one sandboxed-execution
  substrate = the reactivated forkd** (WASM-first → full runner connector). The
  old "deferred (not now)" forkd node is reactivated — skills are due.

## [2026-06-28 22:55] feat | Telegram authorization wired; directions agreed

Octo landed Telegram authorization (octo `b3cfebe`: edge ACL, role→TrustLevel,
runtime-mutable list). Updated Albert's integration:
- **Edge ACL:** `ALBERT_OWNER_CHAT` → `with_acl` seeded with the owner; unlisted
  chats dropped before the bus. No owner → `new` (open) + a plain WARNING log.
  (config.rs owner_chat, error.rs `Acl`, main.rs wiring, ACL at `state/`.)
- **Owner ACL admin (agent-driven):** mirrored octolab — `/allow <id>` `/deny <id>`
  `/allowed` as a **deterministic reflex** (security stays out of the LLM),
  owner-gated via the `role` tag on incoming channel metadata, dispatched to the
  telegram connector via `publish_and_await_response`. Emoji-free per new rule.
- Builds clean against updated octo (kaeru bumped 0.2.1→0.3.0; all additive).

**Two standing user rules recorded** (also saved to cross-session memory): no emoji
in code/logs; Conventional-Commits style (`feat:`/`fix:`/`doc:`/`chore:`), no
Claude co-author trailer.

**Directions agreed** (recorded in roadmap Capabilities):
1. **Declarative skills** = next concrete build.
2. **Proactivity = scheduler + memory routines** — reuse the scheduler; a routine
   is a recurring system alarm (`{routine: "memory_reflection"}`, silent) vs a user
   reminder; seed a base routine on startup; depends on kaeru's reflection primitive
   (next kaeru release).
3. **Graph (Phase 2) earns its keep** — user confirmed the payoff is *inter-stage
   context*: the graph threads accumulated PEMRR context node-to-node (tier 2 of
   [[context_memory_layering]]), vs today's one opaque turn.

## [2026-06-28 23:41] feat | Loop scratchpad built; declarative skills designed

Worked through "the loop should have its own context window." Landed on: kaeru stays
a **tool** (operational+persistent memory); the loop gets a **super-operational
scratchpad** — distinct from chat (a log) and kaeru (durable). Its value: makes
multi-step tasks **verifiable** (done = every step `verified`, not "the model felt
finished"). Honest framing: this is the known agent todo/plan pattern, just placed
correctly; not novel.

- **Built** `albert/src/scratchpad.rs`: channel-keyed in-memory store + 5 rig tools
  (`scratchpad_goal/step/mark/note/clear`). Cogitator renders the current scratchpad
  into the preamble each turn (like active_reminders); `run_agent` now takes the
  channel and binds the scratchpad tools; BASE_PREAMBLE teaches usage. Builds clean;
  emoji-free (also caught a stray `⏰` in a tracing log, removed — U+23F0 was outside
  my earlier grep range). Design: [[loop_scratchpad]]. NOT yet exercised live (needs
  the LLM path).
- **Declarative skills designed** [[declarative_skills]]: SKILL.md dir (one dir per
  skill), startup `SkillRegistry`, **catalog rendered each turn** (visibility),
  **`load_skill` tool** pulls one body on demand (accessibility / progressive
  disclosure — catalog is the index, body is the page, like kaeru structural recall).
  kaeru may index metadata but doesn't own skills. Executable skills stay on the
  forkd/sandbox track. Implementation is the next build.

Layering now: chat (Octo, a log) | scratchpad (super-operational task state) | kaeru
(durable memory, tool); plus a skills catalog (menu) + `load_skill` (page).

## [2026-07-09 22:46] chore | kaeru pinned to v0.4.1 (reflect); soul/memory design; octo connectors brief

- **kaeru pinned to the v0.4.1 release** — `albert/Cargo.toml` kaeru-core/kaeru-rig
  path deps → `git = "https://github.com/LamantinAI/kaeru", tag = "v0.4.1"`. Builds
  (cozo recompiled from the git source, 2m). `kaeru_reflect` is present in 0.4.1 →
  the memory-reflection routine is unblocked. (My local checkout had been on the
  `ceounittt` fork without the tag; origin is now LamantinAI.)
- **Soul/memory design settled** (OpenClaw's "reads all its files every request,
  half a minute per request" anti-pattern):
  - **Memory** — already solved: kaeru is curated, on-demand recall, NOT a
    memory.md bulk-injected each turn. We don't have OpenClaw's problem.
  - **Soul/instructions** — must be in the prompt every turn (they define the
    agent; can't be on-demand), but must NOT be disk-read per request. Design:
    **hold in RAM, hot-reload via mtime check** (stat each turn, re-read the body
    only if changed). Keep the always-present core LEAN; push specialized behavior
    to on-demand declarative skills. So: memory=kaeru on-demand, specialization=
    skills on-demand, only a thin soul/instruction core always-present (in RAM).
- **Octo connectors handoff written** [[octo_connectors_handoff]] — for a separate
  octo session: `connectors/yandex/` (Yandex Calendar via CalDAV — reqwest WebDAV +
  icalendar; list/create/delete events) + `connectors/smtp/` (generic send via
  lettre; `email.send`). Both organs (env-as-tools), secrets via app-password env,
  zero cogitator change; Albert wires them in main.rs like the scheduler.

Next local builds queued: soul/instructions RAM+hot-reload; base reflect routine
(now unblocked); declarative skills impl.

## [2026-07-09 23:02] feat | Base memory-reflection routine (scheduler-driven)

Albert's first proactive routine — reuses the scheduler, no new substrate.
- `on_alarm` now checks the alarm payload for `routine`; `{ routine:
  "memory_reflection" }` routes to `run_routine` (silent, no user message), else the
  existing reminder path.
- `run_routine("memory_reflection")` runs a silent reflection agent turn: prompt
  "reflect on your memory", tools now include `kaeru_reflect` (v0.4.1) + `link` +
  `synthesise`; it pulls the maintenance work-list and acts on it.
- `seed_base_routine` (detached task on cogitator start) **idempotently** seeds the
  reflection alarm: retries until the scheduler answers, checks list_alarms, adds
  only if absent. Period via `ALBERT_REFLECTION_SECS` (default 6h).
- **Verified live** (console, `ALBERT_REFLECTION_SECS=2`, dummy key): seed → every
  2s fire → routed to `run_routine` → reflection turn (401 on the dummy key,
  proving the plumbing). Cleaned the smoke-test alarm from `state/` so a real run
  seeds a fresh 6h routine. Builds clean, emoji-free.

Also pinned kaeru to v0.4.1 (committed `chore` in the code repo) — that's what
shipped `kaeru_reflect`.

## [2026-07-09 20:30] feat | Config-as-data: albert.toml + hot-reloaded soul/system prompts

Two moves the user asked for, both against the OpenClaw anti-pattern (re-reading
big files every request):

- **`albert.toml`** (config-as-data, like Octo's connector manifests). Path via
  `ALBERT_CONFIG` (default: next to the binary, else the crate dir). Secrets stay
  in env, **named in the TOML by their env-var name** (`openai_key_env`,
  `token_env`) — so the file is committable. model / base_url / owner_chat /
  reflection period / history / state paths / max_tool_turns all move out of code
  and env into the TOML. Relative paths resolve against the config file's dir.
- **soul.md + system.md** — the big `BASE_PREAMBLE` const leaves the code into two
  files: `soul.md` (persona) + `system.md` (memory/reminders/scratchpad protocol).
  Loaded into **RAM, hot-reloaded by mtime** (`stat` each turn, re-read a body only
  when it changed) — not disk-read per request. Terse embedded fallback if a file
  is missing. This resolves the A/B question → both files externalized.
- Also: refactor pass earlier this session — cogitator split into `acl`/`routines`
  (<500 lines/file), imports brace-grouped, leaf entities imported (no module paths
  in code bodies). Recorded as durable Rust style rules in cross-session memory.

**Migration note for the user:** `.env` now holds only secrets (`ALBERT_OPENAI_KEY`,
`OCTO_TELEGRAM_TOKEN`) + optional `ALBERT_CONFIG`. `ALBERT_LLM_MODEL` / `_BASE_URL` /
`_HISTORY` / `_REFLECTION_SECS` / `ALBERT_OWNER_CHAT` in `.env` are now **ignored** —
their values live in `albert.toml`. In particular `[telegram].owner_chat` must be
set in the TOML or the bot runs OPEN (with a warning).

## [2026-06-21 23:56] task | Bus backpressure fix landed — GraphCogitator unblocked

Implemented `octo_bus_backpressure_fix.md` in live octo (`~/code/personal/octo/`,
commit `03467eb`). The brief matched the code 1:1.

- **Phase 1 (visibility + knobs):** `Subscription::next()` now warns + counts on
  broadcast `Lagged` (no more silent drop), exposed via `lagged_total()`.
  `subscribe_sync` takes `SubscribeOptions`; threaded through the runtime. New
  `Cogitator::subscribe_options()` hook.
- **Phase 2 (per-subscriber shim):** forwarder task → bounded per-subscriber queue
  honoring DropOldest/DropNewest/Steer/Throttle/best-effort Block; a slow consumer
  no longer stalls the others. `Steer` supersedes by channel/correlation = the
  steering primitive (closes the "Octo lacks steering" gap from Finding 2).
- Default `SubscribeOptions` → `DropOldest` (= prior ignored-opts behavior) →
  octolab unchanged. Phase 3 (never-drop control lane) deferred.
- Acceptance covered by 3 new tests; octo-core 59 pass.

Marked the brief **done** (+ Outcome), struck the roadmap prereq, updated index.
**No remaining Octo-core blockers for the GraphCogitator** — its three tools
(kaeru-rig, octo-rig, graph-llm) are in hand and the steering primitive now
exists. Next concrete move stays the GraphCogitator draft.

## [2026-06-23 00:23] note | Reviewed colleague's two octo PRs; closing both, harvesting the best

@widgetii opened LamantinAI/octo #1 (backpressure) and #2 (proactive loop). Read
the code.

- **#1 backpressure** — high-quality, correct (semaphore-slot true Block,
  lock-released-before-await publish, no per-subscriber tasks, kept `subscribe_sync`
  sig). But **duplicates ours** (`03467eb`, already in Albert) and **downgrades
  `Steer`** (which we need). → close. Recorded its semaphore-`Block` design in this
  brief as a **Phase-3 reference** (true never-drop control lane).
- **#2 proactive loop** — strong, additive, loop-safe (`react_proactive`: chat
  hard-excluded + `source==self` guard, filter-union anti-footgun, `NOOP`, per-kind
  dedup history). Its `connectors/scheduler` is a *config-static timer* (`every`/
  `after`/`timer.tick`/`timer.fire`); collides on path/name with our local dynamic
  **alarms/reminders** scheduler (`2b52a44`, `alarm.fired` + `octo.scheduler.*`
  control surface + disk persistence) — **we keep ours and extend it**. → close,
  but **harvest the best code into ours** (router `PayloadPredicate` definite; the
  proactive cogitator loop next, rewired onto our `alarm.fired`/`sensor.anomaly`).

No GitHub token on the box (SSH only) → the user closes both on web with the
drafted comments. Both PRs credited in-thread per plan.

## [2026-06-28 22:04] note | Octo authz stack landed; handoff for Albert's Telegram auth

Built a full authorization stack in octo (outside Albert) and wrote
`octo_authz_handoff.md` as the next session's starting point. Albert path-deps the
sibling octo checkout, so it's all available now (octo commits ahead of origin).

- **Telegram edge ACL + factory** (`053d8c8`): per-chat allow-list dropped before
  the bus (untrusted never reaches cognition); `ChannelMetadata.trust` + `role`
  stamped; connector is now a `type="telegram"` factory for `octo.toml`.
- **octolab config-driven** (`29e1124`): the reference wiring (octo.toml + factory)
  to mirror in Albert.
- **Runtime-mutable ACL** (`4e06b10`): `octo.telegram.{allow_chat,remove_chat,
  list_chats}` control commands (manageable actor); owner-only `/allow` reflex
  (deterministic, no LLM, gated on incoming `role==owner`).
- **`Filter::with_min_trust`** (`8d7a15f`) general trust gate; **`ConnectorContext:
  Clone`**.

Next Albert move: wire authz into its Telegram channel (currently `new` =
allow-all) — config-driven manifest with `owner_chat`, plus porting the `/allow`
reflex into `AlbertCogitator`. Details + APIs on the handoff page.
