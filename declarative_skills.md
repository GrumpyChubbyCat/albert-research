# Declarative skills — infra, storage, accessibility, visibility

**Kind:** scratch / design note
**Created:** 2026-06-28
**Status:** worked out (design); implementation is the next build after the scratchpad

A **declarative skill** = a named bundle of *instructions* ("how to do X well") the
agent equips when relevant. **No executable code** — that's the separate sandbox/
forkd track. A skill may tell the agent to use existing tools (dispatch / kaeru /
scratchpad / web), but the skill itself is prose, not a function.

This is the Anthropic-style **progressive-disclosure** skill pattern, placed in
Albert's architecture.

## Storage

- Filesystem `skills/` dir; **one dir per skill**: `skills/<name>/SKILL.md`
  (a dir, not a bare file, so a skill can later bundle assets/scripts when it
  graduates to *executable* — without changing the layout).
- `SKILL.md` = frontmatter + body:
  - frontmatter: `name`, `description` (one-line *when to use*), optional `tools`
    hint.
  - body: the markdown instructions.
- Source of truth is the files (user- or Albert-authored). kaeru may *index* skill
  metadata for recall at scale, but does not own them.

## Infra

- A `SkillRegistry` loaded at startup: scan `skills/`, parse frontmatter + body,
  key by name. Held by the cogitator (mind-side — a registry it consults, not an
  organ on the bus).
- Hot-reload (watch the dir) is a later nicety; startup-load first.

## Visibility — how the agent knows what exists

- Render a compact **skills catalog** into the preamble every turn: one line per
  skill, `name — when-to-use`. Cheap, token-bounded. Same mechanism as the
  connector catalog / active reminders / scratchpad render — the agent sees the menu.

## Accessibility — how the agent uses one

- **Progressive disclosure.** The catalog (names + when) is always visible; the
  full `SKILL.md` body is loaded **on demand** via a `load_skill { name }` tool that
  returns the body into the conversation. The agent then follows those instructions.
- A skill is therefore *instructions injected on demand*, not a callable function.
  Keeps context lean — never inject all skill bodies; only the relevant one when
  invoked. (Injecting every body each turn would blow the token budget; the catalog
  is the index, the body is the page — the same structural-recall idea as kaeru.)

## How it sits with the other surfaces

```
skills catalog   — menu of capabilities (names + when), always in view
load_skill(name) — pulls one skill's full instructions on demand
scratchpad       — where the agent tracks executing a (possibly skill-guided) task
kaeru            — where durable итоги of skill use are remembered (tool)
```

A skill guides *how*; the scratchpad tracks *doing it* verifiably; kaeru keeps what
was learned. Clean division.

## Boundary with executable skills

Declarative = instructions only (this note). **Executable = code in a sandbox**
(forkd, separate Capabilities item). A skill dir can later hold both (`SKILL.md` +
scripts), but running scripts waits on the sandbox substrate. Don't conflate.

## Open questions

- **Discovery at scale.** Small N skills → just list the catalog. Many skills →
  skill discovery becomes a retrieval problem (index in kaeru, recall the relevant
  few). Cross that bridge when the catalog gets long.
- **Auto-equip vs explicit load.** Always show the catalog; let the model decide to
  `load_skill`. Explicit load is cleaner and inspectable than auto-injection.
- **Who authors skills.** User-provided files first; later Albert writing its own
  `SKILL.md` (meta — Albert learning a routine and writing it down) is a natural
  extension that pairs with the proactivity routines.

## Bundled resources — files, scripts, assets

A skill is a **folder**, not just a `SKILL.md` — it can ship templates, references,
examples (informational) and, later, scripts / binary assets (fonts, images, data).
Access mode depends on the kind:

- **Informational text** (templates, references) — `skill_file { name, path }` reads
  it **in place** into context (read-only, jailed to the skill's own folder; built
  2026-07-15). `skill_apply` lists a skill's bundled files. No copy into the
  workspace — the agent reads a skill's resources where they live and does its work
  in the separate octo-code workspace.
- **Scripts (to run) + binary assets (fonts/images/data)** — these **cannot go into
  context**; they must exist as files at a real path to be executed / consumed. The
  clean answer is NOT a materialize-into-context tool but, at the forkd/exec stage,
  **mount the skills root read-only into the sandbox** (as OpenClaw mounts workspaces
  ro/rw; as Claude Code runs `.claude/skills/<name>/scripts/*` in place). Then scripts
  run and assets load by path, in place — no copy, no cache. Their real consumer
  (execution) is forkd-era anyway, so nothing to build now.
- **A materialization cache** (stage files locally + cache, LRU like the body cache)
  is warranted **only if skills go remote** (a cloud/S3 skill store) — fetch-once,
  cache by skill+version. For a **local** skills dir + local sandbox the filesystem
  *is* the cache; mount beats materialize.

So `skill_file` covers the informational need now; scripts/assets are a forkd concern
(mount, don't copy); a cache is a remote-skills concern.

## Connections

- **Sibling working surface:** [[loop_scratchpad]] — skill guides the *how*, the
  scratchpad tracks *doing it*.
- **Memory tool:** [[kaeru]] — indexes skill metadata / remembers outcomes; not the
  store of truth.
- **Plan:** [[roadmap]] — declarative skills are the build after the scratchpad;
  executable skills remain on the forkd/sandbox track.
