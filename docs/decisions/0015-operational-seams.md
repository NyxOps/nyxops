<!-- SPDX-License-Identifier: Apache-2.0 -->
# D-015 Operational seams, key boundaries, and repository layout

Accepted 2026-11-24.

Review of the v1.1 architecture surfaced operational ambiguities across audit ingestion, background worker RLS, multi-replica key storage, linked-record expungement, encrypted search, offline validation, and repository layout.

Adopt the following resolutions:

1. **Separate `audit_events` table:** Domain mutations write to both `events` and `audit_events` in one transaction; reads, exports, and downloads write only to `audit_events` before releasing data. The Merkle worker consumes `audit_events` so domain event consumers never scan read traffic.
2. **RLS for workers and multi-agency sharing:** Tenant tables enforce `FORCE ROW LEVEL SECURITY`; background workers iterate per agency and set `app.agency_id` inside each transaction, while global system tables (`river_*`, audit checkpoints, agency registry) have no RLS. Multi-agency sharing uses explicit `record_shares`/`agency_agreements` tables compiled into read-scoped `EXISTS (...)` RLS predicates.
3. **Bootstrap and provisioning:** `nyx init` bootstraps the first agency and initial admin via OIDC issuer/subject or a single-use token; agency admins manage memberships and roles in the NyxOps UI, with optional OIDC group sync later. External IdP logins verify `amr`/`acr` MFA claims when configured.
4. **Key store and DEK boundaries:** Place wrapped keys behind a `KeyStore` interface: a dedicated local volume for single-node Docker Compose, and a separate key-store PostgreSQL instance (or external KMS/Vault plus a separate destruction-register DB) with 30-day backup retention for multi-replica Helm. Assign each top-level record (each person, each incident) its own DEK, keeping join rows PII-free so expunging one person doesn't destroy a linked incident's DEK.
5. **Search and offline validation:** Use per-agency HMAC blind indexes for exact identifier/report-number lookups and require a per-field privacy ADR before enabling plaintext PostgreSQL FTS/trigram indexes on sensitive fields. Defer offline NIBRS rule evaluation (JS CEL vs. WASM) to Phase 6; run NIBRS validation server-side on save/blur in Phase 3. Protect offline IndexedDB drafts with a per-device officer PIN/passphrase (or WebAuthn PRF) cleared on logout/shift handoff so CJIS session idle locks don't destroy unsent field drafts.
6. **Packaging and dependencies:** Ship the static Typst binary in the distroless worker image (with mounted tmpfs/`emptyDir` scratch and no network) or as an optional Helm sidecar. Record `golang.org/x/oauth2` (BSD-3-Clause, Rule 2, 0 transitive runtime deps outside stdlib, exit plan: standard OAuth2 code+PKCE HTTP client) as the seventh direct Go dependency, mark `go-jose` as transitive via `go-oidc`, and require an ADR before enabling optional OpenTelemetry tracing.
7. **Layout and docs hygiene:** Update Section 9 with `api/`, `policies/`, missing `internal/` packages, `.github/workflows/`, `SECURITY.md`, and `CONTRIBUTING.md`; remove deferred `arrests`/`warrants` from the initial diagram and layout; trim `README.md` to avoid spec drift.

Sections 3, 5, 6, 7, 8, 9, 10, 11, and 12 of [NyxOps_Architecture.md](../../nyx-project-specs/NyxOps_Architecture.md) reflect these decisions.
