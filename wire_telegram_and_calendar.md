# Task: wire Telegram authorization + CalDAV calendar into Albert

Two independent connectors to enable and configure in Albert. Do them in either
order or in parallel. Both follow the same config-driven pattern
(register_connector_type + a per-connector manifest under [connectors] dir).

## Where things are (shared)
- Albert repo:  ~/code/personal/albert/albert/  (this is what we edit)
- octo (sibling, path-dep via albert/Cargo.toml -> ../../octo/octo/...):
    telegram connector: ~/code/personal/octo/octo/connectors/telegram/
    caldav connector:   ~/code/personal/octo/octo/connectors/caldav/
    auth crate:         ~/code/personal/octo/octo/components/http-auth/
    reference wiring:   ~/code/personal/octo/octo/octolab/  (config-driven; mirror it)
  Everything below is on octo main (pushed) and available to Albert via path-dep.
- Handoffs: research/octo_authz_handoff.md, research/octo_caldav_handoff.md


================================================================================
# PART 1 — Telegram authorization (ACL)
================================================================================

## Goal
Albert's Telegram channel is currently allow-all (TelegramConnector::new = no
auth) — a hole for an always-on agent. Wire in the per-chat allow-list so
unauthorized chats are dropped BEFORE the bus, and add an owner-only /allow
reflex to manage the list at runtime.

## What octo gives us
- Telegram edge ACL + factory `type = "telegram"`: a message from an unlisted
  chat is dropped before the bus (untrusted never reaches cognition — the
  prompt-injection boundary). Listed chats get ChannelMetadata.trust + a `role`
  tag (owner/trusted) stamped on the envelope.
- Runtime-mutable ACL (manageable actor): control commands
  octo.telegram.{allow_chat{chat_id,role}, remove_chat{chat_id}, list_chats{}},
  persisted atomically to a JSON file.
- octolab's owner-only /allow /deny /allowed reflex: DETERMINISTIC command (no
  LLM in a security action), gated on the incoming message's role == owner.
- Filter::with_min_trust(level): optional subscription trust floor (defense-in-
  depth; the connector already edge-drops). ConnectorContext is now Clone.
- API: octo_connector_telegram::{factory, TelegramConnector::with_acl, Acl, Role}.

## Steps
1. Cargo: ensure albert/Cargo.toml has the telegram connector as a path dep.
2. main.rs: when a token is present, replace
       add_connector(TelegramConnector::new("telegram", token))
   with the config-driven path:
       builder = builder
           .register_connector_type("telegram", octo_connector_telegram::factory())
           .from_config_file(ALBERT_OCTO_TOML)?;   // map ConfigError -> albert::Error
   Console fallback stays as-is.
3. Manifest: albert/config/connectors/telegram/telegram.toml
       [connector]
       id         = "telegram"
       type       = "telegram"
       token_env  = "OCTO_TELEGRAM_TOKEN"
       acl_path   = "telegram_acl.json"
       owner_chat = <YOUR real chat id>   # seeds you as owner on first run
   Token stays a secret in the env; ACL is the JSON file named here.
4. Port the owner-gated reflex into AlbertCogitator: octolab's
   acl_command + is_owner + format_acl_result are the template (/allow <id>,
   /deny <id>, /allowed). Fits Albert's existing reflex/command path.

## Authorization signal (for gating privileged actions)
- env.channel_metadata.trust (gradient) + the `role` tag (owner/trusted).
- The /allow reflex checks role == owner before dispatching a control command.

## Gotchas
- owner_chat MUST be your real chat id. The ACL starts EMPTY and drops everyone,
  including you — without owner_chat you're locked out of your own bot with no
  one able to grant the first access.
- Two-layer security: edge drop (before bus) + optional Filter::with_min_trust on
  the cogitator subscription.

## Deferred (don't do unless asked)
- Natural-language "add Vasya" via the LLM: deliberately NOT done — security
  actions stay on the deterministic /allow command (a model can be talked into
  granting access; a command can't). Keep it owner-gated if ever revisited.
- Role model: today /allow adds at `trusted`; owner is seeded from owner_chat.
  Decide whether the cogitator may promote a chat to `owner`.


================================================================================
# PART 2 — CalDAV calendar
================================================================================

## Goal
Enable + configure a calendar organ so Albert can list/create/delete events.
Start with one Yandex calendar; make it show up as a tool for the cogitator.

## What octo gives us
- octo-connector-caldav: generic CalDAV (RFC 4791), factory `type = "caldav"`.
  One crate, many calendars (a configured instance per account, addressed by id).
- octo-http-auth: reusable AuthConfig (basic/bearer/oauth2/none), secrets from
  named env vars. Yandex = basic + APP PASSWORD.
- Collection URL is DISCOVERED from a server root (PROPFIND principal ->
  calendar-home-set -> pick VEVENT calendar), so config is just base_url+login+pw.
  Live-validated against Yandex (discovery + create/list/delete round-trip).

## Steps
1. Cargo: add to albert/Cargo.toml if missing:
     octo-connector-caldav = { path = "../../octo/octo/connectors/caldav" }
2. main.rs: register the factory alongside telegram, before from_config_file:
     builder = builder.register_connector_type(
         "caldav", octo_connector_caldav::factory());
   (must run through .from_config_file(ALBERT_OCTO_TOML) so manifests load)
3. Manifest: albert/config/connectors/calendar/calendar.toml
   (under the [connectors] dir already scanned for telegram):
     [connector]
     id       = "calendar"
     type     = "caldav"
     base_url = "https://caldav.yandex.ru"   # discovery; no collection URL by hand
     # calendar = "My events"                # optional: narrow by display name
     auth         = "basic"
     login        = "you@yandex.ru"
     password_env = "OCTO_YANDEX_APP_PASSWORD"   # APP password, NOT account pw
   Put the app password in the env, not the file.
4. Expose as a tool: the connector advertises a catalog via
   ConnectorCapabilities::with_description. If Albert builds its rig/LLM toolset
   from registered connectors' catalogs, `calendar` appears automatically.
   Otherwise dispatch to id="calendar" and await the correlated <kind>.result.

## Commands (env-as-tools; each replies correlated <kind>.result)
- calendar.list_events  { from:<RFC3339>, to:<RFC3339> }
    -> { events: [{ uid, title, start, end, location? }] }
- calendar.create_event { title, start:<RFC3339>, end:<RFC3339>, description?, location? }
    -> { uid }
- calendar.delete_event { uid } -> { deleted: bool }
On failure the result payload is { error: "<msg>" } (never panics the bus).

## Two accounts later (no new code)
Two manifests, different id/login/password_env (e.g. calendar-work,
calendar-personal) -> two distinct tools. Same type = "caldav".

## Gotchas
- App password, never the account password (Yandex/Fastmail/iCloud/Nextcloud).
- Google is oauth2-only and NOT wired end-to-end yet — skip for now.

## Deferred (don't do unless asked)
- Reminders: CalDAV has no push; VALARM is data. Plan = parse VALARM + poll ->
  feed due reminders into Albert's scheduler. Follow-up after CRUD works.
- RRULE recurrence expansion; SMTP connector.
