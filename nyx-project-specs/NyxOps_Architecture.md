# NyxOps architecture and dependency strategy

| | |
| --- | --- |
| Status | Accepted |
| Version | 1.0 |
| Date | 2026-11-23 |
| Replaces | Section 4 (Technology Stack) and the queue, search, and storage choices in the Project Brief v1.0 draft (`archive/NyxOps_ProjectBrief_v1.0_Draft.pdf`) |
| Scope | Backend, frontend, data, security, deployment, and the policy for what NyxOps builds versus what it depends on |

## 1. Summary

NyxOps moves from the Python/Next.js scaffold to a Go backend shipped as a single binary, a static React frontend, and PostgreSQL as the one stateful system that handles records, jobs, search, sessions, and the audit log. A small agency runs four or five containers instead of ten.

NyxOps builds in-house the parts that make it a better records management system than commercial products: authorization, the audit log, NIBRS validation, person matching, retention and expungement, and the event log that ties them together. It depends on outside projects for anything governed by a hard security standard (cryptography, TLS, OpenID Connect sign-in) and for anything whose difficulty lies outside the RMS domain (the database engine, object storage engines, typesetting).

The design goals, in priority order when they conflict:

1. Secure enough to hold Criminal Justice Information (CJI) under the FBI CJIS Security Policy.
2. Operable by an agency with one part-time IT person.
3. Fast: sub-50 ms server time for normal record reads and writes on modest hardware.
4. Maintainable for 10 or more years by a small open source team.

## 2. Findings in the current scaffold

The repos contain little code, so the stack change costs almost nothing now. Fix these defects during the port regardless of language:

| Location | Defect | Fix |
| --- | --- | --- |
| `nyx-api/app/db.py` | `create_engine` (synchronous) receives a `postgresql+asyncpg://` URL from `.env.example`, so the first query fails | Moot after the port; one driver (`pgx`) handles everything |
| `nyx-api/app/db.py` | The docstring recommends `SET app.current_agency_id`; on a pooled connection that setting persists into the next request, which may belong to another agency | Set tenant context with `set_config('app.agency_id', $1, true)` inside the request transaction |
| `nyx-api/app/models/incident.py` | `occurred_at` and `reported_at` are `String(50)` | `timestamptz`, plus a `tstzrange` for approximate occurrence windows |
| `nyx-api/app/models/incident.py` | `report_number` is unique across all agencies while the comment says per agency | Composite unique on `(agency_id, report_number)` |
| `nyx-api/requirements.txt` | No version pins; `python-jose` and `passlib` are unmaintained; two Postgres drivers | Moot after the port |
| `nyx-worker/README.md` | Describes an OR-Tools routing solver called `nyx-solver`, unrelated to the RMS | Remove the repo; workers live in the main binary |
| `nyx-infra/docker-compose.yml` | `minio/minio:latest` is unpinned; MinIO removed console features from its community edition and stopped publishing community builds in 2025 | Replace with the in-house storage layer described in section 7.6 |
| Repo layout | Six repos mean schema, API client, and deployment changes can't land in one commit | One monorepo (section 9) |

## 3. Language: Go

Go and Rust both meet the speed goal. An RMS request spends most of its time waiting on PostgreSQL, so query design and indexing decide latency far more than the choice between two compiled languages.

Go wins on the other three goals:

- The Go standard library covers the HTTP server, TLS, cryptography, JSON, structured logging, and testing. A comparable Rust service pulls in a few hundred crates, and every one of them is supply-chain risk for software holding CJI.
- Go 1.24 and later include a native FIPS 140-3 cryptographic module, turned on at build time with `GOFIPS140`. CJIS requires FIPS-validated cryptography. Rust reaches FIPS through `aws-lc-rs`, which adds a C and CMake toolchain to every build.
- Go's compatibility promise means code written now still builds a decade from now, and new contributors from agency IT shops can read it within days.

Rust stays available for isolated tools where it already leads, such as Typst for PDF generation (section 6).

## 4. Dependency policy

Every proposed dependency goes through these rules in order. The first rule that applies decides.

