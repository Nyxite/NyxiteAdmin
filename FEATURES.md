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
- These are **access/permission groups** (metadata only, no keys). The richer **enterprise/family file-sharing groups** — files shared within a group with per-visibility encryption and a hierarchical group-key layer — are **resolved** (cluster **G-2–G-5**, built in **Phase 4.4**) and **build directly on this same group model**: they reuse these group/member rows as the membership primitive, adding client-side group keys and per-member grants. The admin plane still manages only **membership** (metadata); the group-key layer is client-side and the dashboard never holds a group key (see [OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md), G-2–G-5, and [features/groups.md](https://github.com/Nyxite/Nyxite/blob/main/features/groups.md)).

## User & account management

- List accounts (identity + usage); per-user storage usage, counts, device list, key generations
- **Per-user storage quota** — set a maximum storage size per user; the server counts ciphertext bytes and enforces the limit at upload
- **Per-user deletion-retention override** — override the instance-wide Trash / grace windows for a specific user (same mechanism as the quota; enterprise-gated)
- **Bulk editing** — select and edit **multiple users at once** (role/group assignment, quota, retention override, status, limits) in one action
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
- View, **verify, and export signed audit bundles** — NDJSON + a manifest with a rolling-hash chain and a detached **hybrid Ed25519 + ML-DSA-65** signature (NIST level 3; the content/operational signing boundary went hybrid-PQC at v1.0.0 per PQ-2) over the chain head; the dashboard replays the chain and verifies **both** signature halves to prove an export is complete and untampered
- **SIEM streaming** — forward the audit log to syslog / webhook, with configurable **retention tiers**
- **Metadata-based anomaly detection** — surface and alert on suspicious access patterns (mass-download / exfiltration, impossible-travel logins, brute-force) from metadata alone, never content. This is a differentiator: abuse detection with zero content access.

## Keys & devices (operational, public material only)

- Key-directory consistency, orphaned wrapped keys, and rotation backlog — public/opaque material only

## Configuration & maintenance

- Operator-configured audit retention
- **Deletion-retention windows** — set the instance-wide **Trash** (default 30 d) and **grace** (default 30 d, ~60 d to purge) window defaults, each with a per-user override; drives the staged Trash → grace → purge timeline (DL-1–DL-5). No "empty trash now" / permanent-delete-now control — every delete runs the full timeline
- **Grace-tier restore** — explicit admin action to restore an item that has left the user-visible Trash but is still within the grace window; irreversible after purge; audited
- Triggerable maintenance jobs — blob GC, key-directory consistency checks, share-expiry sweeps — all audited; **no** reindex (there is no server index) and **no** content operations
- Per-user/per-file settings (sync policy, etc.) are **client-encrypted**; the dashboard surfaces them opaque and never decrypts them

## Access & deployment

- Standalone Next.js SSR app reusing the web client's stack (Next.js + shadcn/ui) for a consistent component language
- **Shared design language ([NyxiteDesign](https://github.com/Nyxite/NyxiteDesign), Layer A only).** Consumes the same [`nyxite-tokens.json`](https://github.com/Nyxite/NyxiteDesign) single source of truth as the web client — deep-purple brand accent, Manrope (UI) + Source Serif 4 (content), light/dark themes — plus the standard components built on it (buttons, inputs, tables, badges, dialogs). It adopts **only Layer A** (tokens + standard components); it does **not** take Layer B, the app-shell/editor chrome (rail, toolbar densities, canvas), keeping its **own dense dashboard layout** so an operator always knows they're in an admin context, not the editor. Fonts are self-hosted, never a CDN. (See DS / DS-1 in [OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md).)
- Authenticates by **reusing the server's native admin session/token**; RBAC enforced server-side per request
- Talks only to the server's **`/admin/**` API** — no direct database, blob-store, or content access
- Ships as its **own container**, bound to the **WireGuard** interface only (admin-only); not co-hosted on the public web origin

## License & entitlement (read-only status)

- Surfaces the instance's **license status** decoded from the server's verified token: tier, licensed-to email, **registered/active** state, current **lease expiry**, and any **degrade / read-only lockout** warning with a days-remaining countdown. This is a **read-only** view — the dashboard neither issues nor verifies tokens (verification is the server's offline job; issuance is the vendor-side [`NyxiteLicense`](https://github.com/Nyxite/NyxiteLicense) service).
- In **community mode** the enterprise-gated capabilities appear **disabled with a "requires license" hint**: SSO / OIDC configuration, enterprise reader-groups, the **per-user quota override**, **per-user retention override**, and **per-group size override**, **scoped / custom RBAC roles**, and **signed audit-log export**. Basic admin, per-user management, group cap ≤ 16, and audit viewing stay available. (The sixth L-3 gate — running a **self-hosted `NyxiteSupport` desk** (SUP-11) — is a **deploy-side** capability, not a dashboard control, so it doesn't appear here.) (See L-3 in [OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md).)

## Support helpdesk (link-out only)

- Bug reporting and ticketing are a **separate vendor-side component** — the central, maintainer-run **[Nyxite Support](https://github.com/Nyxite/Nyxite/blob/main/features/support.md)** helpdesk (`NyxiteSupport`). The dashboard holds **no** ticket data and stores nothing; on instances where reporting is enabled (`support.enabled`; SUP-9) it surfaces an **outbound link** to that helpdesk's operator UI, and where reporting is disabled **no link is shown**.
- The support plane is the project's **one consensual, non-E2EE exception**, deliberately **separate** from the zero-knowledge admin/metadata plane this dashboard operates on — it never mixes with the structure/usage/audit surfaces here. See [OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md) (SUP-1–SUP-13).

## Out of scope (privacy over features)

Deliberately **not** in the dashboard, because they would require content access or fall outside the privacy-first remit:

- Content moderation, server-side content search, content DLP, legal-hold content production, and any **break-glass** — impossible under E2EE, by design
- Usage/business analytics, billing, and branding/white-label — dropped to keep the surface privacy-focused
- **Enterprise/family file-sharing group *keys*** (per-visibility encryption, the hierarchical group-key layer) — the feature is **resolved** (G-2–G-5, Phase 4.4) but its keys are **client-side**: the dashboard manages group **membership** only and never holds a group key. Key material is out of scope here by design, not deferred.
- **Ticket data / helpdesk management** — the dashboard stores and manages **no** tickets; bug reporting + ticketing are the separate vendor-side [Nyxite Support](https://github.com/Nyxite/Nyxite/blob/main/features/support.md) component, which the dashboard only **links out** to (see *Support helpdesk (link-out only)* above)

## Resolved decisions

All admin-dashboard sub-decisions are ratified — canonical in [../docs/OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md) (Resolved, AD-1–AD-5):

- **AD-1** — permission enforcement: **target-aware / scoped** (per-permission guard + target `scope`)
- **AD-2** — data path: **server `/admin/**` API** (all reads and writes)
- **AD-3** — dashboard auth: **reuse the native admin token**
- **AD-4** — audit-bundle verification: **server-side** (Next.js route handler)
- **AD-5** — packaging: **own container**, WireGuard-bound

No open questions remain for this component.
