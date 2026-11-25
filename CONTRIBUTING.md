<!-- SPDX-License-Identifier: Apache-2.0 -->
# Contributing to NyxOps

Thank you for helping build NyxOps. Because NyxOps is designed to meet the FBI CJIS Security Policy and handle Criminal Justice Information (CJI), treat every change as security-sensitive.

## Before you start

- Read [nyx-project-specs/NyxOps_Architecture.md](nyx-project-specs/NyxOps_Architecture.md) to understand the system boundaries, security baseline, and roadmap.
- Install the pinned local toolchain with [mise](https://mise.jdx.dev/):

  ```sh
  mise install
  ```

## Branching and commits

- Open pull requests from focused topic branches against `main`.
- Use imperative commit subjects in the form `type: Imperative summary` (for example, `docs: Add contributor guide`).
- Sign off every commit with the [Developer Certificate of Origin (DCO)](https://developercertificate.org/):

  ```sh
  git commit -s
  ```

- Add an Apache-2.0 SPDX header to every new source, configuration, and documentation file.

## Architecture and dependency changes

Any change that alters an accepted architectural boundary or adds/replaces a production runtime dependency (Go module or frontend package) must include:

1. A new ADR in [docs/decisions/](docs/decisions/README.md) (`NNNN-short-title.md`).
2. An updated entry in the [decision log](nyx-project-specs/NyxOps_Architecture.md#13-decision-log) in the same pull request.

Dependency ADRs must state the dependency policy rule applied (Section 4 of the spec), the license, the transitive dependency count, and the exit plan. Linked code must use permissive licenses (MIT, Apache-2.0, BSD-3-Clause) or MPL-2.0; never link GPL, LGPL, or AGPL code.

## Security and data hygiene

- Never commit real CJI, real agency data, or credentials. Use synthetic fixtures only.
- Never claim CJIS compliance or FIPS 140-3 validation until formal certification covers the shipped release.
- Report security vulnerabilities privately per [SECURITY.md](SECURITY.md). Do not open public issues for security bugs.
