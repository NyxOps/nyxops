# nyxops

NyxOps is an open source police records management system (RMS) designed to meet the FBI CJIS Security Policy. This monorepo holds the architecture specification, decision records, brand assets, and (from Phase 0 onward) all application and deployment code.

## Repository map

| Path | Contents |
| --- | --- |
| [nyx-project-specs/NyxOps_Architecture.md](nyx-project-specs/NyxOps_Architecture.md) | Source of truth for architecture, stack, security baseline, repository layout, and roadmap |
| [docs/decisions/](docs/decisions/README.md) | Architecture decision records (D-014 onward; D-001 to D-013 are inline in the spec) |
| [SECURITY.md](SECURITY.md) | Vulnerability disclosure policy and security handling rules |
| [CLAUDE.md](CLAUDE.md) | Instructions and working conventions for AI coding agents |
| `nyx-logo-primary/` | Brand logo source (`.ai`) and PNG |

## Project status

The repository currently contains the accepted architecture specification (v1.2), decision records, and brand assets. Phase 0 introduces the Go application, SQL migrations, OpenAPI specs, and React frontend using the layout in [Section 9 of the architecture spec](nyx-project-specs/NyxOps_Architecture.md#9-repository-layout).

## Architecture at a glance

- **Single binary (`nyx`):** Go 1.24+ (`CGO_ENABLED=0`, `GOFIPS140`) serving the OpenAPI REST API, embedded React SPA, and River background workers, with subcommands `init`, `serve`, `worker`, `migrate`, `backup`, `restore`, and `verify-audit`.
- **Single stateful core:** PostgreSQL 18 (`pgx` v5 + `sqlc`, UUIDv7 keys, `FORCE ROW LEVEL SECURITY`) for records, River jobs, sessions, and tamper-evident audit events.
- **Built in-house:** CEL-to-SQL authorization compiler, transactional event log, RFC 6962 Merkle audit log with agency-chosen witnesses, per-record crypto-shredding, Fellegi-Sunter master person index, encrypted evidence store, and data-driven NIBRS validation/forms.
- **Adopted behind boundaries:** PostgreSQL 18, River, `go-oidc` + `golang.org/x/oauth2` (Zitadel bundled), `cel-go`, Typst (sandboxed PDF output), and optional S3-compatible storage (Garage default).

See [NyxOps_Architecture.md](nyx-project-specs/NyxOps_Architecture.md) for the full design, dependency policy, deployment profiles, and phase exit criteria.

## Contributing

- Make each change on a focused topic branch and open a pull request against `main`.
- Sign off every commit (`git commit -s`) to certify the [Developer Certificate of Origin](https://developercertificate.org/).
- Propose architecture or runtime-dependency changes as a decision record in `docs/decisions/` and update the decision log in [NyxOps_Architecture.md](nyx-project-specs/NyxOps_Architecture.md#13-decision-log) in the same pull request.
- Report security issues privately following [SECURITY.md](SECURITY.md).

## License

Apache-2.0. See [LICENSE](LICENSE).
