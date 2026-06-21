# CLAUDE.md — Operational Memory Vault

Operating protocol for an LLM agent working inside this **operational memory vault**.
This file is **schema**, not content. Read it before any non-trivial action.

## What This Vault Is

This vault is a **cognitive apparatus** — a full graph-based working vault scoped to a specific topic, task, or research initiative. It is where the agent actually thinks: typed nodes, typed edges, wikilinks, curator-maintained `index.md` and `log.md`, `## Connections` blocks. It is the **operational tier** of a two-tier memory design: operational (here) + archival (a separate persistent wiki, e.g. sacrarium).

Same LLM-wiki mechanics as archival (per Andrej Karpathy, gist `442a6bf555914893e9891c11519de94f`). What differs is **scope and lifetime**, not rigor:

- Topic-scoped: one vault per initiative/task, not a global catalog.
- High content volume during active work: this is where most of the reasoning surface gets materialised as pages — hypotheses, experiments, observations, scratch ideas, task state, episodic stream, roadmap, dense edges between all of it.
- First-class operational artifacts on top of the usual types: `task`, `checklist`, `roadmap`, `experiment`, `hypothesis`, `scratch_concept`, `episode`, `draft`.
- Bounded lifetime: when the topic's research concludes, итоги and предпосылки promote to archival; the vault becomes a historical snapshot or is retired.
- Consolidation is a one-way, explicit, logged promotion to archival, preserving `derived_from` provenance across the tier boundary.

## Relationship to the Archival Vault

**This vault produces. The archival vault preserves.**

- Work happens here: reasoning, iteration, scratch, draft, experiment, retry.
- When a concept, entity, summary, or conclusion is **ready** (итог + предпосылки), it is **promoted to archival** through consolidation. Operational retains a `consolidated_to` pointer.
- Operational artifacts (tasks, checklists, roadmaps, experiments) **never promote**. They complete in place and are archived locally or discarded.
- Archival is the source of truth for settled knowledge. If operational needs to reason about a settled fact, it reads from archival.

Wikilinks **do not cross the tier boundary**. Cross-vault references use paths or URLs, not `[[wikilinks]]`.

## Three Layers

Same spirit as the Karpathy LLM-wiki pattern:

1. **Raw sources** (`sources/`) — immutable first-hand material. Never edited or deleted.
2. **Work** — operational artifacts and provisional knowledge the agent maintains. Most of this vault.
3. **Schema** — this file + `STRUCTURE.md` + `README.md`. Rules, not content.

## Language Policy

- Schema files (this file, `STRUCTURE.md`, `README.md`, `index.md`, `log.md`, directory READMEs) — **English**.
- Work content — matches the language of authorship. Russian research stays in Russian.
- Connections blocks when used — always **English** (see Page Types).

## Directory Map

Placement rules in `STRUCTURE.md`. The short version: **start flat, let structure emerge.** Only `sources/` is pre-required. Typed directories (for tasks, experiments, hypotheses, drafts, etc.) appear when a cluster of pages of that kind accumulates enough to warrant grouping — not before.

Required at root:

- `sources/` — raw source material (immutable)
- `index.md` — catalog
- `log.md` — journal

Emergent (create when needed, use the type name as directory name):

- `roadmap/`, `tasks/`, `experiments/`, `hypotheses/`, `scratch/`, `episodes/`, `drafts/` — each collects pages of the corresponding type from § Page Types below, once you have enough of them
- `archive/` — local retirement for completed work that didn't promote

**`drafts/` is slightly special:** even a single draft benefits from being in `drafts/` — drafts are the only promotion candidates to archival, and keeping them isolated makes consolidation passes cleaner.

## Special Files

### `index.md`

Thin catalog. Update as pages stabilise. Operational vaults move fast — `index.md` may lag, and that is OK. `log.md` is the authoritative journal.

### `log.md`

Append-only. Every non-trivial change logged.

Header: `## [YYYY-MM-DD HH:MM] <op> | <short title>`

Ops: `ingest`, `note`, `task`, `experiment`, `consolidate`, `discard`, `reorg`, `lint`.

