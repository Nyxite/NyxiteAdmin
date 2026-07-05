# NyxiteAdmin — Implementation Readiness Assessment

**Question:** Can NyxiteAdmin be **fully implemented across all phases** using only (a) the
component's own repo (`NyxiteAdmin/`) and (b) the shared Nyxite info repo (`Nyxite/`)?

**Verdict: NO.**

The concept, architecture, packaging, auth model, and RBAC/zero-knowledge semantics are
specified well enough. But NyxiteAdmin is, by construction, **nothing but a Next.js renderer
and caller of the server's `/admin/**` API** (AD-2). That API's *codeable contract* — request/
response schemas, pagination, error bodies, the permission catalog, the audit-row schema, the
audit-bundle hash construction, the SIEM/policy config shapes, and the dashboard's own login
flow — is **server-owned and lives in the server repo** (`server specification/12, 03, 08, 09,
13` and the generated `/openapi/v1.json`), which is **not** one of the two allowed inputs.
Within (a)+(b) we have only an *endpoint inventory* (HTTP verb + path + a one-line prose
description) plus the RBAC/existence-hiding *semantics*. You cannot write a typed SSR client,
tables, forms, or a signed-bundle verifier against prose.

Scope note: `/home/luis/Programming/Nyxite/NyxiteServer/` is present on disk but is a *separate
component repo*, explicitly excluded by the task's input rules. Its `specification/` is exactly
where the missing contract lives — which confirms the gap is real, not an oversight of the
shared docs.

---

## Scope covered by the -ADM- steps (the full build surface)

Grepping every `implementation/phase-*.md`, only two steps are tagged `-ADM-`:

- **P4.2-ADM-1** (`phase-4.2-admin-and-audit.md`) — the entire operator dashboard: users/
  instance/structure views; RBAC (permission catalog, role presets + custom roles); groups
  (bidirectional user↔group); per-user quota; bulk multi-user edit; block/unblock; session
  management + security-policy views; audit query + **server-side signed-bundle verification**
  (AD-4); SIEM-stream config; device revoke; client-encrypted settings shown opaque; WireGuard-only.
- **P4.4-ADM-1** (`phase-4.4-groups.md`) — extend the groups view: per-group `max_members`
  override + group key/rotation-health & orphaned-grant surfacing.

Directly consumed server steps that *define the contract* the dashboard depends on:
`P4.2-SRV-1`, `P4.2-SRV-1b`, `P4.2-SRV-2`, `P4.2-SRV-3`, `P4.2-CORE-1`, `P4.2-CORE-2`,
`P4.4-SRV-1..4`; deploy `P4.2-DEP-1` / `P4.4-DEP-1`. Every one of these describes the endpoint/
fixture at the level of "compiles and routes; OpenAPI includes them" and defers the actual
schema to the server repo.

Acceptance cases that exercise the dashboard: `P4.2-TC-1,3,7,9,10,11` and `P4.4-TC-5`.

---

## HARD BLOCKERS (cannot write correct code without these)

### B1 — The `/admin/**` request/response contract / OpenAPI is absent
- **Missing:** field-level request and response schemas for every endpoint the dashboard renders:
  `GET /admin/users`, `/admin/users/{id}`, `PATCH /admin/users/{id}`, `POST /admin/users/bulk`,
  `import`/`export`, `GET /admin/instance`, `/admin/instance/config`, `/admin/permissions`,
  `/admin/roles`, `/admin/groups[/{id}/members]`, `/admin/projects`, `/admin/files`,
  `/admin/users/{id}/sessions`, `/admin/keys/health`, device revoke, `/admin/audit`,
  `/admin/audit/export`, `/admin/audit/stream`.
- **Blocks:** essentially all of **P4.2-ADM-1** and **P4.4-ADM-1** (every view, every form).
- **Looked in:** `NyxiteAdmin/specification/01-administration.md §1.6` (only verb+path+prose
  table); `Nyxite/implementation/phase-4.2` SRV steps; `Nyxite/docs/SPECIFICATION.md §11,§13`.
- **Need:** the server's generated **`/openapi/v1.json`** for the `/admin/**` surface, or the
  equivalent `server specification/12` schema tables (field names, types, nullability, enums).

### B2 — No pagination / filtering / sorting conventions
- **Missing:** how list endpoints page (cursor vs. offset, page-size limits, `next` token),
  and which filter/sort params `GET /admin/users`, `/admin/audit`, `/admin/projects`,
  `/admin/files` accept. The spec bills the surface as "enterprise scale (many users)", so a
  virtualized/paginated users table and audit query are core, not optional.
