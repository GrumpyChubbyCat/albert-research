# Index — Albert Vault

Catalog of pages. Thin by design; `log.md` is the authoritative journal.

## Schema

- `CLAUDE.md` — operational vault protocol.
- `STRUCTURE.md` — placement rules.
- `README.md` — what this vault is (+ the PEMRR frame).

## Design

- `drafts/albert_assembly.md` — **the assembly contract**: Octo as Reaction
  skeleton, component placement (Fluxion connector / kaeru rig-tool /
  graph-flow cogitator), layering, Octo-side gaps. *(draft)*
- `roadmap.md` — assembly plan; current front = GraphCogitator.
- `agent_runtime_landscape.md` — **competitive survey**: 5 "claw" agent runtimes
  (OpenClaw + picoclaw/goclaw + openhuman/moltis); what to borrow, where Albert
  is already ahead. *(scratch / survey)*
- `context_memory_layering.md` — **three tiers, three owners**: hot context (Octo
  history) / staged cognition state (graph-llm checkpointer) / deliberate memory
  (kaeru). Checkpoint+steering = graph-native. *(scratch / design note)*
- `octo_bus_backpressure_fix.md` — **fix brief + outcome**: the one real Octo-core
  gap for the GraphCogitator (silent `Lagged` drop); phased fix, acceptance
  criteria, files. *(task — done, octo `03467eb`)*
- `octo_authz_handoff.md` — **handoff / start here**: the Telegram authorization
  stack landed in octo (edge ACL + factory, runtime-mutable ACL via control
  commands, `/allow` reflex, trust floor); Albert's next move is to wire it into
  its own Telegram channel. *(context)*
- `octo_caldav_handoff.md` — **handoff**: the generic CalDAV connector + reusable
  `octo-http-auth` landed in octo (discovery from a server root, list / create /
  delete events, factory `type = "caldav"`, multi-account); Albert's next move is
  to enable and configure a calendar (Yandex). *(context)*
- `loop_scratchpad.md` — **the agent's super-operational working object**: per-task
  state the loop owns + renders each turn; makes multi-step tasks verifiable.
  *(design — built in `albert/src/scratchpad.rs`)*
- `declarative_skills.md` — **declarative skills design**: storage / infra /
  visibility / accessibility, progressive disclosure (catalog + `load_skill`).
  *(design — implementation next)*
- `octo_connectors_handoff.md` — **handoff for an octo session**: a **generic
  CalDAV** connector (`connectors/caldav/`, one crate → many calendars: Yandex now,
  Google via OAuth2 later) + SMTP send (`connectors/smtp/`); env-as-tools organs,
  zero cogitator change. *(task — open, octo-side)*
- `forkd_isolation_architecture.md` — **phase-3 design**: forkd = a supervised
  connector (not in-process) running untrusted scripts; two sandboxes (host←agent,
  agent←code), three isolation layers (L1/L2/L3), subprocess+bwrap **not WASM**,
  nl-vmnano deploy. *(design — forkd parked until untrusted code)*
- `lamantin_four_pillars.md` — **productization requirements**: SQLite per channel,
  router-enforced multitenancy, per-user isolation, router as access authority; how
  each maps onto Octo's edge ACL / octo-history / forkd L3. *(architecture — mostly
  forward-looking)*
- `zendriver_browser.md` — **library candidate**: Rust-native stealth browser, the
  alternative to a Playwright(Node) sidecar for the web-parser organ (runs under
  forkd). *(scratch — pick at build time)*
- `octo_files_and_workspace_handoff.md` — **handoff / integration task**: octo-code
  file tools + storage connector (put/get/list/delete + promote/checkout) + Telegram
  message coalescing (`InboundMessage`) + file transfer, all over a shared workspace
  (`octo-workspace`). Bytes by reference, never through the model; Albert wires the
  root + factories + two cogitator arms. *(context)*

## Sources (immutable)

- `sources/sacrarium_snapshot/albert_system_map.md` — PEMRR map.
- `sources/sacrarium_snapshot/octo_runtime.md` — Reaction runtime.
- `sources/sacrarium_snapshot/fluxion.md` — Perception substrate.
- `sources/sacrarium_snapshot/kaeru.md` — Memory substrate.

## Not modelled yet

- Experience / Reflection layers (cognition internals) — beyond current scope.
- Executable skills / sandbox execution (forkd) — **designed, parked** (see
  `forkd_isolation_architecture.md`); declarative skills built, see
  `declarative_skills.md`.
