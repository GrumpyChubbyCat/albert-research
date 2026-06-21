# Albert — Research Initiative

Operational vault for **Albert**: a multitask personal/corporate online assistant,
built as an *assembly* of maturing LamantinAI directions rather than one model.
Albert is the long-term integrative goal; this vault is where its **assembly
design** is worked out — how the pieces compose, where the seams are, what each
layer is allowed to do.

This is the operational tier of the two-tier memory model. Live design happens
here; crystallised outcomes consolidate back to the archival vault (sacrarium).

## The frame: PEMRR

Albert decomposes into five layers (see `sources/.../albert_system_map.md`):

| Layer | Question | Component |
|-------|----------|-----------|
| Perception | what is happening | **Fluxion** (streaming vision → scene events) |
| Experience | what it means now | cognition |
| Memory | what is stored | **kaeru** (cognitive graph + recollection) |
| Reflection | what is learned | cognition |
| Reaction | what is done | **Octo** (the runtime / skeleton) |

Octo is the **Reaction** layer and the structural skeleton the rest plug into.

## Where the baseline lives

`sources/sacrarium_snapshot/` — frozen, read-only snapshots from sacrarium:

- `albert_system_map.md` — the PEMRR decomposition and the assembly target.
- `octo_runtime.md` — Octo: the Reaction-layer runtime (tentacles + core, 5 layers).
- `fluxion.md` — Perception substrate (neuromorphic streaming media AI, Rust).
- `kaeru.md` — Memory substrate (typed cognitive graph, CozoDB, two-tier).

These are immutable. Read first.

> **Scope note:** skills/sandbox execution (forkd & friends) is intentionally
> **out of scope for now** — too far off. Not modelled in this vault yet.

## Where active design goes

- `drafts/` — the synthesis pages (promotion candidates to archival).
- `roadmap.md` — the assembly plan; first integration steps.
- `index.md` — catalog; `log.md` — append-only journal.
- Pages emerge at root, group into typed dirs once a cluster justifies it (`STRUCTURE.md`).

## Conventions

- Schema files (`CLAUDE.md`, `STRUCTURE.md`, `README.md`, `index.md`, `log.md`) — English.
- Work content — language of authorship; Russian stays Russian.
- `[[wikilinks]]` resolve **within this vault only**; cross-vault references (to the
  Octo vault at `~/code/personal/octo/research/`, or to sacrarium) use paths.
- `sources/` is immutable. Consolidation to archival is explicit and logged.