1. Core RMS domain logic (authorization, audit, NIBRS, person records, retention): build it. No library models law enforcement records well, and these features are the reason NyxOps exists.
2. Security-critical work governed by a hard standard (cryptographic primitives, TLS, OIDC provider, password and MFA storage): adopt a mature implementation. Writing these in-house adds risk and no value.
3. Small, fully specified work (a migration runner, Prometheus text output, S3 request signing): build it. A few hundred owned lines beat an unowned dependency tree.
4. Build-time only tools (code generators, bundlers, linters): adopt freely, pin versions, and review upgrades. They never execute in production.
5. Single-vendor projects with a history of license changes (MinIO, Redis, Zitadel): use them only behind a NyxOps interface so an agency can swap them.

Any runtime dependency, whether adopted or replacing a built component, needs a short decision record in `docs/decisions/` stating which rule applied, the license, the transitive dependency count, and the exit plan.

## 5. Build and adopt decisions

| Component | Decision | Implementation | Reason |
| --- | --- | --- | --- |
| Database engine | Adopt | PostgreSQL 18 | Foundation for everything else; Postgres 18 adds native `uuidv7()` |
| Database driver | Adopt | `pgx` v5 | Wire protocol, pooling, and binary encoding aren't worth redoing |
| Query layer | Adopt (build time) | `sqlc` | Generates plain Go from SQL; no runtime component |
| Migrations | Build | Embedded SQL files, advisory lock, version table | Roughly 200 lines; RLS policies and triggers stay in reviewable SQL |
| HTTP and API | Build | `net/http` routing, spec-first OpenAPI file, `oapi-codegen` (build time) | No web framework at runtime |
| Job execution | Adopt behind interface | River | Retry, stuck-job rescue, and leader election are subtle; River uses only Postgres and `pgx` (D-009) |
| Event log and outbox | Build | Section 7.1 | Backbone for audit, search indexing, NIBRS, notifications, and sync |
| Identity provider | Adopt, swappable | Any OIDC provider; Zitadel bundled for agencies without one (D-007) | Sign-in, MFA, passkeys, and federation are high-risk to write |
| Token verification | Adopt | `go-oidc`, `go-jose` | JWT verification has known traps such as algorithm confusion and key rotation |
| Browser sessions | Build | Backend-for-frontend; sessions in Postgres; HttpOnly, SameSite=Strict cookies | Tokens never reach browser JavaScript |
| Authorization | Build | Section 7.2 | Core domain; off-the-shelf engines can't compile policy to SQL |
| Audit log | Build | Section 7.3, Merkle math from `golang.org/x/mod/sumdb/tlog` | Core domain; the `tlog` package is small and maintained by the Go team |
| Cryptographic primitives | Adopt | Go standard library in FIPS mode | Never build |
| Envelope encryption and key handling | Build | AES-GCM and HKDF from the standard library; pluggable key-encryption-key source | Required for crypto-shredding (section 7.4) |
| Object storage engine | Adopt, optional | Any S3-compatible service; Garage as the self-hosted default | Distributed storage is out of scope |
| Evidence and file store | Build | Section 7.6, with local-disk and minimal S3 backends | Removes the AWS SDK and the MinIO dependency |
| Search | Build | PostgreSQL full-text search, `pg_trgm`, `fuzzystrmatch` | Avoids a second copy of CJI in a separate index |
| Person matching | Build | Section 7.5 | Core domain |
| NIBRS validation and export | Build | Section 7.7 | Core domain |
| Rule expressions | Adopt | `cel-go` | Safe, non-Turing-complete, specified; inventing an expression language is a known trap |
| Forms | Build | Generated from the rule definitions (section 7.7) | Replaces `react-hook-form` and `zod` |
| Offline drafts | Build | Section 7.8 | Scoped to draft reports, not general sync |
| PDF output | Adopt | Typst CLI in a sandboxed worker process | Typesetting is a deep problem outside the RMS domain |
| TLS termination | Build | `golang.org/x/crypto/acme/autocert` or agency-supplied certificates | Removes the reverse proxy container |
| Metrics and logs | Build | `/metrics` in Prometheus text format; `log/slog` | The OpenTelemetry Go SDK adds a large tree; tracing ships behind a build tag |
| Backup and restore | Build | `nyx backup` wrapping `pg_basebackup` and the file store | Encrypted, verified backups with scheduled restore tests |
| Frontend runtime | Adopt | React, TanStack Router, TanStack Query, Radix primitives | Kept to a short, pinned list |
| Frontend build | Adopt (build time) | Vite, TypeScript, Tailwind CSS | Not shipped to production |
| UI components | Own | shadcn/ui copied into the repo | Code lives in the repo and changes through review |

