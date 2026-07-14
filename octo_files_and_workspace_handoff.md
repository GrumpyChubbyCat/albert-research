# Octo files, storage & message-batching — handoff for Albert

**Status:** integration task — **octo-code DONE** (2026-07-15), storage + telegram-files PENDING
**Progress:**
- [x] (1, partial) one workspace root wired for octo-code (`albert.toml [code]
  workspace` -> `$OCTO_CODE_WORKSPACE`, created on startup).
- [x] (2) octo-code file tools live in the cogitator (`code_tools!`, git-dep on
  octo-code at `c69471e`).
- [ ] (3) storage connector — deferred (base_dir-vs-config-dir workspace-path
  coordination to verify).
- [ ] (4) telegram manifest `workspace`.
- [ ] (5) cogitator `InboundMessage` arm + `chat.send_file` — deferred (vision decision).
**Created:** 2026-07-15
**Part of:** [[roadmap]] (phase 2 file workspace + capabilities)
**Builds on:** [[file_code_tooling]], [[forkd_isolation_architecture]]
**Start here:** A whole file/workspace/storage stack landed in octo. Albert can
now give the model file tools, a durable store, and receive/send files over
Telegram — but the pieces must be enabled and wired. This is that wiring.

Albert **path-depends** on the sibling octo checkout, so all of this is already
available; the octo commits are on `main` (pushed).

## What landed in octo

- **`octo-code`** (`b3ff7dc`) — file tools as rig `Tool`s, **not** a bus
  connector: `read` / `write` / `edit` (string-replace) / `list` / `glob` /
  `grep`, jailed to a workspace root (`$OCTO_CODE_WORKSPACE`). Wired into
  `octo-rig` behind the **`code` feature**; register them all with the
  `code_tools!` macro. Errors returned as data so the model self-corrects.
- **`octo-connector-storage`** (`fd14c07`) — a durable object-store organ
  (env-as-tools). `storage.put/get/list/delete` over string keys, plus
  `storage.promote` (shelf a workspace file) / `storage.checkout` (bring one
  back). Backend swappable behind `StorageBackend` — **local dir now, S3 later**.
  Factory `type = "storage"`.
- **Telegram message coalescing** (`7ed7f8d`) — forwarding several messages (or
  an album) now reaches cognition as **one** input, not N turns. Signal-driven:
  normal chat pays zero latency; coalescing kicks in only for an album
  (`media_group_id`) or a forwarded burst (`forward_origin`), 500 ms window. A
  mixed text+photo burst arrives as the new `octo_core::InboundMessage { text,
  images }`; a lone message still travels as `String` / `Blob`.
- **Telegram file transfer** (`55d6db9`) — inbound documents saved to the shared
  workspace `inbox/` with a reference message emitted; outbound `chat.send_file
  { path, chat?, filename? }` sends a workspace file as a document. **Bytes move
  by reference, never through the model.**
- **`octo-workspace`** (`c69471e`) — the shared path-jail + atomic-write
  primitive (`components/workspace`), used by all three of the above. One place
  to audit confinement.

## The one principle: bytes by reference, never through the model

The **shared workspace** (one directory) is the byte-exchange medium; the model
only ever names paths/keys. octo-code, storage (promote/checkout) and telegram
(inbound/outbound files) all agree on that root — so coordinate it (below).

```
you send a file  -> telegram saves inbox/report.pdf -> model: read/edit -> storage.promote -> shelf
shelf            -> model: storage.checkout inbox/x -> chat.send_file    -> file arrives to you
```

## What Albert should do

### 1. Coordinate ONE workspace root
Pick a durable-but-scratch dir (e.g. `/opt/albert/state/workspace`) and make all
three agree:
- export `OCTO_CODE_WORKSPACE=<dir>` (octo-code reads it),
- storage manifest `workspace = <dir>` (for promote/checkout),
- telegram manifest `workspace = <dir>` (for file transfer).

