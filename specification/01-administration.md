# 01 — Administration (operator dashboard)

> **Zero-knowledge, privacy over features.** The dashboard manages the instance, users, and access — showing **structure, usage, and audit** — but **cannot read content at all**. There is **no break-glass**, because the server holds no key. Everything here operates on **identity, metadata, structure, access control, and policy**; any capability that would need content is intentionally absent (§1.10). It is a **Next.js (SSR) + shadcn/ui** app that consumes the server's `/admin/**` API, reached **only over WireGuard**. Concrete APIs (server-owned) are **[P]**.

## 1.1 Admin model

- Access is governed by **system-defined permissions** (one per feature/capability), bundled into **roles**, granted via **groups**. A bootstrap super-admin exists from install; everything else is delegated (§1.2–1.3). With the enterprise IdP enabled, a Keycloak claim may map onto a role (server [08](https://github.com/Nyxite/NyxiteServer)).
- Dashboard visibility: account list, **structure by opaque ID** (names are encrypted), storage usage, version/key-generation counts, audit log.
- The dashboard **cannot** decrypt names or content. No content-read view and no break-glass exist.
- **Existence-hiding:** the server returns **`404` (not `403`) for any resource the caller has no reach to** (indistinguishable from non-existent — server [13 §13.6a](https://github.com/Nyxite/NyxiteServer)). The dashboard surfaces such responses as a plain "not found" and never implies "exists but forbidden"; `403` appears only for capability/collection denials.

## 1.2 Roles & permissions

- **Permissions are system-defined constants**, one per feature/capability, referenced by stable keys and enforced server-side. Operators cannot create permissions the system can't enforce; new product features ship their own permissions.
- **Roles** bundle permissions. Built-in presets ship out of the box (owner/super-admin, security auditor (read-only), user manager, help-desk); operators may also **create custom roles** by composing them from the fixed permission set (AD-1).
- **Least-privilege / delegated admin** — a scoped role grants only its permissions (e.g. user management without audit, or audit read-only).

**Enforcement — checks are per-permission, never per-role, and target-aware (AD-1).** Every protected capability is guarded, **once, in server code**, by the single permission key it requires (server route policies, e.g. `RequirePermission("users.block")` — server [12 §12.6](https://github.com/Nyxite/NyxiteServer)). A role, built-in or custom, is only a **data bundle** of those keys; adding a custom role adds **no new check site**. The operation→permission mapping is **code-owned** (ships with the feature); the DB stores only role→permission (with scope) and group assignments. At request time the caller's effective grants resolve `token → user → groups → roles → permissions` (union); the guard passes when the caller holds the key **and the target satisfies the grant's `scope`** — a `scope` constrains valid targets (e.g. `{"groups":[...]}` for delegated admin over specific groups, or `{"excludeRoles":["admin"]}` to protect other admins), a **null scope being instance-wide** (super-admin). On failure, existence-hiding applies: no reach → `404`, visible-but-forbidden → `403`. The **server is authoritative**; the dashboard reads the same effective set only to **hide/disable controls** (UX), and never enforces. Account-state gates (**quota**, **block**) are a **separate axis** — they act on the *subject* account at the upload/ACL path (§1.4), not on admin permissions.

## 1.3 Groups

- **Create groups**; a group carries one or more **roles**, so a member's **effective permissions = the union of its groups' roles**.
- **Bidirectional assignment** — from a group, add many users at once; from a user, add to many groups.
- These are **access/permission groups** (metadata only, **no keys**). The richer **enterprise/family file-sharing groups** (per-visibility encryption, the hierarchical group-key layer) are **resolved** (G-2–G-5, Phase 4.4) and **build on this same model** — reusing these membership rows, adding client-side group keys. The admin plane manages only **membership**; the group-key layer is client-side (canonical in `docs/OPEN-DECISIONS.md`, G-2–G-5).
- **File-sharing group size cap.** A file-sharing group's member count is capped at the instance default **`group_max_members` = 16** (community); the **per-group size override** that raises it beyond 16 is an **enterprise-gated** control (G-5, L-3 — see §1.9a), enforced server-side by membership-row count ([server 12 §12.7](https://github.com/Nyxite/NyxiteServer)).

Backing tables (`roles`, `role_permissions`, `groups`, `group_roles`, `group_members`) are server-owned — see server [03](https://github.com/Nyxite/NyxiteServer).

## 1.4 User & account management

- **Per-user storage quota** — set a max storage size per user; the server counts **ciphertext bytes** and enforces the limit **at upload** (server [03](https://github.com/Nyxite/NyxiteServer), [06](https://github.com/Nyxite/NyxiteServer)).
- **Per-user deletion-retention override** — override the instance-wide **Trash** and **grace** windows for a specific user (same override mechanism as the quota; instance defaults set in §1.9). The staged Trash → grace → purge timeline (DL-1–DL-5) is enforced server-side over opaque blobs; the override is an **enterprise-gated** control (§1.9a).
- **Bulk editing** — select multiple users and apply role/group assignment, quota, retention override, status, or limit changes in one action.
- **Block a user (download-only state).** `status = blocked` denies all writes (create/edit/upload/share) and **fully disables the web UI** for that account; **ciphertext download** via the desktop/Android/API path still works so the user keeps local copies. Enforced server-side as an account state (server [09](https://github.com/Nyxite/NyxiteServer)); reversible.
- Administrative **device revoke** — cut off an enrolled device (the admin still cannot read its keys).
- **Provisioning at scale** — SCIM auto-provision/deprovision from an enterprise IdP, CSV import/export, domain-verified auto-join.
- **JIT / time-boxed elevated access** for administrators.

## 1.5 Sessions & security policy

- **Session management** — list active sessions, force-logout, revoke refresh tokens.
- **Policy enforcement** — mandate MFA/passkeys, password policy, session timeout, IP allowlist / geofencing.

## 1.6 Server admin API consumed (`/api/v1/admin/**`) **[P]**

These endpoints are **server-owned** (see server [12](https://github.com/Nyxite/NyxiteServer)); the dashboard's Next.js server side calls them and renders the results.

### Users & instance
| Method | Path | Dashboard view/action |
|--------|------|---------|
| `GET` | `/admin/users` | Users list (identity + usage + status + quota) |
| `GET` | `/admin/users/{id}` | Usage, counts, devices, key generations, group/role membership |
| `PATCH` | `/admin/users/{id}` | Role/group, **quota**, limits, **status** (block/unblock) |
| `POST` | `/admin/users/bulk` | **Bulk edit** — apply changes to many users at once |
| `POST` | `/admin/users/import` / `GET …/export` | CSV import/export; SCIM handles IdP-driven lifecycle |
| `GET` | `/admin/instance` | Status: storage, DB, blob store, relay, key-directory health |
| `GET` | `/admin/instance/config` | Effective **non-secret** configuration |

### Roles, permissions & groups
| Method | Path | Dashboard view/action |
|--------|------|---------|
| `GET` | `/admin/permissions` | System-defined permission catalog (read-only) |
| `GET/POST/PATCH/DELETE` | `/admin/roles[/{id}]` | Role presets + custom roles (compose from permissions) |
| `GET/POST/PATCH/DELETE` | `/admin/groups[/{id}]` | Create/edit groups; assign roles |
| `POST/DELETE` | `/admin/groups/{id}/members` | Add/remove members (bulk) |

### Structure, sessions, keys/devices
| Method | Path | Dashboard view/action |
|--------|------|---------|
| `GET` | `/admin/projects`, `/admin/files` | Structure by **opaque ID** — no names, no content |
| `GET` | `/admin/users/{id}/sessions`; `DELETE …/{sid}` | List / revoke active sessions (force-logout) |
| `GET` | `/admin/keys/health` | Key-directory consistency, orphaned wrapped keys, rotation backlog |
| `POST` | `/admin/users/{id}/devices/{deviceId}/revoke` | Revoke a device |

### Audit
| Method | Path | Dashboard action |
|--------|------|---------|
| `GET` | `/admin/audit?from=&to=&actor=&action=&target=` | Query/filter the audit log |
| `GET` | `/admin/audit/export` | Fetch a signed audit bundle to view and verify **[P]** |
| `GET/PUT` | `/admin/audit/stream` | Configure **SIEM streaming** (syslog/webhook) + retention |

> There are **no** `/admin/breakglass`, content-read, or content-search endpoints — the capability cannot exist under E2EE.

## 1.7 Audit view & signed-bundle verification

The dashboard queries the append-only `audit_log` (server-owned) and lets the operator **filter, view, export, and stream** it. Covered actions: **auth events, device/key lifecycle, shares created/revoked, key rotations, admin actions (role/group/quota/status changes, blocks), and purges** — **never content or keys**.

**Signed audit bundle [P]:** `NDJSON` of audit rows + a `manifest.json` `{ from, to, count, chainHead, alg:"ed25519+ml-dsa-65", signature }`. The rows form a **rolling hash chain**, and the manifest carries a **detached hybrid Ed25519 + ML-DSA-65 signature** (NIST level 3) over the final `chainHead` — audit signing is a content/operational boundary and ships hybrid-PQC at v1.0.0 (PQ-2/PQ-3; the `alg` tag is the agility anchor per PQ-4). The dashboard's verifier **replays the chain and checks both signature halves** (a break in either alone is non-fatal, PQ-1); a clean bundle verifies, a tampered row fails. **Verification runs server-side, in the dashboard's Next.js route handler (AD-4)** — the verifier and the server's audit-signing public key stay off the client.

## 1.8 Security monitoring

- **Metadata-based anomaly detection** — surface and alert on suspicious access patterns (mass-download/exfiltration, impossible-travel logins, brute-force) computed from metadata alone. No content is read; the signals are access counts, timings, IPs, and session data already in the audit log.

## 1.9 Configuration & maintenance

- Effective **non-secret** config only — never the token-signing key, the enterprise Keycloak client secret, or any keys (there is no server content KEK).
- Per-user/per-file settings (sync policy, etc.) are **client-encrypted**; surfaced opaque, never decrypted.
- **Deletion-retention windows** — set the instance-wide **Trash** window default (default **30 days**) and **grace** window default (default **30 days**, ~60 days to irreversible purge), each with a per-user override (§1.4). Windows drive the server's staged deletion timeline (DL-1–DL-5, server [03 §3.3](https://github.com/Nyxite/NyxiteServer), [14 §14.3](https://github.com/Nyxite/NyxiteServer)). There is **no** "empty trash now" / permanent-delete-now control — every delete runs the full timeline (DL-5).
- **Grace-tier restore** — an explicit admin action to restore an item that has left the user-visible Trash but is still inside the grace window (DL-4); after purge it is irreversible. Audited server-side.
- Triggerable maintenance: GC/purge, key-directory consistency checks, share-expiry sweeps — audited server-side. **No** reindex (no server index) and **no** content operations.

## 1.9a License & entitlement (read-only status)

- Renders the instance's **license status** from the server's admin API (server [16 §16.10](https://github.com/Nyxite/NyxiteServer)): tier, licensed-to email, **registered/active** state, current **lease expiry**, and any **degrade / read-only lockout** warning with a days-remaining countdown. **Read-only** — the dashboard neither issues nor verifies tokens (verification is the server's offline job; issuance is the separate vendor-side [`NyxiteLicense`](https://github.com/Nyxite/NyxiteLicense) service).
- In **community mode** the enterprise-gated controls render **disabled with a "requires license" hint** (L-3): SSO/OIDC config, enterprise reader-groups, **per-user quota override** and **per-group size override**, **scoped/custom RBAC**, and **signed audit-log export**. Basic admin, per-user management, group cap ≤ 16, and audit viewing stay available. The gate is a **UX affordance only** — the server enforces entitlement (§16), the dashboard never does.

## 1.9b Support helpdesk (link-out only)

- The in-app bug-reporting helpdesk runs on the **central vendor-side `NyxiteSupport`** service (OPEN-DECISIONS **SUP-1..SUP-13**), which has **its own operator UI**. This dashboard only **links out** to that UI — it holds **no ticket data and stores nothing** about tickets. An enterprise operator who opts into their **own** desk (SUP-10–SUP-13) links out to their local desk instead.
- The link is shown **only where the server advertises `support.enabled`** (server [04 §4.9](https://github.com/Nyxite/NyxiteServer), [14 §14.9](https://github.com/Nyxite/NyxiteServer)); with the 2026-07-12 SUP-9 correction, reporting is available on **all** instances (maintainer's + third-party) unless an operator switches the surface off.
- The support plane is a **consensual, non-E2EE** plane, **separate from the zero-knowledge admin plane** this dashboard operates on; no ticket content ever crosses into the admin API or this UI.

## 1.10 Out of scope (privacy over features)

Deliberately absent, because they would require content access or fall outside the privacy-first remit: content moderation / server-side content search / content DLP / legal-hold content production / **break-glass** (impossible under E2EE); usage/business analytics, billing, branding/white-label (dropped to stay privacy-focused); **enterprise/family file-sharing group *keys*** — the feature is **resolved** (G-2–G-5, Phase 4.4) but its **group-key layer is client-side**, so key material is out of scope here by design (the dashboard manages only membership); not deferred. The **support/helpdesk (in-app bug reporting)** is **not** managed here: it lives on the vendor-side `NyxiteSupport` service and this dashboard only **links out** to its operator UI (§1.9b, SUP-1..SUP-13) — it stores no ticket data.

## 1.11 Access & hosting

- Standalone Next.js SSR app reusing the web client's stack (Next.js + shadcn/ui).
- **Shared design language (DS, Layer A only).** The UI is built from the shared [`NyxiteDesign`](https://github.com/Nyxite/NyxiteDesign) tokens (`nyxite-tokens.json` → generated CSS custom properties + a Tailwind theme, DS-3) that the web client also consumes — same deep-purple brand accent, Manrope (UI) + Source Serif 4 (content) type, and light/dark themes. The dashboard adopts **Layer A only** (tokens + standard shadcn/ui components: buttons, inputs, tables, badges, dialogs); it does **not** take Layer B, the app-shell/editor chrome (DS-1), keeping its own **dense admin layout** so the operator context stays visually distinct from the editor. Fonts are **self-hosted**, never a CDN. See `docs/OPEN-DECISIONS.md` (DS).
- **Authenticates by reusing the server's native admin session/token (AD-3)**; RBAC is enforced server-side per request.
- Its server side talks solely to the server's **`/admin/**` API (AD-2)** — no direct database, blob-store, or content access.
- Ships as its **own container (AD-5)**, bound to the **WireGuard** interface only (admin-only); not co-hosted on the public web origin.