- **Blocks:** P4.2-ADM-1 users/structure/audit list views; `P4.2-TC-10`, `P4.2-TC-1`.
- **Looked in:** `§1.6`, phase-4.2 (only `/admin/audit?from=&to=&actor=&action=&target=` params
  are hinted — no paging/limits). Nothing in `Nyxite/`.
- **Need:** the pagination + query-param convention (ideally in the OpenAPI of B1).

### B3 — Permission catalog schema + scope attachment/JSON schema
- **Missing:** the shape of a permission object from `GET /admin/permissions` (key, label,
  description, category?) and, critically, **where a `scope` attaches** (per role? per
  role-permission row?) and its **full JSON schema**. Only two example fragments exist:
  `RequirePermission("users.block")` and `{"groups":[...],"excludeRoles":["admin"]}`. The
  keys beyond `groups`/`excludeRoles` are unspecified.
- **Blocks:** the RBAC role-composition UI and the delegated-scope editor in P4.2-ADM-1;
  `P4.2-TC-7` (target-aware RBAC).
- **Looked in:** `§1.2`, `SPECIFICATION §11`, `OPEN-DECISIONS AD-1`, phase-4.2 `P4.2-SRV-1b`.
- **Need:** permission-object schema + the canonical `scope` schema (server `03 §3.2a`, `12 §12.6`).

### B4 — Audit-row schema + audit-bundle hash construction (verifier is undefinable)
- **Missing:** (1) the exact **audit-row field set** and its **canonical serialization**;
  (2) the **hash algorithm** and **chaining rule** for the "rolling-hash chain"; (3) how
  `chainHead` is computed; (4) the Ed25519 signature/pubkey encoding and **how the dashboard's
  route handler obtains the audit-signing public key**. Only "NDJSON + `manifest.json`
  `{from,to,count,chainHead,alg:"ed25519",signature}`" and "Ed25519 over chainHead" are pinned.
  BLAKE3-256 is the platform content hash but is **not stated** to be the audit-chain hash.
- **Blocks:** the AD-4 **server-side verifier** in P4.2-ADM-1; `P4.2-CORE-2`; `P4.2-TC-3`.
  A verifier that "replays the chain" needs byte-exact canonicalization + the hash primitive.
- **Looked in:** `§1.7`, phase-4.2 `P4.2-CORE-2`/`P4.2-SRV-2`, `SPECIFICATION §15`.
- **Need:** the `P4.2-CORE-2` fixture artifact (row canonicalization, chain hash algo, test
  vectors) + audit-signing key provisioning.