### Runtime dependency inventory

The production Go binary carries six direct dependencies:

| Module | License | Purpose |
| --- | --- | --- |
| `github.com/jackc/pgx/v5` | MIT | PostgreSQL driver and pool |
| `github.com/riverqueue/river` | MPL-2.0 | Job execution |
| `github.com/coreos/go-oidc/v3` | Apache-2.0 | OIDC client and token verification |
| `github.com/google/cel-go` | Apache-2.0 | Policy and validation expressions |
| `golang.org/x/crypto` | BSD-3-Clause | ACME certificates |
| `golang.org/x/mod` | BSD-3-Clause | Transparency log math and signed notes |

Zitadel (AGPL-3.0), Garage (AGPL-3.0), Typst (Apache-2.0), and PostgreSQL (PostgreSQL License) run as separate processes and NyxOps doesn't link them, so they don't constrain the NyxOps license choice. Section 13 covers the license decision.

## 6. Target architecture

```mermaid
flowchart LR
    browser["Browser (React SPA, IndexedDB drafts)"]
    idp["OIDC provider (agency IdP or bundled Zitadel)"]
    subgraph nyx["nyx binary"]
        http["HTTP + TLS + static UI"]
        authz["Authorization"]
        domain["Domain modules (incidents, persons, evidence, warrants)"]
        rules["Rule engine (CEL)"]
        events["Event log"]
        workers["Workers (River)"]
        audit["Audit log (Merkle)"]
        crypto["Envelope encryption"]
    end
    pg[("PostgreSQL 18")]
    files[("File store: local disk or S3")]
    typst["Typst (sandboxed)"]

    browser -- "HTTPS, session cookie" --> http
    http -- "OIDC code flow" --> idp
    http --> authz --> domain
    domain --> rules
    domain -- "same transaction" --> events
    domain --> crypto --> files
    events --> workers
    events --> audit
    workers --> typst
    domain --> pg
    events --> pg
    audit --> pg
```

The `nyx` binary has subcommands `serve`, `worker`, `migrate`, `backup`, `restore`, and `verify-audit`. Small deployments run `serve` with workers in-process; larger ones run separate `worker` replicas.

## 7. Core subsystems

### 7.1 Event log ties every downstream feature to one transactional stream

Every domain change writes an event row in the same transaction as the data change. Consumers (audit log, search indexer, NIBRS aggregator, notifications, offline sync, future inter-agency sharing) read the stream; none of them reach into domain tables.

- Schema: `events(id uuid, agency_id, aggregate_type, aggregate_id, event_type, payload jsonb, actor_id, xact_id xid8, recorded_at)`.
- Ordering: a `bigserial` cursor isn't safe because a transaction that commits late can insert a lower sequence number after a consumer has moved past it. Consumers read up to the watermark from `pg_snapshot_xmin(pg_current_snapshot())` and order by `xact_id`, so no committed event gets skipped.
- Payloads carry identifiers and changed field names; sensitive field values stay in the encrypted domain tables so crypto-shredding (section 7.4) reaches them.

### 7.2 One policy definition drives both API checks and PostgreSQL RLS

Access rules are written once and enforced twice: by the API before it runs a query, and by PostgreSQL Row-Level Security on every row the query touches. A bug in either layer alone doesn't expose records.

Rules cover:

- Agency tenancy and multi-agency sharing agreements.
- Role and assignment: case team members, supervisors, records staff.
- Sensitivity labels: sealed, juvenile, confidential informant, CJI versus non-CJI.
- Reason for access: sensitive lookups require a stated purpose, which goes into the audit log; break-glass access notifies a supervisor.

