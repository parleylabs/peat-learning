# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not a code project** — there is no build, compile, lint, or test step. It is the
**PEAT learning curriculum**: an onboarding track for engineers adopting the PEAT mesh protocol,
plus the machinery that keeps that curriculum honest against the PEAT source code. The PEAT code
itself lives in six *other* repositories (`github.com/defenseunicorns/peat`, `peat-mesh`,
`peat-btle`, `peat-lite`, `peat-gateway`, `peat-node`) — none of it is here.

The deliverable is therefore prose, not software: numbered markdown modules, two self-contained
HTML tracks, and a `review/` evidence trail. Edits are made by hand or by running one of the two
refresh prompts; correctness is checked by a human reading it against the PEAT source, not by CI.

## Layout

- **`00`–`09` markdown modules** — the ordered curriculum. Read `00-START-HERE.md` first; it is
  the table of contents and the source of the project's framing.
- **`index.html`** — single-page HTML learning hub. It **mirrors the numbered modules** and must
  stay consistent with them.
- **`peat-constrained-networking.html`** — the advanced "Off the Grid" track (PEAT where QUIC
  can't run).
- **`changelog.html`** — reader-facing change history.
- **`PROMPT-peat-training-doc-improvement.md`** — the full, code-anchored adversarial review
  prompt (the deep sweep).
- **`PROMPT-peat-curriculum-refresh.md`** — the cheap, git-delta-driven incremental refresh prompt
  (scheduled). Inherits all rules from the full prompt.
- **`REVIEW-STATE.json`** — the state anchor: per-repo audited commits, open todos, unverifiable
  claims, and the run log. The refresh reads and advances this.
- **`review/`** — the audit trail: `ground-truth.md` (+ per-repo `ground-truth/`), `claim-ledger.md`
  (+ per-document `ledger/`), `assumptions.md`, `gaps-contributions.md`, `use-cases.md`,
  `design-note-discrepancies.md`, `CHANGELOG-review.md`. `review/originals/` (pre-rewrite backups)
  is gitignored — git history is the canonical baseline once committed.

## How the curriculum stays current (the core workflow)

The curriculum is kept in sync with the code by a **scheduled, delta-driven review**, not by hand:

1. A cloud routine clones this repo into a directory named **`learning/`** alongside fresh clones of
   the six PEAT code repos as siblings (`<workspace>/learning/`, `peat/`, `peat-mesh/`, …). Many
   paths in the prompts and `REVIEW-STATE.json` are written `learning/...` for this reason.
2. `PROMPT-peat-curriculum-refresh.md` runs in **`ci` mode**: it reads `REVIEW-STATE.json`, diffs
   each code repo against the last audited commit, and re-audits + rewrites **only** the documents
   the changed code affects (see the routing table in that prompt's §2). No drift → no-op log entry.
3. Every ~30 days the refresh auto-escalates to a **full** sweep
   (`PROMPT-peat-training-doc-improvement.md`).
4. Changes land as a **pull request** with a run-log summary. Work happens on a branch, never on `main`.

When you edit the curriculum, treat these prompts as the spec for *how* edits are supposed to be
made — they encode the labeling, routing, and state-update discipline below.

## Non-negotiable house rules (apply to every edit)

These come from the PEAT repos and the review prompts. Violating them is the main failure mode.

- **Code over everything.** When the PEAT code, an ADR, a README, or existing curriculum prose
  disagree, the **code wins**. Cite `path:line`, ADR number, issue number, or commit. The PEAT
  READMEs and specs lag the code (notably on crypto, role names, and versions) — never "fix" the
  curriculum to match a stale doc.
- **Label every capability** with one of four visible tags: **Shipped** (in code, tested),
  **In-flight** (open issue/PR/epic), **Proposed** (an ADR in `Proposed` status, no code), or
  **Speculative** (invented for teaching, not anywhere). Presenting a proposal as if it ships is
  the worst error. Quantitative figures are cited or explicitly flagged as unverified.
- **FIPS-approved cryptographic primitives only.** PEAT shipped AES-256-GCM, ECDH P-256, Ed25519,
  HKDF-SHA-256, HMAC-SHA-256. References to ChaCha20-Poly1305 / X25519 are **stale docs**, not
  shipped behavior — do not reintroduce them.
- **No vendor/consumer names in generic protocol prose** (use "consumer", "CoT consumer";
  protocol names like CoT/TAK are fine).
- **Preserve the "autonomy under human authority" framing** for any tasking/command content.
- **Keep the prose human.** Avoid AI-cadence ("X isn't Y, it's Z"; choppy three-beat sentences;
  forced rhetorical closers). Lead with concrete numbers.

## Conventions specific to the artifacts

- **The HTML pages are fully self-contained.** Every diagram is inline SVG; no external scripts,
  fonts, or network fetches — they must work offline and air-gapped. Do not add `<script src>`,
  CDN links, or runtime manifest fetches. Every diagram carries a legend.
- **The hub mirrors the modules.** Any change to a numbered module must be reflected in
  `index.html`, and vice versa. Module numbering and order are load-bearing.
- **Freshness stamps are inline and updated by the refresh, not at runtime.** When commits or
  status change, update the block between `<!--SYNC-->` and `<!--/SYNC-->` in `index.html`,
  `peat-constrained-networking.html`, and `changelog.html`; in `changelog.html` also prepend one
  `<tr>` immediately after `<!-- CHANGELOG-ROWS-START -->`. Never invent a date — use the date the
  scheduler passes.

## Diagrams

Diagrams are **code-derived claims, not decoration** — they encode the most drift-prone facts
(layer model, transport set, role/enum lists, hierarchy vocabulary per ADR-066, versions,
ports). That drift is invisible to a prose diff because the facts live inside SVG coordinates
and ASCII box-art, so diagrams get the same discipline as text.

- **Formats.** Markdown structural/flow diagrams use ```mermaid (diffable, GitHub-rendered);
  plain-fence ASCII art is for tiny inline sketches only. HTML diagrams are hand-authored
  inline SVG and must stay **self-contained** — no mermaid.js, CDN, or runtime fetch. To use a
  mermaid source in the HTML, pre-render it to static SVG and inline the result.
- **Re-derive, don't just preserve.** When the code a diagram depicts changes, regenerate the
  diagram's facts — updating the surrounding prose while leaving the old SVG/ASCII is the
  common silent bug. Diagram nodes carry the **shipped/in-flight/proposed/speculative** status
  (color + legend), not only the prose around them.
- **Watch the dual-copy hazard.** The same concept is sometimes drawn twice — as hub SVG and
  as module ASCII (e.g. the layer model, the dependency graph). Those twins must agree.
- **The registry is the source of record.** `review/diagrams.md` lists every diagram with its
  id, location, concept, code provenance (`path:line`/ADR), twin, and last-verified commit.
  The refresh prompts route through it: changed code → affected diagrams → re-derive → advance
  the row. Add a row whenever you author a new diagram.

## When running a refresh / making code-driven edits

- Read `REVIEW-STATE.json` first: `audited_commits` (the baseline to diff against), `open_todos`,
  and `unverifiable_claims` tell you what's known-stale and what couldn't be confirmed.
- After changes, **advance state**: update `audited_commits[repo].head` for changed repos, append a
  `run_log` entry, and append (never overwrite) a dated section to `review/CHANGELOG-review.md`.
- Anything you couldn't verify against code goes into `unverifiable_claims` — logged, not hidden.
- Rewrite only the sections the changed code touches; leave untouched modules alone.
</content>
</invoke>
