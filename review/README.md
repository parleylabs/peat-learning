# Peat Curriculum Review — Artifact Index

This directory holds the deliverables of the Peat learning-curriculum hardening review (prompt:
`learning/PROMPT-peat-training-doc-improvement.md`). The review audited the curriculum claim by claim
against the latest Peat source, rewrote every document in place, and fed the verified current state
back into gbrain. The rewritten curriculum lives in `learning/` (the parent directory); this directory
holds the evidence trail, working artifacts, and the pristine originals.

## Top-level artifacts

| Artifact | What it is |
|---|---|
| `ground-truth.md` | **Ground-Truth Appendix** — the verified current-state model with citations: audited commit per repo, transport shipped-vs-proposed matrix, real CRDT/identity/hierarchy/crypto facts, quantitative claims traced or flagged, ADR status table, open epics. All later claims reference this. |
| `claim-ledger.md` | **Merged Claim Ledger** — every audited curriculum claim with verdict (Wrong/Misleading/Outdated/Oversimplified/Speculative-unlabeled/Unverifiable/Verified) + evidence (`path:line` / ADR / issue) + fix. Merged from the 14 per-document ledgers in `ledger/`. Sorted Wrong/Misleading first. |
| `assumptions.md` | **Assumptions Register** — every premise the curriculum (or the audit) relies on, where it is relied on, whether it holds at the audited HEADs, and the consequence if false. |
| `gaps-contributions.md` | **Gaps & Contribution Map** — what does not exist yet, why it matters, the linked ADR/issue, rough size, and how a new contributor would start. Doubles as "where to contribute." |
| `use-cases.md` | **Use Cases** — five worked end-to-end use cases (TAK/CoT operator picture; UGV joining + tasking; maritime/BLOS relay; disaster-response disconnected cell; remote sensor field), each grounded in shipped code and labeled shipped/in-flight/proposed/speculative leg by leg. |
| `design-note-discrepancies.md` | **Design-Note Discrepancy List** — cross-check of `peat-addressing-transport-sync.md` against ground truth (claim → reality → correction), with corrected status labels. |
| `gbrain-update-summary.md` | **gbrain Update Summary** — pages created/updated in the `default` source (slugs), the wikilink graph, and the no-deletion note. |
| `CHANGELOG-review.md` | **Change Log** — per-document record of what was corrected in each curriculum file and why, plus the consolidated list of claims that remain UNVERIFIABLE. |

## Sub-directories

| Directory | What it holds |
|---|---|
| `ground-truth/` | Per-repo ground-truth audits: `peat.md`, `peat-mesh.md`, `peat-btle.md`, `peat-lite.md`, `peat-gateway.md`, `peat-node.md`. The line-by-line source reads behind the merged `ground-truth.md`. |
| `ledger/` | Per-document claim ledgers (one per curriculum file): `hub.md`, `constrained.md`, and `00-start-here.md` … `09-specs.md`. Merged into the top-level `claim-ledger.md`. |
| `originals/` | **Pristine pre-rewrite copies** of every rewritten curriculum file (the 14 docs + `peat-addressing-transport-sync.md`). Used for revert/diff. |

## How to revert

The rewritten curriculum files live in `learning/` (the parent directory). Their pre-rewrite originals
are preserved in `learning/review/originals/`. To restore any document to its pre-review state, copy it
back from `originals/`, e.g.:

```sh
# from the repo root
cp learning/review/originals/02-peat-protocol.md        learning/02-peat-protocol.md
cp learning/review/originals/peat-learning-hub.html      learning/peat-learning-hub.html
cp learning/review/originals/peat-constrained-networking.html learning/peat-constrained-networking.html
```

To revert the whole curriculum at once:

```sh
# HTML tracks + markdown modules
cp learning/review/originals/peat-learning-hub.html          learning/
cp learning/review/originals/peat-constrained-networking.html learning/
cp learning/review/originals/0*.md                           learning/
# the design note (secondary doc; not rewritten, validated only)
cp learning/review/originals/peat-addressing-transport-sync.md learning/
```

gbrain changes are **not** reverted by the above — see `gbrain-update-summary.md`. Per memory
`keep-peat-docs-in-gbrain-default`, Peat doc pages in the `default` source are never deleted; correct
them via `put_page` update, not deletion.
