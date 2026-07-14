# forkd & isolation — sandboxing architecture (roadmap, phase 3)

**Kind:** research note / design conclusions
**Informs:** [[roadmap]] (phase 3 forkd / sandboxed execution), [[file_code_tooling]],
[[declarative_skills]]
**Builds on:** [[openclaw_code_fs]], [[lamantin_four_pillars]]
**Provenance:** distilled from the kaeru `lamantin-ai` vault export
(`sources/lamantin-ai-export.tgz` → `forkd-octo-execution-layer`,
`isolation-forkd-sandboxing-architecture`), woven in here so it lives in the
roadmap rather than a tarball.
**Status:** settled design; forkd itself PARKED until Albert runs untrusted code.

## Two DISTINCT sandboxes — never conflate

- **Agent-sandbox** — protects the **HOST from the agent** (and from anything that
  compromises the agent). This is L1 below.
- **forkd / script-sandbox** — protects the **AGENT from the code it runs**
  (untrusted, model-generated scripts). This is L2 below.

They are different directions of trust. A container gives you the first and **not**
the second: a script inside can still trash the agent's own kaeru vault.

## forkd is a CONNECTOR, not an in-process tool

Its whole value is isolation + supervision + a **fault boundary**. Running scripts
as an in-process rig tool would let a hung/rogue script **wedge the cogitator's
tool-loop**. So forkd is a supervised Octo connector: the cogitator dispatches
`forkd.run` → forkd runs the script in a sandbox **in its own task** → returns
`forkd.result`. The LLM reaches it through the existing `dispatch_to_connector`
([[file_code_tooling]] octo-rig bridge); a dedicated `run_script` tool would just be
sugar over that.

Contrast with **kaeru / scratchpad / octo-code file tools**, which ARE in-process
tools — correct, because they're local & trusted, no isolation needed. forkd
differs *precisely because it runs untrusted code*. (This mirrors how Claude Code
itself executes: bash never runs as a bare in-process tool — a bubblewrap sandbox +
permission gate + hook-veto always sits between the model and the host.)

- **Substrate vs door.** The sandbox *mechanism* (bwrap/nsjail + drop-privs + cwd
  logic) is a **library** — this is "forkd, the lowest execution layer, not a
  connector." It is *wrapped AS a connector* so Octo can call & supervise it. Both
  statements are true; they're different layers.
- **bwrap itself** is neither connector nor tool — it's the mechanism the sandbox
  uses.
- **MVP shortcut:** start with an in-process `run_script` tool, promote to a
  connector later; the LLM-facing contract is identical.

## Decision: real scripts → subprocess sandbox, NOT WASM

Scripts are python/bash = **real interpreter processes** → the substrate must be
**subprocess + a Linux sandbox** (bwrap / nsjail / landlock / seccomp / cgroups),
not WASM. WASM can't host a python/bash interpreter, so "WASM-first" only ever
covers a constrained calc-like DSL, never Albert's actual `bash`/`exec` need.

**Reconciling with the roadmap's "WASM-first → full runner" wording:** these are
two different capabilities, not two tiers of one. If the goal is *running real
shell/python* (it is), skip WASM and go straight to the subprocess+bwrap runner.
Keep WASM only if a separate, sandboxed *pure-compute* faculty is ever wanted.

## THREE isolation layers — defense in depth, complementary

None replaces another.

