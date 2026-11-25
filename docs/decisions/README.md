<!-- SPDX-License-Identifier: Apache-2.0 -->
# Architecture decision records

Decisions **D-001 through D-013** were accepted as part of the initial architecture specification and are recorded inline in [NyxOps_Architecture.md](../../nyx-project-specs/NyxOps_Architecture.md#13-decision-log).

Standalone decision records in this directory start at **D-014**.

## When to add a decision record

Add a numbered file (`NNNN-short-title.md`) and update the decision log in `nyx-project-specs/NyxOps_Architecture.md` in the same pull request whenever you:

- Change an accepted architectural boundary or security invariant.
- Add or replace a production runtime dependency (Go module or frontend package). Dependency records must name:
  1. The dependency policy rule applied (Section 4 of the spec).
  2. The license.
  3. The transitive dependency count.
  4. The exit plan.

## Index

| ID | Title | Status |
| --- | --- | --- |
| D-014 | [Record access, storage and pilot readiness](0014-record-lifecycle.md) | Accepted |
| D-015 | [Operational seams, key boundaries, and repository layout](0015-operational-seams.md) | Accepted |
| D-016 | [International and low-resource readiness](0016-international-and-low-resource-readiness.md) | Accepted |
