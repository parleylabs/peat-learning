<img src="assets/peat-wordmark.png" alt="Peat" width="200">

# Peat Learning

Onboarding curriculum for new teams adopting the **Peat** mesh protocol, plus the
machinery that keeps it honest against the code.

## What's here

- **`index.html`** — single-page learning hub and site landing page (mirrors the numbered modules).
- **`peat-constrained-networking.html`** — advanced "Off the Grid" track (running Peat where QUIC can't).
- **`00`–`09` markdown modules** — the ordered curriculum (start at `00-START-HERE.md`).
- **`PROMPT-peat-training-doc-improvement.md`** — the full, code-anchored adversarial review prompt.
- **`PROMPT-peat-curriculum-refresh.md`** — the cheap, git-delta-driven incremental refresh prompt (scheduled).
- **`REVIEW-STATE.json`** — the state anchor: per-repo audited commits + run log. The refresh reads this.
- **`review/`** — audit trail and deliverables (ground truth, claim ledger, assumptions, gaps, use cases, change log) and `review/originals/` (pre-rewrite backups for one-line revert).

## How the docs stay current

The curriculum is kept in sync with the code by a scheduled review, not by hand:

1. A weekly cloud routine clones this repo into a directory named **`learning/`** alongside fresh clones
   of the six Peat code repos as siblings:
   ```
   <workspace>/
     learning/        ← this repo
     peat/  peat-mesh/  peat-btle/  peat-lite/  peat-gateway/  peat-node/   ← github.com/defenseunicorns/*
   ```
2. It runs `learning/PROMPT-peat-curriculum-refresh.md` in **`ci` mode**: reads `REVIEW-STATE.json`,
   diffs each code repo against the last audited commit, and re-audits + rewrites **only** the
   documents the changed code affects. No drift → no-op.
3. Every ~30 days the refresh auto-escalates to a **full** sweep (`PROMPT-peat-training-doc-improvement.md`).
4. Changes land as a **pull request** against this repo, with a run-log summary.

Local runs (with the gbrain MCP available) additionally refresh the `peat-current-state` and
`peat-open-issues-snapshot` gbrain pages; the cloud routine skips that step (no local gbrain).

## Source of truth

The Peat **code** is always authoritative. Every claim in the curriculum is labeled
**shipped / in-flight / proposed / speculative**, and quantitative figures are cited or flagged.
See `review/ground-truth.md` for the audited current-state model and `review/CHANGELOG-review.md`
for the history of corrections.

## House rules (inherited from the Peat repos)

- FIPS-approved cryptographic primitives only.
- No vendor/consumer names in generic protocol prose ("consumer", "CoT consumer" — protocol names like CoT are fine).
- "Autonomy under human authority" framing preserved for any tasking/command content.