### 2. Enable octo-code (file tools)
- `albert/Cargo.toml`: `octo-rig = { path = "...", features = ["code"] }`.
- When building the rig agent, add the tools in one line:
  ```rust
  let agent = octo_code::code_tools!(client.agent(model).preamble(p)).build();
  ```
  (or `octo_rig::{ReadTool, WriteTool, …}` individually).

### 3. Register the storage factory + manifest
```rust
builder = builder.register_connector_type("storage", octo_connector_storage::factory());
```
```toml
# config/connectors/storage/storage.toml
[connector]
id = "storage"
type = "storage"
backend = "local"
root = "/opt/albert/state/artifacts"   # durable, backed up — NOT the workspace
workspace = "/opt/albert/state/workspace"
```
The model reaches it via the existing dispatch tool (env-as-tools), zero
cogitator change.

### 4. Telegram manifest: add `workspace`
Add `workspace = "/opt/albert/state/workspace"` to Albert's telegram manifest so
inbound documents land where octo-code can read them and `chat.send_file` can
read them back. (Batching defaults are fine; tune with `batch_debounce_ms` /
`batch_max_wait_ms`, `0` disables.)

### 5. Cogitator wiring (two small things)
- **`InboundMessage` arm** — handle the coalesced multimodal payload, mirroring
  octolab: after the `String` / `Blob` arms,
  ```rust
  if let Some(msg) = incoming.payload_as::<InboundMessage>().cloned() {
      let text = msg.text.unwrap_or_default();
      let image = msg.images.into_iter().next(); // or handle all (vision)
      self.respond(incoming, text, image, ctx).await;
  }
  ```
  Without it, mixed text+photo bursts are ignored.
- **`chat.send_file` trigger** — to send a file to the user, the cogitator emits
  a `chat.send_file` envelope targeted at the telegram connector, `channel =
  the current chat`, payload `{ path }` (chat also accepted in the payload). Give
  the model a small tool/reflex that maps "send this file" → that envelope.

## Command / API surface

- octo-code tools (behind `code`): `read`/`write`/`edit`/`list`/`glob`/`grep`,
  paths relative to `$OCTO_CODE_WORKSPACE`.
- storage (dispatch to its id):
  - `storage.put { key, content | content_base64 }` → `{ key, bytes }`
  - `storage.get { key }` → `{ key, content | content_base64, encoding, bytes }`
  - `storage.list { prefix? }` → `{ keys }`
  - `storage.delete { key }` → `{ deleted }`
  - `storage.promote { workspace_path, key }` → `{ key, bytes }`
  - `storage.checkout { key, workspace_path }` → `{ workspace_path, bytes }`
- telegram outbound: `chat.reply` (text/`Blob` as before) + `chat.send_file
  { path, chat?, filename? }`.
- payloads: `octo_core::InboundMessage { text: Option<String>, images: Vec<Blob> }`.

## Open / decide next

- **Images in `InboundMessage`** — octolab takes only the first; Albert (vision)
  should handle all `images`.
- **Multiple images / files in a batch** — coalescing covers album photos into
  `images`; forwarded *documents* still emit one reference message each (no
  `files` field on `InboundMessage` yet). Add if needed.
- **S3 backend** — `StorageBackend` is ready; the S3 impl (rust-s3/aws-sdk) is
  the productization step. Manifest shape sketched in `storage.toml`.
- **Google CalDAV / SMTP** — unrelated, tracked in [[octo_caldav_handoff]] /
  [[octo_connectors_handoff]].

## Connections

- **Plan:** [[roadmap]]. **Sibling handoffs:** [[octo_authz_handoff]],
  [[octo_caldav_handoff]].
- **Code:** octo at `~/code/personal/octo/octo/` (commits `b3ff7dc` `fd14c07`
  `7ed7f8d` `55d6db9` `c69471e` on `main`, pushed); Albert at
  `~/code/personal/albert/albert/`.
