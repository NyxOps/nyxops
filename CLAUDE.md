# CLAUDE.md

NyxOps is an open source police records management system (RMS). It stores Criminal Justice Information (CJI) and must meet the FBI CJIS Security Policy. Treat every change as security-sensitive.

## Source of truth

- `nyx-project-specs/NyxOps_Architecture.md` defines the architecture and the accepted decision log (D-001 to D-013). Read the relevant section before designing anything, and follow it over any older document.
- `archive/` holds superseded documents. Don't build from them.
- If a task conflicts with the spec, stop and tell the user instead of working around the spec.

## Current state

- This repo (`NyxOps/nyxops`, formerly `nyx-design`) is the monorepo. The old scaffold repos (`nyx-api`, `nyx-web`, `nyx-infra`, `nyx-worker`) were deleted on 2026-11-23; don't recreate them or reintroduce FastAPI, Next.js, Celery, RabbitMQ, Redis, MinIO, Meilisearch, or Keycloak.
- Spec section 2 lists the scaffold's defects; don't repeat them in the port.
- No Go code, build commands, or tests exist yet. Don't invent commands; check the repo first.

## Target stack

- Go in one binary (`nyx`) for API, web UI, and workers; `CGO_ENABLED=0`, `GOFIPS140` on.
- PostgreSQL 18 as the only stateful core service. Use `pgx` v5 and `sqlc`; no ORM. Primary keys are UUIDv7.
- `net/http` with spec-first OpenAPI and `oapi-codegen`; no web framework, no GraphQL.
- River for jobs, `go-oidc` for sign-in (Zitadel bundled), `cel-go` for rules and policies.
- React SPA built with Vite and served by the Go binary; TanStack Router and Query, shadcn/ui, Tailwind CSS. No Next.js.
- Monorepo layout is in the spec, section 9.

## Rules

Dependencies:

- Don't add a runtime dependency (Go module or frontend package that ships to production) without a decision record in `docs/decisions/` naming the policy rule applied, the license, the transitive dependency count, and the exit plan. Ask the user before adding one.
- Never link GPL, LGPL, or AGPL code. Linked code must be permissive or MPL-2.0.
- Never implement cryptographic primitives, TLS, an identity provider, password storage, MFA, or an expression language. Use the Go standard library, `go-oidc`, and `cel-go`.
- Build in-house: authorization, audit log, event log, NIBRS validation, person matching, evidence store, retention, migrations, metrics output, S3 client.

Database and tenancy:

- The application role has no superuser or `BYPASSRLS`. Every tenant table has an `agency_id` column, RLS enabled, and `FORCE ROW LEVEL SECURITY`.
- Set request context with `set_config('app.agency_id', $1, true)` inside the transaction. Never use session-level `SET` on pooled connections.
- RLS policies come from the policy compiler (`internal/authz`). Don't hand-write a policy that diverges from the CEL definition.
- Timestamps are `timestamptz`. Approximate times use `tstzrange`.
- Uniqueness that is per agency includes `agency_id` in the constraint.
- Records are soft-deleted; expungement uses crypto-shredding (spec 7.4), not `DELETE`.

Events and audit:

- Write each domain change and its event row in the same transaction.
- Event consumers read by `xact_id` up to the `pg_snapshot_xmin` watermark, never by a serial cursor.
- Never update or delete audit rows, and never write code that needs to.
- Keep CJI values out of event payloads, logs, metrics, error messages, and test fixtures. Use synthetic data in tests.

Security claims:

- Don't claim CJIS compliance or FIPS 140-3 validation in code comments, docs, or release notes. Say "designed to meet" until certification exists.

## Working conventions

- Commits need a DCO sign-off (`git commit -s`). Don't commit or push unless the user asks.
- License headers and new files are Apache-2.0.
- An architecture change means updating the spec's decision log and adding a decision record, in the same change.
- Docs use sentence case headings and plain, active sentences.
