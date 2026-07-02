# Nyxite Admin — Specification (v1.0.0)

This folder is the detailed, implementation-level specification for the **Nyxite Admin** dashboard — the **Next.js + shadcn/ui** operator surface that manages the instance, accounts, and audit. It is a **separate component**, extracted out of the server so administration is no longer built into the backend.

It expands the architectural planning documents in the central [`Nyxite`](https://github.com/Nyxite/Nyxite) repository (`features/admin.md`) into a concrete build specification.

## Guiding principle: zero-knowledge by construction

The dashboard **never touches encrypted content or keys.** It reads only **structure, usage, and audit by opaque ID** from the server's admin API; names are ciphertext it cannot decrypt, and there is **no break-glass** because the server holds no key. This is architectural, not a policy the dashboard enforces: no content key or plaintext ever reaches this component.

It is a **full Next.js (SSR) app**, but its server side only ever calls the server's `/admin/**` API — it has no blob-store or content access of its own — and it is reachable only over **WireGuard** (admin-only).

## Boundary with the server

The **server** (`NyxiteServer`) owns the admin **data + enforcement plane**: the `/admin/**` API, audit-log storage/append + signed-export generation, `admin`-role auth, device-revoke enforcement, and the operational jobs. See [server `specification/12`](https://github.com/Nyxite/server). This dashboard is the **operator-facing surface** over that plane. Where the two disagree on the API contract, the server spec wins.

## Source of truth

The central `Nyxite` repo is authoritative. This spec links to `docs/OPEN-DECISIONS.md` rather than restating decisions.

## Proposal convention

- **[P]** — *Proposed.* A concrete decision filled in by this spec, subject to confirmation; not yet ratified in the master docs.
- **[AD-n]** — references an admin-dashboard decision (AD-1…AD-5) in `docs/OPEN-DECISIONS.md`.

## Documents

| # | Document | Covers |
|---|----------|--------|
| 01 | [administration.md](01-administration.md) | Operator dashboard: instance/user/structure views, audit query + signed-export verification, device revoke, maintenance triggers, access model |

## Status

Specification for a greenfield build. No dashboard code exists yet; the `admin` repo currently holds `FEATURES.md`, `LICENSE.md`, and this `specification/` set. This document set defines what to build.

## License

PolyForm Noncommercial License 1.0.0 — see the repo `LICENSE.md`.