- **L1 — infra (host ← agent):** systemd-hardening OR a container (Docker/podman).
- **L2 — code/scripts (agent ← the scripts it runs):** forkd → bwrap. **v0** =
  drop-uid + `cwd = scratch-dir` + `/tmp` (this is exactly the "/tmp or its own
  folder" confinement — forkd v0, not a competitor to it).
- **L3 — per-skill capabilities (script ← resources):** `net` / `fs_rw` / `timeout`
  / `mem`, declared per skill in `forkd.toml`; forkd grants the **minimum** and the
  sandbox enforces. This ties to the [[lamantin_four_pillars]] "access checking by
  the router" — **forkd is the router for code**.

**KEY:** L1 does NOT replace L2. A container confines the agent from the host, but
a script inside can still corrupt the agent's own data → code-level isolation (L2)
is needed even inside Docker.

## Deploy context (nl-vmnano)

Albert runs as a **root systemd service on the bare host, no container**. Two L1
options:

- **Harden the systemd unit** (systemd IS a sandbox you already have): `User=`
  (drop root — the main win), `NoNewPrivileges`, `ProtectSystem=strict`,
  `ProtectHome`, `PrivateTmp`, `ReadWritePaths=/opt/albert/{state,kaeru}`,
  `RestrictNamespaces`, `RestrictAddressFamilies`,
  `SystemCallFilter=@system-service`, `CapabilityBoundingSet=`. Need to chown
  `state/` & `kaeru/` to the new user. **~80% of the safety for ~20% effort**, no
  image pipeline.
- **Containerize** (no systemd inside → the container IS the boundary): non-root
  `USER`, `--read-only` rootfs, `--cap-drop=ALL`, no-new-privileges, no
  `--privileged`, never mount `docker.sock` or sensitive host paths; kaeru+state as
  named volumes (backed up); `.env` via secret. Orchestration can still be a systemd
  unit running `podman run`. Cleaner/more portable, but adds build→image→run.

"Shouldn't break its own container" has two meanings: (a) **escape to host** —
prevented by proper container config; (b) **corrupt its OWN data** — NOT prevented
by the container (needs read-only code mount + a backed-up data volume, i.e. still
L2).

**Nested-sandbox caveat (later):** bwrap / user-namespaces INSIDE an unprivileged
container is fiddly (needs nested userns, e.g. rootless podman, or run scripts in a
sibling ephemeral container).

## Shared workspace handoff — octo-code ⇄ forkd

The in-process file tools ([[file_code_tooling]], `octo-code`) and forkd operate
on **one directory, two ways**. octo-code writes to the workspace root
(`$OCTO_CODE_WORKSPACE`, e.g. `/tmp/octo-code/<session>`); forkd, when it runs a
script, **bind-mounts that same directory** into the sandbox as the only writable
mount, so a script's `sed -i foo.txt` edits the very file the tools `read`/`write`:

```
bwrap --unshare-all --die-with-parent \
  --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib \  # interpreter, ro
  --proc /proc --dev /dev \
  --bind $OCTO_CODE_WORKSPACE /work --chdir /work \              # the SAME dir, rw
  -- sed -i 's/foo/bar/' notes.txt
```

So the shared workspace **is the handoff surface** between the two faculties: a
synchronous, path-jailed file tool and a sandboxed exec, both over the same tree.

Design points to honor when building forkd:

- **One root, agreed by config.** `$OCTO_CODE_WORKSPACE` (file tools) and forkd's
  bind-source must be the same path — a single "session workspace". Reinforces the
  ephemeral-scratch vs durable-storage split ([[file_code_tooling]]): the shared
  workspace is throwaway; finished artifacts go to the storage connector.
- **The sandbox is STRONGER than the lexical jail, not weaker.** octo-code's guard
  is a Rust path-string check; forkd's is the kernel — `/etc`, `/home` aren't even
  in the mount namespace, only `/work` is writable. A script physically cannot
  `sed /etc/passwd` (the path doesn't exist inside). Bind-mounting doesn't pierce
  the jail; it replaces it with a harder one.
- **No races.** The cogitator drives tools sequentially in its tool-loop, so
  `write` and `forkd.run` never touch a file concurrently; octo-code's atomic
  temp+rename writes help regardless.
- **v0 nuance.** forkd v0 (drop-uid + cwd=scratch, no mount-ns) sees the workspace
  simply because it is a real host dir. The moment full bwrap with a mount
  namespace is enabled, the workspace must be **explicitly bind-mounted** in or it
  vanishes from the sandbox — the bind is the mechanism.

## Where this sits on the roadmap

- **Now (phase 2):** file tools are safe by folder-confinement alone — no forkd.
  See [[file_code_tooling]].
- **Phase 3 (this note):** `bash`/`exec` + executable skills converge on forkd as
  the one sandboxed-execution substrate. An **executable skill = declarative
  SKILL.md ([[declarative_skills]]) + scripts run in forkd**.
- **Browser:** [[zendriver_browser]] is a candidate browser faculty wired as a
  connector *on top of* forkd.
