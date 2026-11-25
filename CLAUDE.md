# CLAUDE.md

NyxOps is an open source police RMS for small and resource-constrained agencies worldwide, designed to meet the FBI CJIS Security Policy as its security baseline. Treat every change as security-sensitive.

## Source of truth

- [nyx-project-specs/NyxOps_Architecture.md](nyx-project-specs/NyxOps_Architecture.md) is the source of truth (v1.3, D-001 to D-016). Stop and ask the user if a task conflicts with it.
- No Go code, build commands, or tests exist yet. Check the repo before running commands.

## Rules

Dependencies:

- Do not add a production runtime dependency (Go or frontend) without user approval and an ADR in `docs/decisions/` (policy rule, license, transitive count, exit plan).
- Never link GPL, LGPL, or AGPL code. Linked code must be permissive or MPL-2.0.
- Never implement crypto primitives, TLS, an IdP, password/MFA storage, or an expression language.

Domain and localization:

- Keep core schemas country-agnostic: never hard-code US-only crime codes, address/phone formats, or English-only name assumptions. Support Unicode, RTL, single names, patronymics, and approximate DOBs.

Database and tenancy:

- Non-superuser app role without `BYPASSRLS`; every tenant table has `agency_id`, RLS enabled, and `FORCE ROW LEVEL SECURITY`.
- Set tenant context with `set_config('app.agency_id', $1, true)` inside the transaction; never session-level `SET`.
- Generate RLS policies from `internal/authz`; never hand-write diverging policies.
- Use UUIDv7 PKs, `timestamptz`, `tstzrange` for approximate times, and agency-scoped unique constraints.
- Soft-delete records; expungement uses crypto-shredding (spec 7.4), never `DELETE`.

Events and audit:

- Write domain mutations, `events`, and `audit_events` in the same transaction.
- Commit an `audit_events` row before returning any sensitive read, search, export, or download; never put reads in `events`.
- Consume `events` by `xact_id` up to `pg_snapshot_xmin`, never a serial cursor.
- Never update or delete audit rows. Keep CJI out of event payloads, logs, metrics, errors, and test fixtures (use synthetic data).

Security claims:

- Never claim CJIS compliance or FIPS 140-3 validation; say "designed to meet" until certified.

Formal verification:

- Model-check high-harm state machines and concurrency protocols with `/statechart` before implementing them, then bind each check to the code with `/create-formal-verification-skill` (keep in sync with `/maintain-formal-verification-skill`):
  - Phase 0: OIDC sign-in, session revocation, and `nyx init` bootstrap.
  - Phase 1: `xact_id` / `pg_snapshot_xmin` event consumer, audit-before-disclosure + Merkle checkpointing, and RLS/share/connection-pool isolation.
  - Phase 4: Resumable encrypted evidence uploads and chain-of-custody finalization.
  - Phase 5: Crypto-shredding, legal holds, destruction register, and restore reconciliation.
  - Phase 6: Offline draft key wipe on shift handoff and station-to-regional sync.

## Working conventions

- Commit and push only when asked. Use focused topic branches, PRs against `main`, `type: Imperative summary` subjects, and `git commit -s` (DCO).
- New files use Apache-2.0 headers.
- Architecture changes must add an ADR in `docs/decisions/` and update the spec's decision log in the same change.
- Docs use sentence case headings and plain, active sentences.
