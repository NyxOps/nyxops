<!-- SPDX-License-Identifier: Apache-2.0 -->
# D-014 Record access, storage and pilot readiness

Accepted 2026-11-24.

Search indexes, attachment storage and recovery copies can retain information after an incident key is destroyed. A signed log can prove inclusion only for accesses the application captured.

Use a restricted initial search inventory, independent attachment objects and a separately backed-up key volume with destruction records that survive restore. Capture disclosures before returning data. Bind external sign-in to the originating browser and checkpoint event consumers at individual event boundaries.

These choices limit initial search coverage, duplicate storage for identical uploads and make sensitive reads depend on audit availability. The first real-data pilot follows a restore drill and access, isolation and hold tests. Sections 7, 10 and 11 of the [architecture](../../nyx-project-specs/NyxOps_Architecture.md) define the implementation and acceptance criteria.
