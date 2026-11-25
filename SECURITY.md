<!-- SPDX-License-Identifier: Apache-2.0 -->
# Security policy

NyxOps is designed to meet the FBI CJIS Security Policy and handle Criminal Justice Information (CJI). Treat every security issue as high priority.

## Reporting a vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

Report vulnerabilities privately using [GitHub Security Advisories](https://github.com/NyxOps/nyxops/security/advisories/new) for this repository.

Include:

- A clear description of the issue and affected component or file paths.
- Steps to reproduce using **synthetic data only** (never include real CJI, real agency data, or credentials).
- Impact assessment and any suggested mitigation.

## Security claims

NyxOps is designed to meet CJIS Security Policy requirements and uses Go's native FIPS 140-3 cryptographic module (`GOFIPS140`). NyxOps does not claim CJIS compliance or FIPS 140-3 validation until formal certification covers the shipped release.
