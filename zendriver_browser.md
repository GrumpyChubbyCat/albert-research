# Zendriver — browser automation (Rust)

**Kind:** research note / library candidate
**Informs:** [[roadmap]] (web search + parse organ)
**Builds on:** [[forkd_isolation_architecture]]
**Provenance:** distilled from the kaeru `lamantin-ai` vault export
(`sources/lamantin-ai-export.tgz` → `zendriver-browser-automation-rust`).
**Status:** candidate; not chosen yet — weigh against the Playwright sidecar.

## What it is

**Zendriver** — a stealth website-parsing / browser-automation library in **Rust**
(also exists as a Claude plugin). Candidate for teaching Albert to browse the web,
likely wired into Octo as a **browser connector on top of forkd**
([[forkd_isolation_architecture]]).

## Why it matters for the web organ

The [[roadmap]] web-parser bullet currently commits to a **Playwright (Node)
sidecar** the connector drives (`web.fetch { render: auto|http|browser }`, agent
chooses the render path). Zendriver is the **Rust-native alternative** to that
sidecar:

- **Pro:** no Node sidecar to supervise — a Rust crate in-process (or under forkd),
  one language, one dependency tree; "stealth" (TLS/fingerprint) is its whole point,
  which is exactly the JS/captcha fast-path the roadmap wants and the reason
  **webclaw was rejected** (AGPL + paid-cloud offload).
- **Con / to verify:** driving a real browser still means spawning Chromium — so it
  belongs **under forkd** (subprocess + sandbox), same as any exec faculty; it is
  not a pure in-process tool. Maturity, headless-stealth effectiveness, and license
  need checking before committing.

## Decision posture

Keep both on the table for the web organ: **Playwright sidecar** (proven, Node) vs
**Zendriver** (Rust-native, fewer moving parts, sandbox under forkd). Pick when the
web-parser connector is actually built; the connector's `render` contract is the
same either way, so the backend swap is internal.
