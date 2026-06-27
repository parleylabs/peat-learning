# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not a code project** — there is no build, compile, lint, or test step. It is the
**Peat learning curriculum**: an onboarding track for engineers adopting the Peat mesh protocol,
plus the machinery that keeps that curriculum honest against the Peat source code. The Peat code
itself lives in eight *other* repositories (`github.com/defenseunicorns/peat`, `peat-mesh`,
`peat-btle`, `peat-lite`, `peat-gateway`, `peat-node`, `peat-flutter`, `peat-sapient`) — none of it
is here. (`peat-flutter` = the Flutter/Dart client binding over `peat-ffi`; `peat-sapient` = the
SAPIENT sensor-standard bridge, both added to tracking 2026-06-25 — see their
`review/ground-truth/` records.)

The deliverable is therefore prose, not software: numbered markdown modules, two self-contained
HTML tracks, and a `review/` evidence trail. Edits are made by hand or by running one of the two
refresh prompts; correctness is checked by a human reading it against the Peat source, not by CI.

## Layout

- **`00`–`09` markdown modules** — the ordered curriculum. Read `00-START-HERE.md` first; it is
  the table of contents and the source of the project's framing.
- **`index.html`** — single-page HTML learning hub. It **mirrors the numbered modules** and must
  stay consistent with them.
