# Nyxite Admin — Features

Next.js + shadcn/ui operator dashboard (reuses the **web client** stack). A **separate component** — the administration surface extracted out of the server into its own repo, so operating an instance is no longer built into the backend. It is the single place an operator manages the instance, accounts, access, and audit at **enterprise scale (many users)**.

**Privacy first / full E2EE — zero-knowledge by construction, and privacy outranks features.** The dashboard **never touches encrypted content or keys**. Everything below operates on **identity, metadata, structure, access control, and policy** — never file content. It reads structure/usage/audit **by opaque ID** from the server's admin API; file/folder names are ciphertext it cannot decrypt, and there is **no break-glass** because the server holds no key. Any admin capability that would require reading content is **intentionally absent** (see *Out of scope*, below). Resource lookups follow the platform's **existence-hiding** rule — a resource you have no reach to returns **`404`, not `403`**, so an authz failure never reveals that an id exists. It is a **full Next.js (SSR) app**, but its server side only ever calls the server's `/admin/**` API; it has no blob-store or content access of its own, and is reached over **WireGuard** (admin-only).

## Roles & permissions

- **Permissions are system-defined** — one permission per feature/capability, fixed by the system (not free-form). New product features introduce their own permissions; operators cannot invent permissions the system can't enforce.
- **Roles bundle permissions.** Built-in role presets ship out of the box (e.g. owner/super-admin, security auditor (read-only), user manager, help-desk); operators may also **create custom roles in the UI** by composing them from the fixed permission set.
- **Least-privilege / delegated administration** — grant a scoped role without full admin. Enforcement is **per-permission and target-aware**: a grant may carry a **scope** (e.g. restricted to certain groups, or excluding other admins), enabling "manage non-admins only" or "admin over group X"; a null scope is instance-wide. The server evaluates permission **and** target on every call (existence-hiding applies: no reach → 404, visible-but-forbidden → 403).

## Groups

- **Create groups** and assign users to them; a group carries one or more **roles**, so members inherit the union of those roles' permissions.
- **Bidirectional assignment** — from a **group view**, add multiple users at once; from a **user view**, add one user to multiple groups.
- These are **access/permission groups** (metadata only, no keys). The richer **enterprise/family groups** — files shared within a group with per-visibility encryption and hierarchical keys — are a **separate, deferred** capability that will build on this group model (see [OPEN-DECISIONS.md](../docs/OPEN-DECISIONS.md), backlog: enterprise/family groups).

## User & account management

- List accounts (identity + usage); per-user storage usage, counts, device list, key generations
- **Per-user storage quota** — set a maximum storage size per user; the server counts ciphertext bytes and enforces the limit at upload
- **Bulk editing** — select and edit **multiple users at once** (role/group assignment, quota, status, limits) in one action
- **Block a user (download-only state).** A blocked account can **only download files to keep them locally**: the server denies all writes (create/edit/upload/share) and the **web UI is fully disabled** for them; ciphertext download via the desktop/Android/API path still works so they retain local copies. Enforced server-side as an account state, reversible.
- Administrative **device revoke** — cut off an enrolled device's enrollment/relay access (the admin still cannot read its keys)
- **Provisioning at scale** — SCIM auto-provisioning/deprovisioning from an enterprise IdP, CSV import/export, domain-verified auto-join
- **JIT / time-boxed elevated access** for administrators

## Sessions & security policy

- **Session management** — list active sessions, force-logout, revoke refresh tokens
- **Policy enforcement** — mandate MFA/passkeys, password policy, session timeout, and IP allowlist / geofencing

## Instance & structure (no content, no names)

- Instance status / health: storage, PostgreSQL, blob store, relay, and key-directory health
- Effective **non-secret** configuration view (never token-signing key, Keycloak client secret, or any key)
- Projects and files browsed **by opaque ID**: owners, sizes, content-type, version counts — **never names, never content**

## Audit & security monitoring

- Query and filter the append-only audit log (auth, device/key lifecycle, shares, admin actions, key rotations, purges — **never content**)
- View, **verify, and export signed audit bundles** — NDJSON + a manifest with a rolling-hash chain and a detached **Ed25519** signature over the chain head; the dashboard replays the chain and checks the signature to prove an export is complete and untampered
- **SIEM streaming** — forward the audit log to syslog / webhook, with configurable **retention tiers**
- **Metadata-based anomaly detection** — surface and alert on suspicious access patterns (mass-download / exfiltration, impossible-travel logins, brute-force) from metadata alone, never content. This is a differentiator: abuse detection with zero content access.

## Keys & devices (operational, public material only)

- Key-directory consistency, orphaned wrapped keys, and rotation backlog — public/opaque material only

## Configuration & maintenance

- Operator-configured audit retention
- Triggerable maintenance jobs — blob GC, key-directory consistency checks, share-expiry sweeps — all audited; **no** reindex (there is no server index) and **no** content operations
- Per-user/per-file settings (sync policy, etc.) are **client-encrypted**; the dashboard surfaces them opaque and never decrypts them

## Access & deployment

- Standalone Next.js SSR app reusing the web client's stack (Next.js + shadcn/ui) for a consistent component language
- Authenticates by **reusing the server's native admin session/token**; RBAC enforced server-side per request
- Talks only to the server's **`/admin/**` API** — no direct database, blob-store, or content access
- Ships as its **own container**, bound to the **WireGuard** interface only (admin-only); not co-hosted on the public web origin

## Out of scope (privacy over features)

Deliberately **not** in the dashboard, because they would require content access or fall outside the privacy-first remit:

- Content moderation, server-side content search, content DLP, legal-hold content production, and any **break-glass** — impossible under E2EE, by design
- Usage/business analytics, billing, and branding/white-label — dropped to keep the surface privacy-focused
- **Enterprise/family file-sharing groups** (per-visibility encryption, hierarchical keys) — deferred; tracked as a backlog item and will extend the group model here
- **Support / helpdesk tooling**, starting with a **simple bug-reporting** flow — deferred; tracked as a backlog item

## Resolved decisions

All admin-dashboard sub-decisions are ratified — canonical in [../docs/OPEN-DECISIONS.md](../docs/OPEN-DECISIONS.md) (Resolved, AD-1–AD-5):

- **AD-1** — permission enforcement: **target-aware / scoped** (per-permission guard + target `scope`)
- **AD-2** — data path: **server `/admin/**` API** (all reads and writes)
- **AD-3** — dashboard auth: **reuse the native admin token**
- **AD-4** — audit-bundle verification: **server-side** (Next.js route handler)
- **AD-5** — packaging: **own container**, WireGuard-bound

No open questions remain for this component.
