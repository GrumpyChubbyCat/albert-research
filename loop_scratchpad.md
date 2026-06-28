# Loop scratchpad — the agent's super-operational working object

**Kind:** scratch / design note
**Created:** 2026-06-28
**Status:** agreed; being implemented

A per-task working object the ReAct loop owns and the agent **sees each turn** —
distinct from the chat transcript and from kaeru. The point: make multi-step
tasks **verifiable**.

## Where it sits (the corrected layering)

```
1. Chat transcript      — Octo history: dialogue + actions, a LOG (linear, append-only)
2. Scratchpad           — super-operational task STATE: explicit subtasks + status   ← this
3. kaeru                — agentic memory, a TOOL: operational + PERSISTENT (recall/jot)
```

kaeru is **operational + persistent** (even its cognitive tier is meant to survive
sessions). The scratchpad is **super-operational** — more volatile than kaeru,
scoped to one task, gone when the task ends. So it is NOT kaeru and NOT chat; it's
the loop's own window.

## Why a scratchpad, not the transcript

The loop already accumulates a transcript (tool calls + dialogue). But a transcript
is a **log** — you can't check "did I finish all parts?". A scratchpad is **state**:

```
goal: <one line>
1. [verified]    did X
2. [done]        did Y          (done = acted; verified = actually checked)
3. [blocked]     need data Z
notes: <findings / decisions>
```

"Done" for the task = **all steps verified**, not "the model felt finished." That
is the verifiability payoff — and the seed of a cheap verify-step on the plain loop
(check all-verified before declaring done), a step toward graph cognition without
the graph.

Also: today Albert's `record()` keeps only `[user, final-answer]` — the intra-turn
tool reasoning is **dropped between turns**. A task spans many messages over time;
the scratchpad carries its state across those turns where the transcript doesn't.

## Shape

- **Content:** `goal`, `steps` (each `{text, status, note?}`), `notes`.
- **Status:** `pending | in_progress | done | verified | blocked`.
- **Scope:** one active scratchpad **per channel** (the current task in that
  conversation). Survives turns within the task; cleared on completion.
- **Storage:** a lightweight in-memory store keyed by channel — **not kaeru**
  (super-operational; losing it on restart is acceptable, the итог was consolidated).
- **Rendering:** the current scratchpad is injected into the preamble every turn
  (like `active_reminders`), so the agent literally sees it.

## Mechanics — agent-authored via tools

The agent writes it (matches "builds itself the object"); the loop renders it back.
Tools: `scratchpad_goal` (set/replace goal, fresh task), `scratchpad_step` (add a
subtask), `scratchpad_mark` (set a step's status; mark `verified` only after an
actual check), `scratchpad_note` (record a finding), `scratchpad_clear` (finish/
abandon — consolidate to kaeru first if durable). Read is implicit (always rendered).

This is the known agent todo/plan/scratchpad pattern (Claude Code's TODO tool, etc.)
— not novel; we just place it correctly: super-operational, distinct from kaeru.

## Boundary with kaeru

Scratchpad **works**; kaeru **keeps**. On task completion the agent `jot`s/`synthesise`s
the durable итог to kaeru, then clears the scratchpad. On resuming a task it can
`recall` from kaeru to rehydrate. Clean seam; kaeru stays a tool.

## Connections

- **Refines:** [[context_memory_layering]] — the scratchpad is a sharper tier-2
  (super-operational), with kaeru kept as the tier-3 tool.
- **Plan:** [[roadmap]] — the immediate next build (lighter than skills or the graph).
