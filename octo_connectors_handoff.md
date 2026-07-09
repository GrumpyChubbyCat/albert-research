# Octo connectors handoff — generic CalDAV (Yandex/Google/…) + SMTP

**Status:** open (for a separate octo session)
**Created:** 2026-07-09
**Kind:** handoff / implementation-ready brief (work happens in `~/code/personal/octo/octo/`)
**For:** making Albert useful now — a work calendar and email.

Two new connectors to build in the octo repo, scheduler/telegram-shaped. Both are
**organs**: bidirectional/output `Connector`s that advertise a `description`, so
Albert reaches them through the existing env-as-tools dispatch — **zero cogitator
change** (proven by the scheduler). Albert wires them in `main.rs` like the
scheduler once they land.

## Pattern to follow (both connectors)

Copy the shape of `connectors/scheduler` + `connectors/telegram`:
- impl `Connector` (`id`, `capabilities`, `run`); `run` = `tokio::select!` over a
  control subscription (`Filter::by_kind("<ns>.*")` or `by_target(self.id)`) +
  shutdown.
- `ConnectorCapabilities::bidirectional()/output_only()` with
  `.with_accept_kinds([...])` and **`.with_description(CATALOG)`** so the connector
  shows up in the cogitator's env-as-tools catalog with its command shapes.
- Control command → apply → publish a correlated `<kind>.result` with
  `.with_correlation(cmd.id)` (so `publish_and_await_response` on Albert's side
  gets the answer). Same as the scheduler's `on_control`.
- **Secrets via env**, named in TOML/config like telegram's `token_env` (never the
  secret literal in config). Yandex needs an **app password** (not the account
  password) for both CalDAV and SMTP.

---

## 1. `connectors/caldav/` — a GENERIC CalDAV connector (Yandex, Google, …)

**Not Yandex-specific.** CalDAV is a standard (RFC 4791 / iCalendar 5545), so **one
crate serves every CalDAV provider** — Yandex, Fastmail, Nextcloud, iCloud, Google.
New crate `octo-connector-caldav` at `connectors/caldav/`. (A `connectors/yandex/`
crate can still hold Yandex-*specific* things later — Disk etc. — but the calendar
is generic CalDAV, not a Yandex thing.)

**Multi-instance = many calendars.** Follow the dyn-HTTP pattern ("one crate, many
instances"): one configured **instance per account**, each with its own `id` +
`description` (`calendar-work`, `calendar-personal`, …). Albert sees them all in the
env-as-tools catalog and dispatches to the right `target`. Ten calendars = ten
configs, zero new code. (One account may also hold several collections; MVP = the
default/primary collection per instance.)

**Protocol** (WebDAV verbs + iCalendar; the dyn HTTP connector can't do these):
- Discover the collection: `PROPFIND` (or configure the collection URL).
- List a range: `REPORT` with a `calendar-query` filtering `VEVENT` + time-range.
- Create/update: `PUT` an `.ics` (VEVENT) to `<collection>/<uid>.ics`.
- Delete: `DELETE <collection>/<uid>.ics`.

**The one real split — AUTH mode (protocol is shared, login isn't):**
- **`basic` (app password)** — Yandex, Fastmail, Nextcloud, iCloud. Trivial, do
  this first. Yandex needs an **app password**, not the account password.
- **`oauth2` (bearer + refresh)** — **Google** (`apidata.googleusercontent.com/
  caldav/v2/<calId>/events`). Same CalDAV, but needs an OAuth2 access token + refresh.
  A **follow-on** (one added auth mode in the same connector), not day-one.

Model auth as an enum on the instance config so a new provider = a preset + an auth
mode, never a new connector.

**Crates.** `reqwest` (rustls) — custom methods via
`reqwest::Method::from_bytes(b"PROPFIND"/b"REPORT")`; build/parse iCalendar with
`icalendar` and/or `ical`; `quick-xml` for the small CalDAV XML bodies.

**Command kinds (accept) + CATALOG (per instance):**
- `calendar.list_events { from: <RFC3339>, to: <RFC3339> }` → `{ events: [{ uid,
  title, start, end, location? }] }`.
- `calendar.create_event { title, start, end, description?, location? }` → `{ uid }`.
- `calendar.delete_event { uid }` → `{ deleted: bool }`.

**Config (per instance):** `id`, `caldav_base` (provider preset — e.g.
`https://caldav.yandex.ru`), `auth = basic|oauth2`, `login`, `password_env` (app
password) or the OAuth2 fields, optional `collection` (else discover). Read-only
(list) first if create/delete is fiddly — even list-only is immediately useful.

**Why it matters:** pairs with reminders ("remind me an hour before the meeting" =
Albert reads the calendar) and a morning routine (brief today's events). One
connector, every calendar the user owns.

---

## 2. `connectors/smtp/` — outbound email (generic)

New crate `octo-connector-smtp` — **generic** SMTP send (configure it for Yandex:
`smtp.yandex.ru:465` SSL, login + app password; works for any provider).

**Crate.** `lettre` (async, `tokio1-rustls-tls` feature) — the standard Rust SMTP
client. Build a `Message`, send over a `SmtpTransport`/`AsyncSmtpTransport`.

**Shape.** `output_only()` (send only). Command kind:
- `email.send { to, subject, body, cc?, bcc?, reply_to? }` → `{ ok: bool, error? }`.

**Config:** `smtp_host`, `smtp_port` (465 SSL / 587 STARTTLS), `from` (address),
`username`, `password_env`. TLS by default.

**Scope note:** inbound email (IMAP → emit `email.received` envelopes, making it
bidirectional) is a natural **later** extension; MVP is send-only.

---

## Scope boundaries (do NOT)

- No cogitator changes — these are organs; Albert dispatches to them via env-as-tools.
- SMTP MVP = send only (IMAP later). CalDAV MVP can be list-only if create/delete
  drags.
- Don't hardcode secrets; app passwords via `*_env` like telegram's `token_env`.

## After they land (Albert side, trivial)

Add each connector in `albert/src/main.rs` next to the scheduler
(`.add_connector(...)`), constructed with its config/env. Their `description` puts
them in Albert's catalog automatically — the model starts using them with no
prompt/cogitator change (maybe a one-line hint in the base instructions).

## Connections

- **Pattern source:** the scheduler connector (`octo-connector-scheduler`) and
  telegram connector (auth/env-secret shape).
- **Plan:** [[roadmap]] — Capabilities / web+organs; these are the first
  "useful-now" organs beyond chat + scheduler.
