# Lamantin architecture — four pillars

**Kind:** research note / architectural requirements
**Informs:** [[roadmap]] (productization / multi-user LamantinAI),
[[forkd_isolation_architecture]] (L3 = the router-for-code)
**Provenance:** distilled from the kaeru `lamantin-ai` vault export
(`sources/lamantin-ai-export.tgz` → `lamantin-architecture-four-pillars`).
**Status:** requirements captured; mostly forward-looking (multi-user product).

## The four pillars

Architectural requirements for **Lamantin** (the productized, multi-user form of
the Albert stack):

1. **SQLite per channel** — each channel gets its own SQLite store, not one shared
   DB. Natural isolation boundary + simple backup/restore unit per channel.
2. **Secure multitenancy enforced at the ROUTER** — tenant separation is not a
   per-handler concern sprinkled through the code; it is enforced at the routing
   layer, once, authoritatively.
3. **Per-user data isolation** — one user's data is unreachable from another's; a
   corollary of (1)+(2) at the data layer.
4. **Access checks performed by the router ITSELF** — the **router is the authority
   on who may reach what**. Authorization is a routing decision, not something a
   downstream organ is trusted to self-police.

## How it maps onto what Octo already has

- **Pillar 4 (router = access authority)** — *partially realized.* The Telegram
  **edge ACL** drops unlisted chats **before the bus** ([[octo_authz_handoff]]),
  `ChannelMetadata.trust` + `role` tags travel on the envelope, and
  `Filter::with_min_trust` is a subscription-level gate. That's exactly "access
  decided at the routing edge, not by the organ." The general multi-tenant router
  authority (across *all* connectors, not just Telegram) is the forward work.
- **Pillar 4 also = forkd's L3** ([[forkd_isolation_architecture]]) — per-skill
  capability grants make **forkd the router for code**: same principle, applied to
  what a script may touch instead of what a chat may reach.
- **Pillar 1 (SQLite per channel)** — points at an **`octo-history` SQLite backend**
  (today: in-memory / file). A per-channel SQLite store is the concrete backend that
  satisfies both "hot context" and this pillar.
- **Pillars 2 & 3 (multitenancy / per-user isolation)** — mostly **not built**;
  these are the productization front (turning single-user Albert into multi-user
  LamantinAI). Capture now, design when the product needs more than one owner.

## Open

- A cross-connector **router authority** abstraction (generalize the Telegram edge
  ACL into a runtime-wide access decision point) is the natural home for pillars
  2–4. Not needed for single-user Albert; needed for Lamantin.
- Decide whether "channel" (pillar 1) and "user" (pillar 3) are the same boundary
  or nested (a user with several channels).
