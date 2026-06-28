# Octo authz update — handoff for the next Albert session

**Status:** context / starting point
**Created:** 2026-06-28
**Part of:** [[roadmap]]
**Start here:** Albert's Telegram channel currently has **no authorization** — the
first move of the next session is to wire in the access control that octo now
provides.

This page hands off a batch of octo work (security + config + bus) done outside
Albert. Albert **path-depends** on the sibling octo checkout
(`albert/Cargo.toml` → `../../octo/octo/...`), so all of this is **already
available to Albert** — no push needed. (The octo commits below are on octo's
local `main`, ahead of `origin`; push when convenient.)

## What landed in octo (relevant to Albert)

Authorization stack for the Telegram connector (the headline):

- **Edge ACL + factory** (`053d8c8`): the Telegram connector enforces a per-chat
  allow-list — a message from an unlisted chat is **dropped before the bus**, so
  untrusted input never reaches cognition (the prompt-injection boundary, and
  "don't even respond in unauthorized chats"). Listed chats get
  `ChannelMetadata.trust` + a `role` tag stamped on the envelope. The connector is
  now a **factory** (`type = "telegram"`), so it joins the `octo.toml`
  config-driven assembly. Token stays a secret in the env; the ACL is a JSON file
  named by the connector's manifest.
- **octolab went config-driven** (`29e1124`): reference for how to assemble the
  Telegram connector from `octo.toml` + `register_connector_type` + a per-connector
  `telegram.toml`. Mirror this in Albert.
- **Runtime-mutable ACL** (`4e06b10`): the connector accepts
  `octo.telegram.{allow_chat,remove_chat,list_chats}` control commands (manageable
  actor, like the scheduler), mutates the list behind a lock, persists it
  atomically. octolab gained an **owner-only `/allow <id>` / `/deny <id>` /
  `/allowed`** reflex — a *deterministic* command (no LLM in a security action)
  that checks the incoming message's `role == owner` before dispatching.

Two general primitives also landed:

- **`Filter::with_min_trust(TrustLevel)`** (`8d7a15f`): a subscription can require
  a minimum channel trust; channel-tagged envelopes below it are filtered out,
  internal/system traffic (no `channel_metadata`) passes. A general runtime trust
  gate — defense-in-depth on top of the connector's edge drop.
- **`ConnectorContext` is now `Clone`** (in `4e06b10`): a connector can hand a
  publish/subscribe handle to a spawned helper task.

(Earlier, already known: bus backpressure with `Steer` + `subscribe_options`
[[octo_bus_backpressure_fix]], and the router `PayloadPredicate` numeric
thresholds — both also available.)

## What Albert should do (the actual task)

Albert's `main.rs` builds the channel as `TelegramConnector::new("telegram",
token)` — **allow-all, no auth**. `new` is unchanged, so Albert still compiles;
authz is opt-in. Add it:

**Recommended — config-driven (mirror octolab):**
1. Add `albert/config/octo.toml` (`[connectors] dir = "connectors"`) and
   `albert/config/connectors/telegram/telegram.toml` with:
   ```toml
   [connector]
   id = "telegram"
   type = "telegram"
   token_env = "OCTO_TELEGRAM_TOKEN"
   acl_path = "telegram_acl.json"
   owner_chat = <YOUR real chat id>   # so the bot answers you on first run
   ```
2. In `main.rs`, when a token is present:
   ```rust
   builder = builder
       .register_connector_type("telegram", octo_connector_telegram::factory())
       .from_config_file(ALBERT_OCTO_TOML)?;   // add ConfigError -> albert::Error
   ```
   instead of `add_connector(TelegramConnector::new(...))`. Console fallback stays.
3. Port the **owner-gated `/allow` reflex** into `AlbertCogitator` (octolab's
   `acl_command` + `is_owner` + `format_acl_result` are the template). It fits
   `AlbertCogitator`'s existing reflex/command path.

*Simpler alternative:* `TelegramConnector::with_acl(id, token, acl, acl_path)` in
code, skipping the manifest — but the config-driven path is the agreed convention.

## New octo API surface Albert can use

- `octo_connector_telegram::{factory, TelegramConnector::with_acl, Acl, Role}`.
- Control commands to the telegram connector:
  `octo.telegram.allow_chat` `{chat_id, role}`, `…remove_chat` `{chat_id}`,
  `…list_chats` `{}` → each replies `<kind>.result` (correlated).
- Authorization signal on inbound envelopes: `env.channel_metadata.trust`
  (gradient) + the `role` tag (`owner` / `trusted`) — gate privileged actions on
  these (the `/allow` reflex does).
- `Filter::with_min_trust(level)` for a cogitator subscription floor (optional;
  telegram already edge-drops).

## Open / decide next

- Owner `role` is seeded from `owner_chat` in the manifest; Albert may want the
  cogitator able to **promote** a chat to `owner` too (today `/allow` adds at
  `trusted`). Decide the role model when porting `/allow`.
- The LLM path ("add Vasya" in natural language) was deliberately **deferred** —
  security actions stay on the deterministic `/allow` command. Revisit only if
  wanted, and keep it owner-gated.
- Albert's scheduler is the dynamic alarms/reminders one; the Telegram authz is
  independent of it.

## Connections

- **Plan:** [[roadmap]].
- **Bus prerequisite already done:** [[octo_bus_backpressure_fix]].
- **Code:** octo at `~/code/personal/octo/` (commits `053d8c8` `29e1124` `8d7a15f`
  `4e06b10` on local `main`); Albert at `~/code/personal/albert/albert/`.