Policies use a restricted CEL subset over subject attributes (roles, assignments, agency) and record attributes (agency, labels, case team). A compiler translates each policy into a SQL predicate for `CREATE POLICY` statements in a generated migration. The application role isn't a superuser, every tenant table uses `FORCE ROW LEVEL SECURITY`, and the request transaction sets context with `set_config(..., true)`.

Differential tests generate random subjects and records and assert the API decision equals the database decision. Cedar and OPA can't emit SQL, which is why NyxOps builds this compiler.

### 7.3 The audit log is tamper-evident and verifiable by outside parties

Every record access and change appends an entry to a Merkle tree log using the Certificate Transparency construction (RFC 6962), with tree math from `golang.org/x/mod/sumdb/tlog`.

- Leaves hold a hash of the audit event; the event itself stays in Postgres.
- At a fixed interval, NyxOps signs a checkpoint (tree size and root hash) with an Ed25519 key in the signed-note format.
- Agencies can publish checkpoints to an oversight body, a court, or a public endpoint.
- Each agency chooses its own witnesses (for example a state auditor, a court clerk, or a neighboring agency) that cosign checkpoints with their own Ed25519 keys. NyxOps doesn't run or require a shared witness network; `nyx verify-audit` checks cosignatures against the witness keys the agency configures (D-012).
- `nyx verify-audit` proves any entry is included and any later checkpoint extends an earlier one. A database administrator who edits or deletes an entry breaks the proof.

Checkpoints reveal only a size and a hash, so publishing them exposes no CJI. The database also revokes `UPDATE` and `DELETE` on audit tables from the application role, and `pgaudit` records direct database sessions.

### 7.4 Crypto-shredding makes expungement reach backups

