# Structure Rules — Operational Memory Vault

Placement rules for this vault. Companion to `CLAUDE.md`.

## Core Principle

**Minimum up front, structure emerges.** The vault starts flat. A subdirectory appears only when a cluster of pages of the same kind has accumulated enough to justify grouping — not before.

Typed page **concepts** (task, checklist, roadmap, experiment, hypothesis, scratch, episode, draft) are defined in `CLAUDE.md § Page Types`. They are ways to **shape a page when you write one**, not directories you must fill.

Create a typed directory when answering "which of my open hypotheses have no experiment?" by eyeballing the root becomes annoying. Until then, the root is fine.

## Required Files and Directories

Only these are pre-required:

- `CLAUDE.md`, `STRUCTURE.md`, `README.md` — schema.
- `index.md` — catalog.
- `log.md` — journal.
- `sources/` — raw first-hand material. **Immutable** — never edited by the agent. One subfolder per topic if useful; flat is fine.

Everything else is emergent.

## Emergence Heuristics

- **Pages at root** until a type cluster shows up.
- **Create a subdirectory** when you have ≥3 pages of the same type and scanning the root for them gets tedious. Use the type name: `experiments/`, `hypotheses/`, `roadmap/`, etc.
- **Sub-group inside a typed directory** only when the type itself grows — group by sub-initiative, not by date.
- **Do not pre-create empty directories.** Git doesn't track them anyway.
- **`drafts/` is a special case**: when you have any draft (even one), put it in `drafts/` — drafts are the only promotion candidates to archival, and keeping them isolated makes consolidation passes cleaner.
- **`archive/` as retirement** — completed tasks, retired roadmaps, discarded scratch. Appears when you want to retire things without deleting.

## Naming

- Lowercase.
- English transliteration or English.
- Underscores, not spaces.
- Topic, not mood or date.
  - Exception: episodes start with timestamp for chronological scan — `YYYY-MM-DD_HHMM_<slug>.md`.
- Name for a six-months-later reader.

Good: `octo_memory_writer_refactor.md`, `consolidation_trigger_policy.md`.
Less good: `idea.md`, `thursday_notes.md`, `important.md`.

## When Unsure

Put it at the root and note the intent in `log.md`. Classification can happen later — that's the whole point of operational tier.

Do not invent a new top-level directory without thinking about whether it will still be useful in a month.

## Maintenance

- Keep the root minimal — schema files + `index.md` + `log.md` + `sources/` at a minimum; typed directories only where they've earned their place.
- Directory-level `README.md` appears only where the directory has local conventions worth stating beyond what's here.