Time-of-day is **required** here (unlike archival, which uses date only) because within-day ordering matters at operational churn rates.

### `sources/`

Raw material. Agents: read freely, never modify. On ingest, produce a summary in `drafts/`. Initially empty when the template is cloned; drop relevant material here as the initiative starts.

## Page Types

All page types below are operational-first. They are **concepts** — shapes a page can take — not directory assignments. Write a page where it makes sense to put it; promote to a typed directory only when the type cluster grows (see `STRUCTURE.md`).

Where an archival template is referenced (draft section), follow the archival vault's CLAUDE.md conventions.

### Task

```markdown
# <task title>

**Status:** open | in_progress | blocked | done | abandoned
**Created:** YYYY-MM-DD
**Closed:** YYYY-MM-DD (when status is done/abandoned)
**Blocks:** [[other_task]]
**Part of:** [[roadmap_page]] | [[checklist_page]]

## Goal
One paragraph.

## Subtasks
- [ ] ...
- [x] ...

## Notes
Freeform.

## Outcome
Written when closed. If it produced a promotable итог, link to the archival target once consolidated.
```

### Checklist

Lightweight bundle of subtasks grouped by purpose. Often part of a task or roadmap.

```markdown
# <checklist title>

**Part of:** [[task_or_roadmap]]

- [ ] ...
- [ ] ...
```

### Roadmap

```markdown
# <roadmap title>

**Scope:** <what it covers>
**Status:** active | superseded | done
**Supersedes:** [[old_roadmap]]

## Current phase
...

## Phases
1. ...
2. ...

## Open fronts
- ...
```

Roadmap revisions are frequent. Keep revision history in `log.md`; do not preserve every revision in the page.

### Experiment

```markdown
# <experiment title>

**Targets:** [[hypothesis_page]]
**Status:** planned | running | done | abandoned
**Started:** YYYY-MM-DD
**Completed:** YYYY-MM-DD

## Method
...

## Prediction
...

## Result
Filled when done.

## Conclusion
`verifies` | `falsifies` | `inconclusive` for the target hypothesis.
```

### Hypothesis

```markdown
# <hypothesis>

**Status:** open | supported | refuted | inconclusive
**Targeted by:** [[experiment_pages]]

## Claim
One or two sentences.

## Why this matters
...

## Expected consequences
- ...
```

### Scratch

Free-form, no required structure. Name by topic. If it matures, rewrite as a first-class page (hypothesis, draft concept) or delete.

### Episode

```markdown
# [YYYY-MM-DD HH:MM] <short title>

**Kind:** chat | observation | action | decision
**Significance:** low | medium | high
**Informs:** [[pages this event influenced]]

Freeform body.
```

### Draft (concept / entity / summary)

Provisional version of an archival page. Use the archival vault's page templates — see its `CLAUDE.md`.

Required fields on drafts:
- `**Status:**` = `draft` | `ready_for_consolidation` | `consolidated`
- `**Target archival path:**` (once known)
- `**Consolidated to:**` (once promoted)
- `## Connections` block — required on drafts (archival requires it; drafts inherit this obligation pre-emptively)

Drafts are the **only** direct promotion candidates into archival. Keep them in `drafts/` once you have any — even one — so consolidation passes scan one place.

## Edge Semantics

Typed edges are encoded by prose phrasing + section placement (same approach as the archival wiki). Common ones in operational context:

- `refers_to` — basic wiki reference.
- `part_of` — task part of roadmap; checklist item part of checklist.
- `blocks` — task blocks task.
- `targets` — experiment targets hypothesis.
- `verifies` / `falsifies` — experiment result → hypothesis.
- `derived_from` — claim / concept → source or episode. **Critical for consolidation** — this chain survives the tier boundary.
- `supersedes` — iteration replaces earlier version.
- `consolidated_to` — operational draft → archival page (after promotion). One-way, one-shot.

## Operations

### Ingest

Adding a new source:

