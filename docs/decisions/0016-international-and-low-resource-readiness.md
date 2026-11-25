<!-- SPDX-License-Identifier: Apache-2.0 -->
# D-016 International and low-resource readiness

Accepted 2026-11-24.

NyxOps v1.2 tied its reporting model, person matching, and UI assumptions to US policing (FBI NIBRS, English-only phonetic encoders, patrol laptops). Resource-constrained agencies in developing nations share the same core needs—modest hardware, no license fees, tamper-evident records—under different crime standards, languages, and connectivity.

Adopt these boundaries while keeping the CJIS security baseline intact:

1. **Pluggable reporting packs (`internal/reporting/`, `packs/`):** Decouple core records from any single national standard. Ship UNODC ICCS and FBI NIBRS as base packs and load national or state overlays from a runtime directory.
2. **i18n and flexible identity fields:** Support localized UI/pack strings, Unicode, RTL layouts (SPA and Typst PDFs), single names, patronymics, multiple surnames, and unknown or approximate dates of birth.
3. **Locale-pluggable person matching:** Keep Fellegi-Sunter in `internal/mpi` behind locale-aware normalization and comparator interfaces rather than hard-coding English-only encoders.
4. **Mobile-first and intermittent connectivity:** Target low-end Android devices, resumable encrypted uploads, and Phase 6 station-to-regional sync over the event log. Keep US-specific integrations (e.g., NCIC) optional.

This aligns NyxOps with Digital Public Goods principles at modest abstraction cost in `internal/reporting` and `internal/mpi`. Reflected in [NyxOps_Architecture.md](../../nyx-project-specs/NyxOps_Architecture.md).