### B5 — Dashboard's own authentication flow is unspecified
- **Decision is set** (AD-3: reuse the server's native admin session/token, WireGuard-only) but
  the **mechanics are not**: which endpoints implement login (password + required TOTP, and/or
  passkey/WebAuthn ceremony), token/refresh/session-cookie handling, and how the SSR layer
  attaches the admin token to `/admin/**` calls. These auth endpoints are server-owned
  (`server 08`) and appear **nowhere** in `§1.6` or (a)+(b).
- **Blocks:** the dashboard shell/login — a prerequisite for *every* view in P4.2-ADM-1.
- **Looked in:** `§1.11`, `SPECIFICATION §10,§13`, `OPEN-DECISIONS #9, AD-3`.
- **Need:** the native-auth API contract (login/TOTP/passkey/refresh) and the token-forwarding
  pattern for the SSR client.

### B6 — SIEM stream config + security-policy config have no defined contract (or endpoint)
- **SIEM:** `GET/PUT /admin/audit/stream` is listed, but the config body (syslog target,
  webhook URL, auth, the "retention tiers" enum/values) is unspecified.
- **Security policy:** `§1.5` and `SPECIFICATION §11` require the dashboard to *mandate MFA/
  passkeys, set password policy, session timeout, IP allowlist/geofencing* — yet **there is no
  `/admin/policy` (or equivalent) endpoint anywhere in the `§1.6` table**. P4.2-ADM-1 says
  "security-policy views" (read-only?), contradicting the "enforcement/mandate" wording in the
  feature docs.
- **Blocks:** the SIEM-config and security-policy UI in P4.2-ADM-1.
- **Looked in:** `§1.5, §1.6`, `SPECIFICATION §11`.
- **Need:** SIEM config schema + retention-tier values; a policy endpoint + schema, or an
  explicit ruling that policy is read-only/deferred for v1.

---

## SCOPE MISMATCHES / MAJOR AMBIGUITIES (feature docs richer than the build plan)

These are listed as dashboard capabilities in `FEATURES.md` / `§1.4` / `§1.8` / `SPECIFICATION
§11` but are **absent from the only two -ADM- build steps** and have **no endpoint or schema**.
Each needs a scoping ruling (v1.0.0 vs. deferred) *and* a contract before it can be built:

- **S1 — SCIM auto-provision/deprovision** (`§1.4`, `SPECIFICATION §11`). No SCIM surface in
  `§1.6`; not in P4.2-ADM-1. Contract + "is the dashboard the SCIM configurator?" unspecified.
- **S2 — JIT / time-boxed elevated access** (`§1.4`, `SPECIFICATION §11`). No mechanism, no
  schema (is it a scoped role with an expiry? an approval flow?). Not in any -ADM- step.
- **S3 — CSV import/export column schema** (`§1.4`). Endpoint listed; the CSV column set is
  undefined, so the import UI/validation cannot be built.
- **S4 — Domain-verified auto-join** (`§1.4`). No endpoint, no verification model.
- **S5 — Metadata-based anomaly detection** (`§1.8`, billed as a differentiator). No endpoint;
  unspecified whether signals are computed server-side (dashboard just renders) or in the
  dashboard, and no result schema. Not in P4.2-ADM-1.
- **S6 — Instance health & config shapes** (`GET /admin/instance`, `/instance/config`). The set
  of health probes and the "effective non-secret config" key list are undefined.
- **S7 — Structure object shape** (`GET /admin/projects`, `/admin/files`). Fields promised
  ("owner, size, content-type, version count, opaque id") but no schema/relationships/paging.
- **S8 — Group key/rotation-health & orphaned-grant shape** (P4.4-ADM-1, also `/admin/keys/
  health`). Promised but no response schema.

---

## MINOR AMBIGUITIES (workable with sensible defaults / low risk)

- **M1 — Error body envelope.** The *status semantics* are excellent (existence-hiding: no-reach
  `404`, visible-but-forbidden `403`, uniform `401`; also `409/412/429`+`Retry-After` from the
  test method). The **JSON error body shape** is not pinned. Assumable.
- **M2 — Bulk partial-failure response.** `P4.2-TC-10` says "atomically-or-reported"; the
  per-target result shape for `POST /admin/users/bulk` is not given. Designable, then reconciled.
- **M3 — Visual design / brand.** `OPEN-DECISIONS.md` "Design" backlog item states the design
  system / brand / UX conventions are **undefined**. shadcn/ui gives components; a build can
  proceed on defaults, but the intended look-and-feel is unresolved.
- **M4 — Client-encrypted settings display** (`P4.2-SRV-3`, `/me/settings`, per-file
  `metadata_enc`). "Show opaque" is clear enough; exact opaque representation is minor.
- **M5 — CORE fixture artifacts.** `P4.2-CORE-1` (settings object `settings=7`) and
  `P4.2-CORE-2` (audit bundle) are *described* but the actual pinned fixtures/vectors are not
  present as files in (a)+(b); needed for conformance but not for first-cut UI.

---

## What IS sufficient (for fairness)

- Component identity, stack (Next.js SSR + shadcn/ui), packaging (own container, WireGuard-only),
  and the "calls only `/admin/**`, no content/keys" boundary — fully specified and unambiguous.
- All five component decisions **AD-1..AD-5 are resolved**; `features/admin.md` states "no open
  questions remain for this component."
- The **RBAC conceptual model** (system permissions → roles → groups → union; per-permission,
  target-aware guards; server-authoritative, dashboard only hides/disables) and the
  **existence-hiding 404/403/401 rules** are specified precisely enough to drive UX logic.
- The **endpoint inventory** (which endpoints exist and what each view does) is a complete map —
  it just lacks the schemas hung off each endpoint.
- Block/quota/bulk/device-revoke **behaviours** are conceptually clear from `§1.4` + test cases.
- No *open decision* conceptually blocks the component; the blockers are **missing
  contract detail that lives in the server repo**, not unresolved product questions.

---

## Bottom line

NyxiteAdmin is **architecturally ready but not build-ready from (a)+(b) alone**. It is a thin
zero-knowledge client over a server API whose *contract is the entire spec of this component* —
and that contract is deliberately housed in the server repo (`server specification/12` and the
generated `/openapi/v1.json`), outside the two permitted inputs. The thin `specification/`
(one chapter) gives an accurate map but not a schema; you cannot type a request body, paginate a
table, render an audit row, verify a bundle, or log the operator in from prose alone.

**Single most unblocking artifact:** the server's **`/openapi/v1.json`** (or `server
specification/12`) covering the full `/admin/**` surface — it resolves B1, most of B2/B6/S6–S8
in one document. Then B3 (permission/scope schema), B4 (audit-chain fixture), B5 (native-auth
flow), and the S-series scoping rulings.

**Hard blockers: 6 (B1–B6). Scope-mismatch gaps needing a ruling + contract: 8 (S1–S8).
Minor ambiguities: 5 (M1–M5).**
