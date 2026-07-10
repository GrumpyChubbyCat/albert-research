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

## Sources (immutable)

- `sources/sacrarium_snapshot/albert_system_map.md` — PEMRR map.
- `sources/sacrarium_snapshot/octo_runtime.md` — Reaction runtime.
- `sources/sacrarium_snapshot/fluxion.md` — Perception substrate.
- `sources/sacrarium_snapshot/kaeru.md` — Memory substrate.

## Not modelled yet

- Experience / Reflection layers (cognition internals) — beyond current scope.
- Executable skills / sandbox execution (forkd) — deferred (declarative skills are
  designed, see `declarative_skills.md`).