- **`peat-constrained-networking.html`** — the advanced "Off the Grid" track (Peat where QUIC
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
  `design-note-discrepancies.md`, `CHANGELOG-review.md`, and `measurement.md` (how the site's
  cookieless analytics + feedback widget measure curriculum impact, and where the dashboard/issues
  live). `review/originals/` (pre-rewrite backups) is gitignored — git history is the canonical
  baseline once committed.

## How the curriculum stays current (the core workflow)

The curriculum is kept in sync with the code by a **scheduled, delta-driven review**, not by hand:

1. A cloud routine clones this repo into a directory named **`learning/`** alongside fresh clones of
   the eight Peat code repos as siblings (`<workspace>/learning/`, `peat/`, `peat-mesh/`, …). Many
   paths in the prompts and `REVIEW-STATE.json` are written `learning/...` for this reason.
2. `PROMPT-peat-curriculum-refresh.md` runs in **`ci` mode**: it reads `REVIEW-STATE.json`, diffs
   each code repo against the last audited commit, and re-audits + rewrites **only** the documents
   the changed code affects (see the routing table in that prompt's §2), **including the diagrams in
   those docs** (resolved through the diagram registry). No drift → no-op log entry.
3. Every ~30 days the refresh auto-escalates to a **full** sweep
   (`PROMPT-peat-training-doc-improvement.md`), which includes **Phase 6b — a diagram verification
   sweep** that re-derives *every* diagram in the registry against current code and advances its
   last-verified commit, exactly as Phase 2 re-audits every prose claim.
4. Changes land as a **pull request** with a run-log summary. Work happens on a branch, never on `main`.

When you edit the curriculum, treat these prompts as the spec for *how* edits are supposed to be
made — they encode the labeling, routing, and state-update discipline below.

## Non-negotiable house rules (apply to every edit)

These come from the Peat repos and the review prompts. Violating them is the main failure mode.

- **Code over everything.** When the Peat code, an ADR, a README, or existing curriculum prose
  disagree, the **code wins**. Cite `path:line`, ADR number, issue number, or commit. The Peat
  READMEs and specs lag the code (notably on crypto, role names, and versions) — never "fix" the
  curriculum to match a stale doc. **But for a crate consumed by a *published* version pin, the
  shipped truth is the published release (read a consumer lockfile + checksum), not just the git HEAD
  — flag when they diverge** (e.g. crates.io `peat-btle 0.4.0` shipped non-FIPS crypto its source had
  already migrated off). **Verify any cited `repo#NNN` against the repo before repeating it, and
  re-derive ADR status/counts each pass — never carry forward an unconfirmed issue, status, or count.**
- **Label every capability** with one of four visible tags: **Shipped** (in code, tested),
  **In-flight** (open issue/PR/epic), **Proposed** (an ADR in `Proposed` status, no code), or
  **Speculative** (invented for teaching, not anywhere). Presenting a proposal as if it ships is
  the worst error. Quantitative figures are cited or explicitly flagged as unverified.
- **FIPS-approved cryptographic primitives only.** Peat's *source* uses AES-256-GCM, ECDH P-256,
  Ed25519, HKDF-SHA-256, HMAC-SHA-256. ChaCha20-Poly1305 / X25519 in the READMEs are **stale docs** —
  never reintroduce them into prose — but they can still ride in a **published-but-not-re-published**
  crate (crates.io `peat-btle 0.4.0`), so distinguish *source* from *published artifact* precisely
  rather than claiming a blanket "FIPS-clean." Note also: FIPS-*approved algorithm* ≠ CMVP-validated
  *module* (the RustCrypto crates are the former, not the latter).
- **"Peat" is a name, not an acronym.** Never write all-caps `PEAT` for the protocol; only the
  `PEAT_` environment-variable prefix and the peat-lite `PEAT` wire-MAGIC literal are legitimately
  uppercase. The former project name was **HIVE** (a backronym), not a Peat expansion.
- **No vendor/consumer names in generic protocol prose** (use "consumer", "CoT consumer";
  protocol names like CoT/TAK are fine).
- **Preserve the "autonomy under human authority" framing** for any tasking/command content.
- **Keep the prose human.** Avoid AI-cadence ("X isn't Y, it's Z"; choppy three-beat sentences;
  forced rhetorical closers). Lead with concrete numbers.

## Conventions specific to the artifacts

- **The HTML pages are fully self-contained.** Every diagram is inline SVG; no external scripts,
  fonts, or network fetches — they must work offline and air-gapped. Do not add `<script src>`,
  CDN links, or runtime manifest fetches. Every diagram carries a legend.
  - **Approved exception — analytics + feedback widget (PRESERVE, do not strip).** A single
    cookieless GoatCounter beacon (`//gc.zgo.at/count.js`, `data-goatcounter=".../count"`) and the
    inline feedback widget (`<div id="fb">` 👍/👎 + GitHub-issue-prefill "report an edit") live in
    the footer of `index.html`, `peat-constrained-networking.html`, and `changelog.html`. This is a
    deliberate, signed-off relaxation: the beacon is *one async request that fails silently with no
    network*, so a downloaded/air-gapped copy still renders and works — it just doesn't report. The
    rule is therefore "no external assets **for rendering**, plus this one privacy-light beacon."
    The refresh/review gates must **keep** this block on rewrites (don't flag it as a
    self-containment violation), preserve the `id="fb"` markup + the SPA `active()`/`gcView()`
    wiring, and must **not** introduce any *other* external resource. The `YOURCODE` in
    `data-goatcounter` is the registered site code; never overwrite a real code back to `YOURCODE`.
- **The hub mirrors the modules.** Any change to a numbered module must be reflected in
  `index.html`, and vice versa. Module numbering and order are load-bearing.
- **Freshness stamps are inline and updated by the refresh, not at runtime.** When commits or
  status change, update the block between `<!--SYNC-->` and `<!--/SYNC-->` in `index.html`,
  `peat-constrained-networking.html`, and `changelog.html`; in `changelog.html` also prepend one
  `<tr>` immediately after `<!-- CHANGELOG-ROWS-START -->`. Never invent a date — use the date the
  scheduler passes.
- **Visual standard (`DESIGN-SYSTEM.md`).** All reader pages follow the adapted Visual Explainer standard
  in `learning/DESIGN-SYSTEM.md` (live reference: `DESIGN-SYSTEM.html`): the component system (cards,
  status badges, the data-table pattern for any 4+-row / 3+-column table), accessibility (semantic HTML,
  `prefers-reduced-motion`, no emoji for status), and **light + dark** via `prefers-color-scheme`. It is
  bounded by the two rules above — **self-contained** (no CDN/runtime/external font; pre-render Mermaid to
  inline SVG) and the **Parley Labs brand + status palette**. A *visual & design gate* enforces it on
  every refresh; the monthly sweep migrates legacy visuals incrementally (never big-bang).

## Diagrams

Diagrams are **code-derived claims, not decoration** — they encode the most drift-prone facts
(layer model, transport set, role/enum lists, hierarchy vocabulary per ADR-066, versions,
ports). That drift is invisible to a prose diff because the facts live inside SVG coordinates
and ASCII box-art, so diagrams get the same discipline as text.

- **Formats.** Markdown structural/flow diagrams use ```mermaid (diffable, GitHub-rendered);
  plain-fence ASCII art is for tiny inline sketches only **and is a migration target** (prefer a
  mermaid twin so both copies are real, diffable diagrams). HTML diagrams are hand-authored
  inline SVG and must stay **self-contained** — no mermaid.js, CDN, or runtime fetch. To use a
  mermaid source in the HTML, pre-render it to static SVG and inline the result. **Authoring hygiene**
  (mermaid source): `flowchart TD` for 5+ nodes / branching (`LR` only for 3–4-node linear flows);
  **max ~10–12 nodes** — beyond that split into a hybrid (mermaid overview + a CSS card grid); `<br/>`
  for multi-line labels (never `\n`); quote labels; alphanumeric IDs.
- **Re-derive, don't just preserve.** When the code a diagram depicts changes, regenerate the
  diagram's facts — updating the surrounding prose while leaving the old SVG/ASCII is the
  common silent bug. Diagram nodes carry the **shipped/in-flight/proposed/speculative** status
  (color + legend / interactive filter), not only the prose around them.
- **One status palette across all pages:** Shipped = green, In-flight = amber, Proposed = blue,
  Speculative = purple (exact hex in `review/diagrams.md`). Error/rejected states use a separate
  red and are not a status. Keep `index.html` and `peat-constrained-networking.html` in sync.
- **Light mode = dark figure-plate (decision A).** Diagrams always sit on a fixed **dark surface card**
  in both light and dark page modes, so existing inline-SVG colors stay valid with **zero rework**. Full
  theme-adaptive SVGs are optional/opt-in per registry row, never required. HTML diagrams get inline
  **zoom / pan / reset / expand** controls (inline JS only — no CDN). Full rules: `DESIGN-SYSTEM.md` §4.
- **Watch the dual-copy hazard.** The same concept is sometimes drawn twice — as hub SVG and
  as module ASCII (e.g. the layer model, the dependency graph). Those twins must agree.
- **The registry is the source of record.** `review/diagrams.md` lists every diagram with its
  id, location, concept, code provenance (`path:line`/ADR), twin, and last-verified commit, plus a
  prioritized **Proposed diagrams (backlog)**. The refresh prompts route through it: changed code →
  affected diagrams → re-derive → advance the row. Add a row whenever you author a new diagram. A
  **full sweep re-verifies every row** against current code (Phase 6b), not only diagrams whose
  prose changed, and advances each last-verified commit; unconfirmable diagram facts go to
  `unverifiable_claims`.

## When running a refresh / making code-driven edits

- Read `REVIEW-STATE.json` first: `audited_commits` (the baseline to diff against), `open_todos`,
  and `unverifiable_claims` tell you what's known-stale and what couldn't be confirmed.
- After changes, **advance state**: update `audited_commits[repo].head` for changed repos, append a
  `run_log` entry, and append (never overwrite) a dated section to `review/CHANGELOG-review.md`.
- Anything you couldn't verify against code goes into `unverifiable_claims` — logged, not hidden.
- **Diagrams are audited like claims:** re-derive every affected diagram (every diagram on a full
  sweep), advance its row in `review/diagrams.md`, keep the canonical status palette, and log any
  unconfirmable diagram fact.
- Rewrite only the sections the changed code touches; leave untouched modules alone.
- **Run the impact lens before finishing** (see both prompts' impact/blast-radius check): an edit to
  one section must not silently break another. Check hub↔module mirroring, diagram twins,
  cross-references and internal links/anchors, module numbering/order, the `<!--SYNC-->` stamps, and
  that the HTML stays self-contained with valid SVG/mermaid. Fix or flag every impact you create.