Each record group (a person's record set, or an incident) has its own data encryption key, wrapped by the agency's key-encryption key. Sensitive columns and attached files are encrypted with the data key.

When a court orders expungement, NyxOps destroys the data key and records the destruction in the audit log. The ciphertext in the live database, replicas, and backups becomes unreadable.

The hard part is the key table itself: if full backups contain it, old backups still hold the destroyed key. The design keeps wrapped data keys in a separate key store with its own backup stream and a fixed 30-day retention window (D-011). An expungement completes when that window passes, and the system reports the completion date to the requester.

Key-encryption-key sources sit behind an interface: an encrypted file for small agencies, TPM, cloud KMS, or HashiCorp Vault.

### 7.5 The master person index resolves duplicates with explainable matching

Duplicate person records are a chronic problem in existing RMS products. NyxOps builds probabilistic record linkage using the Fellegi-Sunter model:

- Blocking keys (phonetic surname code, birth year, identifier fragments) limit candidate pairs.
- Comparators: Double Metaphone and NYSIIS for names, Jaro-Winkler distance, date-of-birth transposition and off-by-one checks, alias graphs, and exact matches on identifiers such as driver license and state ID.
- Expectation-maximization estimates match weights per agency from that agency's data.
- NyxOps never merges automatically. Pairs above an upper threshold go to a review queue with a per-field explanation of the score.
- A link table records merges instead of destructive updates, so any merge can be reversed and audited.

### 7.6 The evidence store is content-addressed, encrypted, and backend-agnostic

- Files are stored by SHA-256 of their plaintext; the hash goes into the chain-of-custody record at intake, so any later change to the file is detectable.
- Files are encrypted with the record group's data key before they leave the process.
- Legal holds and retention schedules block deletion at the storage layer.
- Backends: local disk (default for small agencies, no extra container) and a minimal S3 client implementing SigV4 signing, multipart upload, and the GET, PUT, HEAD, and DELETE calls NyxOps needs. Garage, Ceph RGW, AWS S3, and other S3-compatible services all work.

### 7.7 NIBRS rules are data, and the same rules drive forms and validation

- The FBI NIBRS technical specification error codes become rule packs: JSON files containing CEL expressions, error codes, messages, and the fields each rule reads. A JSON Schema validates pack files at build time.
- States add their IBR variations as extra packs without changing code.
- The API validates against the active packs on every save, and the frontend shows errors inline as the officer types, instead of at the month-end submission.
- The forms engine generates field visibility, required fields, and allowed values from the same definitions, so the UI and the server can't drift apart.
- Exports produce NIBRS XML from the same data model. Offline validation reuses the rule packs compiled to WebAssembly (section 7.8).

### 7.8 Offline drafts let officers write reports without a connection

General-purpose sync engines remain a research problem; draft sync isn't, because a draft belongs to one officer until submission.

- The browser stores drafts in IndexedDB, encrypted with a non-extractable WebCrypto key tied to the session.
- Drafts use client-generated UUIDv7 identifiers.
- The server copy wins on submission and runs full validation; the event log feeds draft updates back to the officer's other devices.
- CJIS rules for mobile devices still apply: agencies must manage the devices that store drafts.

### 7.9 AI assistance (later phase) runs on agency-controlled models with officer sign-off

Candidate features: suggest a NIBRS offense code from the narrative, extract persons and vehicles for linking, and detect PII for public-records redaction.

Constraints:

- The integration layer is in-house with pluggable providers; self-hosted models are the default so CJI stays inside the agency boundary.
- Every suggestion requires an officer to accept it, and the audit log records which model produced it.
- NyxOps won't build risk scoring, predictive policing, or suspect identification features.

## 8. Security baseline

- Builds: `CGO_ENABLED=0`, `GOFIPS140` set, distroless static base image, non-root user, read-only root filesystem.
- Transport: TLS 1.2 or later everywhere, including between `nyx` and PostgreSQL.
- Sessions: server-side, idle timeout and absolute lifetime set per agency to match CJIS session-lock requirements.
- Secrets: no default passwords; `nyx init` generates every secret at install.
- CI: `govulncheck`, `golangci-lint` with `gosec`, `go test -race`, Playwright end-to-end tests, Renovate for dependency updates.
- Supply chain: SBOM from `syft` attached to every release, images signed with `cosign`, SLSA build provenance from GitHub Actions.
- Database: application role without superuser or `BYPASSRLS`, separate migration role, `pgaudit` on.

## 9. Repository layout

```
nyxops/
  cmd/nyx/              entry point and subcommands
  internal/
    api/                generated server interfaces and handlers
    authz/              policy engine and SQL compiler
    audit/              Merkle log, checkpoints, verification
    events/             event log and consumers
    crypto/             envelope encryption, key sources
    files/              evidence store and backends
    mpi/                person matching
    nibrs/              rule packs, validation, XML export
    records/            incidents, persons, arrests, evidence, warrants
  db/
    migrations/         SQL migrations, including generated RLS policies
    queries/            sqlc query files
  rules/                NIBRS and state rule packs (JSON)
  web/                  React frontend
  deploy/
    compose/            small-agency Docker Compose profile
    helm/               Kubernetes chart
  docs/
    decisions/          architecture decision records
```

## 10. Deployment profiles

The small-agency profile runs these containers:

| Container | Role | Notes |
| --- | --- | --- |
| `nyx` | API, web UI, workers, TLS | Single static binary |
| `postgres` | All application state, job queue, search, sessions, audit log | Also hosts Zitadel's database |
| `zitadel` | Identity | Omitted when the agency uses its own OIDC provider |
| `garage` | Object storage | Omitted when files use local disk or existing S3 |

Target hardware for an agency of up to roughly 100 sworn officers: 2 vCPU, 4 GB RAM, 100 GB SSD. These figures are targets; load testing in phase 1 must confirm them.

Larger and multi-agency deployments use the Helm chart with separate `serve` and `worker` replicas and a managed or replicated PostgreSQL.

## 11. Work NyxOps won't build

- An identity provider, password storage, or MFA.
- Cryptographic primitives or a TLS stack.
- An expression language or query planner.
- A general-purpose sync engine.
- A distributed storage engine.
- A typesetting engine.

## 12. Roadmap

| Phase | Deliverables | Exit criteria |
| --- | --- | --- |
| 0. Foundations | Monorepo, migration runner, HTTP layer, OIDC sign-in against any provider, sessions, CI and release signing | A signed release installs with `nyx init` and a user signs in through Zitadel and Entra ID |
| 1. Trust core | Event log, Merkle audit log, policy-to-RLS compiler, envelope encryption | Differential authorization tests pass; `nyx verify-audit` detects a manually altered entry |
| 2. First vertical slice | Incident module with narratives, attachments, and status workflow | End-to-end incident creation under RLS with audit entries; load test confirms section 10 targets |
| 3. NIBRS | Rule engine, FBI rule pack, forms engine, XML export | Exports pass FBI test-file validation |
| 4. Persons and evidence | Master person index, evidence store, chain of custody | Matching precision and recall measured on a synthetic dataset |
| 5. Retention | Crypto-shredding, retention schedules, legal holds, backup and restore | A restore drill shows an expunged record unreadable in a pre-expungement backup after the key-store retention window |
| 6. Field and AI | Offline drafts, AI assistance | Pilot with one agency |

## 13. Licensing

Every component NyxOps links into its own binary or ships in its frontend bundle uses a permissive or file-level copyleft license:

| Component | License | Effect on NyxOps |
| --- | --- | --- |
| `pgx`, React, TanStack, Radix, shadcn/ui, Vite, Tailwind CSS | MIT | None beyond keeping notices |
| `go-oidc`, `go-jose`, `cel-go`, TypeScript | Apache-2.0 | Keep notices; compatible with any permissive license |
| `golang.org/x/crypto`, `golang.org/x/mod`, Go standard library, `protobuf` and `antlr4` (via `cel-go`) | BSD-3-Clause | None beyond keeping notices |
| River | MPL-2.0 | Changes to River's own files stay MPL-2.0; the rest of NyxOps can use any license |
| Zitadel, Garage | AGPL-3.0 | Separate processes, not linked; NyxOps ships unmodified upstream images, so no obligation passes to NyxOps code |
| PostgreSQL, Typst | PostgreSQL License, Apache-2.0 | Separate processes |

No GPL or LGPL code links into NyxOps, so the project can choose the most permissive license available.

NyxOps uses Apache-2.0. MIT and 0BSD grant the same freedoms to use, modify, and resell, but Apache-2.0 adds an explicit patent grant from every contributor and a patent-retaliation clause. Government procurement and legal offices look for that patent grant, and it protects agencies from a contributor later asserting a patent against them. Contributions use the Developer Certificate of Origin (`Signed-off-by` lines) instead of a contributor license agreement.

The existing repos carry MIT license files; the monorepo starts under Apache-2.0, and the current MIT code can be relicensed because the project owns all of it.

## 14. Resolved questions

| Question | Resolution |
| --- | --- |
| Bundled identity provider | Zitadel (D-007) |
| FIPS claims | NyxOps claims FIPS 140-3 validation only once a CMVP certificate covers the Go Cryptographic Module version in the shipped Go release (D-010) |
| NyxOps license | Apache-2.0 (D-013, section 13) |
| Job queue | River, kept behind the NyxOps job interface (D-009) |
| Key-store retention window | 30 days (D-011) |
| Audit checkpoint witnesses | Agency-chosen witnesses only (D-012) |

## 15. Decision log

| ID | Decision | Status |
| --- | --- | --- |
| D-001 | Go for backend and workers | Accepted |
| D-002 | PostgreSQL 18 as the only stateful core service | Accepted |
| D-003 | Drop RabbitMQ, Redis, Celery, Meilisearch, and MinIO | Accepted |
| D-004 | REST with spec-first OpenAPI; no GraphQL in phase 1 | Accepted |
| D-005 | Static React SPA served by the Go binary; drop Next.js | Accepted |
| D-006 | Build authorization, audit, event log, NIBRS, person matching, evidence store, and retention in-house | Accepted |
| D-007 | Any OIDC provider; bundle Zitadel for agencies without one | Accepted |
| D-008 | Merge the six repositories into one monorepo | Accepted |
| D-009 | River for job execution | Accepted |
| D-010 | No FIPS validation claim without a CMVP certificate covering the shipped Go version | Accepted |
| D-011 | 30-day retention window for key-store backups | Accepted |
| D-012 | Audit checkpoints cosigned by agency-chosen witnesses only | Accepted |
| D-013 | Apache-2.0 for all NyxOps code, contributions under DCO | Accepted |
