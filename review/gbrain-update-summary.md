# gbrain update summary — Peat current verified state

Updated 2026-06-18. All writes to the `default` source. No page deleted. Pages cross-linked with
`[[wikilinks]]`. Source of truth for content: `learning/review/ground-truth.md` and
`learning/review/gaps-contributions.md` (the line-by-line per-repo audit at HEAD `35d0f11`).

## Pages created

| Slug | Type | What it captures |
|---|---|---|
| `peat-current-state` | reference | **New canonical current-state page.** Transport shipped-vs-proposed matrix (§2), CRDT types (§3), four identity schemes (§4), hierarchy/roles (§5), crypto/FIPS posture (§6), control/data-plane reality (§7), verified-vs-unverifiable quantitative claims (§8), ADR status table (§9), and the gaps & contribution map (§10). Audited-commit table up front. Write-through file: `/Users/bryanbui/2ndBrain2.0/peat-current-state.md`. |

## Pages updated in place (not duplicated)

| Slug | Type | Change |
|---|---|---|
| `peat-hierarchy-vocabulary` | concept | **Corrected a material error.** The prior version claimed the canonical leaf tier is `Platform (0)` and that ADR-066 is authoritative. Both are false: shipped enums (`peat-mesh/src/beacon/types.rs:56-67`, `peat-protocol/.../authorization.rs:331-343`) use **`Node`** as the leaf, and **ADR-066 is Status: Proposed**. Rewrote to present (a) what ships today — leaf is `Node`, workspace mixes four vocabularies; (b) ADR-066/ADR-068 as the *proposed* intended vocabulary; (c) legacy variants; and added a note not to conflate hierarchy tiers with the three role/authority enums. Added a prominent CRITICAL CORRECTION callout. Write-through file: `/Users/bryanbui/2ndBrain2.0/peat-hierarchy-vocabulary.md`. |
| `peat-open-issues-snapshot` | reference | Refreshed against HEAD `35d0f11`. Two key corrections: **#857 is ADR-046 Phase-4 LabelSelector, NOT tombstone GC** (tombstone GC is **peat-node#136**); hierarchy-rename epic set is now **#904 + #968 + #970** (ADR-068 base-unit convergence on `Node`). Reorganized per-repo issues to match the audit, elevated **peat-btle#75 (CMVP/aws-lc-rs)** as the top FIPS-readiness item, noted gateway is a control plane / not production-secure, peat-node gRPC unauthenticated (#38), peat-node speaks QUIC-only, and the "25 RPCs" → 28/27 correction. Preserved the starter-issues-by-role and contribution-gate sections. Write-through file: `/Users/bryanbui/2ndBrain2.0/peat-open-issues-snapshot.md`. |

## Pages deleted

None. (Per memory `keep-peat-docs-in-gbrain-default`: never delete Peat doc pages in `default`;
correct via update/config only.)

## Wikilink graph

- `peat-current-state` → links to `peat-hierarchy-vocabulary`, `peat-open-issues-snapshot`,
  `peat-modularity-leaf-crates`, `research/quic-iridium-sbd-feasibility`.
- `peat-hierarchy-vocabulary` → links to `peat-current-state`, `peat-open-issues-snapshot`,
  `peat-modularity-leaf-crates`, `peat-code-source-in-gbrain`.
- `peat-open-issues-snapshot` → links to `peat-current-state`, `peat-hierarchy-vocabulary`.

All link targets are existing pages (verified via `list_pages`/`search`). `put_page` reported
`status: created_or_updated` and `write_through.written: true` for all three.
