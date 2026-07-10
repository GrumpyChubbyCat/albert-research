# Octo CalDAV calendar — handoff for Albert

**Status:** context / starting point
**Created:** 2026-07-10
**Part of:** [[roadmap]]
**Start here:** Albert has no calendar organ yet. octo now ships a generic CalDAV
connector plus a reusable auth crate; the next move is to **enable and configure**
a calendar (Yandex to start) so Albert can list / create / delete events.

This page hands off a batch of octo work (CalDAV + reusable http-auth) done
outside Albert. Albert **path-depends** on the sibling octo checkout
(`albert/Cargo.toml` → `../../octo/octo/...`), so all of this is **already
available to Albert** — no push needed. (The octo commits below are on octo's
`main`, now also on `origin`.)

## What landed in octo

- **`octo-http-auth`** (`3d91f40`, now at `components/http-auth/`): a small crate
  shared by connectors. `AuthConfig` is a deserializable enum
  (`auth = basic | bearer | oauth2 | none`) whose secrets are **named env vars**,
  never literals in a manifest. `HttpAuth` is the runtime wrapper: resolves a
  `Credential` and, for `oauth2`, exchanges a refresh token for short-lived access
  tokens (cached until near expiry). Reusable by CalDAV / SMTP / HTTP connectors.
- **`octo-connector-caldav`** (`c346ed9`, at `connectors/caldav/`): a generic
  CalDAV connector (RFC 4791). One crate, many calendars — a configured instance
  per account, each an **env-as-tools organ** reached by dispatching a command to
  its `id`. iCalendar bodies over WebDAV verbs; auth via `octo-http-auth`. The
  collection URL is either explicit or **discovered** from a server root (PROPFIND
  `current-user-principal` → `calendar-home-set` → pick a VEVENT calendar), so a
  manifest can be just `base_url` + login + password. It is a **factory**
  (`type = "caldav"`), so it joins the `octo.toml` config-driven assembly.
- **`components/` regroup** (`94ed695`): `octo-history` and `octo-http-auth` moved
  under `components/` (crate names unchanged; imports untouched). Cosmetic —
  mentioned only so the path `../../octo/octo/components/http-auth` isn't a
  surprise if you ever reference it directly.

**Validated live against Yandex:** collection discovery and a full
create → list → delete round-trip through a `base_url`-only connector.

## What Albert should do (the actual task)

Same shape as the Telegram authz wiring — config-driven, mirror octolab.

1. **Register the factory** in `main.rs` alongside the telegram one:
   ```rust
   builder = builder
       .register_connector_type("caldav", octo_connector_caldav::factory())
       .from_config_file(ALBERT_OCTO_TOML)?;
   ```
   (Add `octo-connector-caldav = { path = "../../octo/octo/connectors/caldav" }`
   to `albert/Cargo.toml` if not already a dep.)

2. **Add a manifest** `albert/config/connectors/calendar/calendar.toml`
   (under the `[connectors] dir` already scanned for telegram):
   ```toml
   [connector]
   id       = "calendar"
   type     = "caldav"

   base_url = "https://caldav.yandex.ru"   # discovery; no collection URL by hand
   # calendar = "My events"                # optional: narrow by display name

   auth         = "basic"
   login        = "you@yandex.ru"
   password_env = "OCTO_YANDEX_APP_PASSWORD"   # APP PASSWORD, never the account one
   ```
   Put the app password in the environment, not the file.

3. **Give the cogitator the tool.** The connector advertises a catalog
   (`ConnectorCapabilities::with_description`), so if Albert builds its rig/LLM
   tool set from registered connectors' catalogs, `calendar` shows up
   automatically. Otherwise, dispatch commands to `id = "calendar"` and await the
   correlated `<kind>.result` (same request/response pattern as everything else).

## Commands (env-as-tools)

Dispatch a command envelope to the connector's `id`; each replies with a
correlated `<kind>.result`:

- `calendar.list_events` `{ from: <RFC3339>, to: <RFC3339> }`
  → `{ events: [{ uid, title, start, end, location? }] }`
- `calendar.create_event`
  `{ title, start: <RFC3339>, end: <RFC3339>, description?, location? }`
  → `{ uid }`
- `calendar.delete_event` `{ uid }` → `{ deleted: bool }`

On failure the result payload is `{ error: "<message>" }` (the connector never
panics the bus).

## Two calendars (work + personal) — no new code

Multiple instances of one type is already supported: the factory is keyed by
`type`, identity is `id`. Two Yandex accounts = **two manifest files**, different
`id` + `login` + `password_env`:

```toml
# calendar-work.toml
[connector]
id = "calendar-work"
type = "caldav"
base_url = "https://caldav.yandex.ru"
auth = "basic"
login = "work@yandex.ru"
password_env = "OCTO_YANDEX_WORK_APP_PASSWORD"
```
```toml
# calendar-personal.toml
[connector]
id = "calendar-personal"
type = "caldav"
base_url = "https://caldav.yandex.ru"
auth = "basic"
login = "personal@yandex.ru"
password_env = "OCTO_YANDEX_PERSONAL_APP_PASSWORD"
```

The model then sees two distinct tools (`calendar-work`, `calendar-personal`).

## Provider notes

- **Yandex / Fastmail / iCloud / Nextcloud:** `auth = "basic"` with an **app
  password** (not the account password). Discovery works from the server root.
- **Google:** `auth = "oauth2"` only (raw-password / "less secure apps" was
  removed ~2022). `octo-http-auth` has the oauth2 refresh flow, but the Google
  end-to-end (consent, first refresh token) is **not yet wired** — defer unless
  needed.

## Open / decide next

- **Reminders / alarms.** CalDAV has **no native push/subscribe** — a `VALARM`
  is just data inside the VEVENT. To make Albert *notified*, the plan is: parse
  `VALARM` on read and **poll** the calendar on a cadence, feeding due reminders
  into Albert's scheduler (the dynamic alarms/reminders actor). Not built yet;
  it's the natural follow-up once list/create work end-to-end.
- **Recurrence (RRULE).** `list_events` returns raw VEVENTs; recurring-event
  expansion over a time range is not handled yet. Fine for simple one-off events;
  revisit if Albert needs recurring ones.
- **SMTP connector.** The CalDAV brief also wanted email (send via `lettre`,
  reusing `octo-http-auth`). Separate connector, not started.

## Connections

- **Plan:** [[roadmap]].
- **Sibling handoff:** [[octo_authz_handoff]] (same config-driven wiring pattern —
  `register_connector_type` + per-connector manifest under `[connectors] dir`).
- **Code:** octo at `~/code/personal/octo/octo/` (commits `3d91f40` `c346ed9`
  `94ed695` on `main`, pushed); connector at `connectors/caldav/`, auth crate at
  `components/http-auth/`; Albert at `~/code/personal/albert/albert/`.
