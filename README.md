# nyxops

NyxOps is an open source police records management system (RMS) for small and resource-constrained agencies worldwide, designed to meet the FBI CJIS Security Policy as its security baseline.

## Repository map

| Path | Contents |
| --- | --- |
| [nyx-project-specs/NyxOps_Architecture.md](nyx-project-specs/NyxOps_Architecture.md) | Architecture, stack, security baseline, layout, and roadmap (v1.3) |
| [docs/decisions/](docs/decisions/README.md) | Architecture decision records (D-014 onward; D-001–D-013 inline in spec) |
| [SECURITY.md](SECURITY.md) | Vulnerability disclosure policy |
| [CLAUDE.md](CLAUDE.md) | Working conventions for AI coding agents |
| `nyx-logo-primary/` | Brand logo assets |

## Status

Architecture specification accepted (v1.3); application code starts in Phase 0.

## Architecture at a glance

- **Single binary (`nyx`):** Go 1.24+ (`CGO_ENABLED=0`, `GOFIPS140`) serving the OpenAPI REST API, embedded mobile-first React SPA, and River workers.
- **Single stateful core:** PostgreSQL 18 (`pgx` v5, `sqlc`, UUIDv7, `FORCE ROW LEVEL SECURITY`).
- **Built in-house:** CEL-to-SQL authorization, transactional event log, RFC 6962 Merkle audit log with agency-chosen witnesses, per-record crypto-shredding, locale-pluggable Fellegi-Sunter person matching, encrypted evidence store, and pluggable reporting packs (UNODC ICCS, FBI NIBRS, overlays).
- **Adopted behind boundaries:** PostgreSQL 18, River, `go-oidc` + `golang.org/x/oauth2` (Zitadel bundled), `cel-go`, sandboxed Typst, and optional S3 storage (Garage default).

## Contributing

- Open pull requests from focused topic branches against `main` and sign off commits (`git commit -s`, [DCO](https://developercertificate.org/)).
- Architecture or runtime-dependency changes require an ADR in `docs/decisions/` and an update to the [decision log](nyx-project-specs/NyxOps_Architecture.md#13-decision-log).
- Report security issues privately per [SECURITY.md](SECURITY.md).

## License

Apache-2.0. See [LICENSE](LICENSE).