1. Place the file in `sources/`.
2. Read it.
3. Write a summary page (Summary draft — see Page Types) following the archival summary template.
4. Cross-reference existing drafts/hypotheses that this source informs.
5. Log under op `ingest`.

### Scratch-first

Do not stall on structure. A new idea can start as a scratch page anywhere. Over time:

- matures → rewrite as a draft (concept/entity) or a hypothesis page
- dead end → delete, with a one-liner in `log.md` under op `discard`

### Experiment cycle

1. Write a hypothesis page for the claim.
2. Write an experiment page, linked via `Targets: [[hypothesis_page]]`.
3. Run; record as you go.
4. Write Result + Conclusion.
5. Update hypothesis status.
6. If supported and stable, write the hypothesis-as-concept as a draft, queue for consolidation.

### Consolidate (promotion to archival)

The critical interface. This is a deliberate, logged operation — never silent.

**Trigger:** reflection step at end of task/experiment, explicit user request, or when a draft has been stable for N iterations.

**Protocol:**

1. Identify the **итог**: the specific concept, entity, or summary that is ready.
2. Identify the **предпосылки**: the `derived_from` chain that supports it — sources, episodes, experiments, concepts it rests on.
3. In the archival vault, create or update the target page per archival's CLAUDE.md. Include the provenance chain as `derived_from` wikilinks (if target archival has the source pages) or as path references.
4. In the operational draft, set `status: consolidated` and `consolidated_to: <archival path>`.
5. Log under op `consolidate`: source draft, archival target, what promoted.
6. Optional: after a retention window, delete the operational draft. The `consolidated_to` pointer is the authoritative trace.

**What promotes:**
- a stable concept draft
- a verified hypothesis (as a concept)
- a source summary
- a canonical entity discovered during research

**What does not promote:**
- operational artifacts: tasks, checklists, roadmaps, experiments — they complete locally
- failed experiments — log the result, do not promote as positive knowledge
- scratch that didn't mature
- intermediate reasoning that didn't survive

### Discard

When a scratch or draft is clearly dead:

1. Delete or move to `archive/discarded/`.
2. Log one line under op `discard` with the reason.

### Task completion

1. Update status to `done` or `abandoned`.
2. Write the Outcome section.
3. If the task produced a promotable итог, run consolidate.
4. Keep the task in place with final status, or move it to an `archive/` subtree once tasks pile up.
5. Log under op `task`.

### Lint

Operational lint is lighter than archival. Check:

- tasks stuck in `in_progress` > N days
- hypotheses open with no experiments targeting them
- drafts stable for N iterations — candidates for consolidate
- scratch older than M days not promoted — candidates for discard
- dead wikilinks
- drafts missing `Connections` block (drafts require it — see Page Types)

Report, propose, apply after confirmation.

## Retention

This vault's lifetime is bounded. At end of the initiative/project:

- Promoted items are already in archival → safe to discard here.
- Completed tasks: archived locally or deleted.
- Roadmaps marked `done` or `superseded`: keep for history.
- Scratch and drafts that never promoted: review, promote last-minute candidates, discard the rest.
- Whole vault: archived read-only as a tarball in the archival vault's `sources/`, or deleted — policy choice per initiative.

## Conventions

Filenames: lowercase, English transliteration, underscores. Same as archival.

Wikilinks: Obsidian-style `[[filename]]`. Resolution scope is **this vault only**. Cross-vault references use paths or URLs.

Connections block: **optional on most operational pages**, **required on drafts**. When used, keep it English (structural element).

## What To Do Before Editing

- Read this file.
- Read `STRUCTURE.md`.
- Skim the latest entries in `log.md` to know what was just touched.
- For changes that touch >5 pages: state the plan first, wait for confirmation.

## What Not To Do

- Do not edit anything under `sources/`.
- Do not silently promote to archival — consolidation is explicit and logged.
- Do not mirror archival's strict schema everywhere — operational allows provisional work by design.
- Do not treat `index.md` or `log.md` as scratch space.
- Do not fabricate wikilinks to pages that don't exist.
- Do not use wikilinks across the tier boundary — they won't resolve.
- Do not backfill `log.md` with invented historical entries.
