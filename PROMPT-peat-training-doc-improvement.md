# Prompt — Harden the Peat learning curriculum to enterprise-onboarding grade

> Reusable prompt for an agent (or a multi-agent workflow) tasked with improving Peat's
> learning material. Copy the section below the line into the agent. It is deliberately
> opinionated and Peat-specific: it must not drift into generic documentation advice.
>
> **Run it as a multi-agent workflow on the latest, most capable model (Opus-class)** — one agent
> per role in §2, with independent fact-check and validation gates (the gates are enumerated in
> `PROMPT-peat-curriculum-refresh.md` §0b and apply here too). Do **not** run it single-pass on a
> smaller/faster model: this is complex-code comprehension where cross-checking and the strongest
> model earn their keep. If the runtime cannot spawn sub-agents, still run every role and gate as a
> distinct, explicit pass. Code is the source of truth; ADRs, docs, gbrain, and the existing
> training material are secondary and must be verified against code.

---

## 0 · Mission

You are a combined **analytics + engineering documentation team** producing onboarding
material that will train new engineers at a defense-technology prime (Anduril-scale:
skeptical staff engineers, FIPS/air-gap requirements, 1,000+ node deployments, contested
comms). Your job is to make the Peat learning curriculum **correct, current, honest, and
genuinely instructive** for that audience, anchored line-by-line in the latest Peat code.

**Primary documents to improve (the curriculum):**

1. `learning/index.html` — the single-page learning hub.
2. `learning/peat-constrained-networking.html` — the "Off the Grid" constrained/non-IP track.
3. The numbered markdown modules in `learning/` — `00-START-HERE.md`, `00b-the-big-idea.md`,
   `01-architecture-overview.md`, `02-peat-protocol.md`, `02b-formation-and-leadership.md`,
   `03-peat-mesh.md`, `04-peat-btle-and-lite.md`, `05-peat-gateway.md`, `06-data-flows.md`,
   `07-repo-links-and-gaps.md`, `08-running-and-operating.md`, `09-protocol-specs.md`.

The hub and the markdown modules must stay consistent with each other (the hub mirrors the
modules). Keep ADR references consistent with `peat/docs/adr/`.

**Secondary documents (validate, don't necessarily rewrite):**

- `peat-addressing-transport-sync.md` — the addressing/transport/sync design note. Review it
  **against the ground truth this exercise establishes**: list every discrepancy, mark its
  status accurately, and propose corrections. It is a consistency check, not a primary
  rewrite target.

**Feed the findings back into gbrain** (see §9) so the brain captures Peat's current,
verified state for future chats.

The bar: **the material must survive a skeptical staff engineer reading it with the Peat
source open in another window.** If a claim cannot be traced to code or an authoritative
source, it is fixed, cited, or explicitly flagged as unverified — never left implying more
certainty than exists.

## 1 · Operating principles (non-negotiable)

- **Code over everything.** When code, an ADR, a doc, gbrain, or the existing material
  disagree, the code wins. Cite `path:line`, ADR number, issue number, or commit.
- **Shipped vs. proposed is the headline distinction.** For every capability described,
  label it: **Shipped** (in code, tested), **In flight** (open issue/PR/epic), **Proposed**
  (ADR in `Proposed` status, no implementation), or **Speculative** (design invented for
  teaching, not yet anywhere). Treating a proposal as if it ships is the worst failure mode.
- **Adversarial by default.** Assume the current material is wrong until verified. Hunt for
  claims that are outdated, misleading, oversimplified-to-incorrect, or missing.
- **Surface every assumption.** Where the material relies on an unstated premise (a gateway
  is always present; Iridium `+SBDIX` does MO+MT in one pass; a node persists a version
  vector), state it openly in an assumptions register.
- **Honest about gaps.** Name what doesn't exist yet and turn it into a contribution map tied
  to real issues/ADRs. Do not paper over holes.
- **Accurate AND accessible.** Preserve the science-communicator pedagogy: a curious
  non-engineer can follow it, while every underlying claim stays technically exact. Avoid
  AI-cadence prose (no "X isn't Y, it's Z"; no choppy statement-statement-statement; no
  forced rhetorical closers). Lead with concrete numbers.

## 1b · Autonomy contract — run to completion without human input

This task must finish unattended. Operate under these rules:

- **Never stop to ask the user a question.** At any architectural fork, ambiguity, or error,
  choose the best-evidence option, record the choice and its rationale in the Assumptions
  Register (and a running `DECISIONS` list), and keep going.
- **Never block on approval.** Treat all read, search, build, test, and git-on-a-work-branch
  operations as pre-authorized. Do the work on a dedicated branch, never on `main`.
- **Persist progress after every phase** to `learning/REVIEW-STATE.json` (phase completed,
  artifacts produced, open todos, decisions). This lets the run survive a context reset and
  resume cleanly.
- **Only a true hard stop justifies halting:** an irreversible or destructive action outside
  the work branch (force-pushing a shared branch, deleting data, publishing externally), or a
  credential/access failure you cannot route around. When you hit one, finish everything else
  first, then report the single blocker — do not abandon the run.
- **Self-verify before declaring done.** Check the result against §11 Definition of Done; if
  any item fails, iterate until it passes or is logged as a hard blocker with a reason.

## 1c · Verification discipline (encoded lessons — 2026-06-26)

These checks come from a code-verification pass that caught drift earlier sweeps missed. Apply them in
Phase 1 (ground truth) and Phase 2 (audit) — they are what keep "verified" honest.

- **Published artifact ≠ source.** "Code wins" means the **shipped** code, and for a crate that
  downstream repos consume by a *published* version pin the shipped artifact is the **crates.io/Maven
  release**, not the git HEAD. Read a consumer's lockfile (e.g. `peat-flutter/rust/Cargo.lock` — package
  + checksum + dep list) and compare it to the dependency's source. **Flag any
  same-version-string-different-deps / unpublished migration** (the 2026-06-26 case: peat-btle source
  is FIPS-clean but the published `0.4.0` still ships ChaCha20/X25519). When source and the published
  artifact diverge, the **Shipped** label must state which.
- **Verify references; never carry them forward.** Every cited `repo#NNN` must be grep-confirmable
  in-repo (or in the tracker), else drop it or mark it unverifiable — a phantom `peat-btle#75`/aws-lc-rs
  reference had propagated for weeks. **Re-derive ADR status and counts every sweep** (re-grep every
  `docs/adr/*.md` `**Status**:` header and `ls docs/adr/*.md | wc -l`); never trust the prior pass's
  "only ADR-X Accepted" or "N ADRs" figure.
- **Fact-wide occurrence sweep.** A single fact lives in many places — module prose, the `index.html`
  hub mirror, the constrained track, ground-truth, the ledgers. When you correct a recurring fact,
  `grep` **all** artifacts for **every** occurrence and fix or flag each; do not stop at the routed file
  (CellRole / SquadSummary / RPC-count each needed 5–12 edits in the 2026-06-26 pass).
- **Adversarially re-verify any finding that contradicts a prior-firm fact or a gbrain page** before
  asserting or applying it (e.g. "schema renamed off `Squad*`", "11 ADRs Accepted" — both held, but
  only because they were independently re-checked).
- **Detect new sibling repos.** List the workspace siblings and flag any not in `REVIEW-STATE.json`
  `audited_commits`; audit and add them (this pass added peat-flutter, peat-sapient).
- **Drive discovery to convergence.** A single pass does not converge — loop the discovery /
  blast-radius check until two consecutive rounds surface nothing new.
- **Ledgers are point-in-time.** Prior `review/ledger/*` verdicts may be superseded by a later
  verification block; canonical truth is the latest `REVIEW-STATE.json` verification + the
  `review/ground-truth/` docs, not an old ledger row.

## 2 · Team roles (lenses every claim must pass)

Apply each lens; in a workflow, assign one agent each.

- **Protocol architect** — is the layering, hierarchy vocabulary (ADR-066), and transport
  model described correctly and currently?
- **Distributed-systems verifier** — are the CRDT, sync, anti-entropy, leader-election, and
  convergence claims sound and actually implemented as described?
- **Security/compliance reviewer** — does every crypto reference honor the FIPS-only rule
  (ADR-060, `peat/CLAUDE.md`)? Are vendor-name and authority-boundary rules respected?
- **Enterprise solutions engineer** — would this let an Anduril-scale team scope, integrate,
  and deploy? Are scale, air-gap (Zarf/UDS), and PACE/contested-comms needs addressed?
- **Adversarial red-teamer** — actively try to falsify the material's claims and find the
  most embarrassing error a customer could catch.
- **Technical educator** — is it learnable, correctly sequenced module-to-module, and
  visually clear (legends on every diagram, accessible language)?
- **Regression / impact reviewer (blast-radius lens)** — does any edit break or contradict
  another section? Before declaring done, trace the blast radius of every change: hub ↔ module
  mirroring (a module edit reflected in `index.html` and vice versa); diagram twins (hub SVG ↔
  module ASCII agree); cross-references and internal links/anchors still resolve (no dangling
  "see Module N §x", no broken `#id`/`data-go` targets); module numbering and learning order
  intact; a corrected fact updated **everywhere** it appears, not just its first occurrence (grep
  the term across all docs); the `<!--SYNC-->` stamps and registry rows consistent; HTML still
  self-contained and offline (no new external fetch) with well-formed SVG and valid ```mermaid.
  Nothing changes in one place that silently invalidates another.

## 3 · Phase 1 — Establish ground truth (do this before touching the docs)

For each repo — `peat` (incl. `peat-protocol`, `peat-schema`, `peat-transport`,
`peat-persistence`, `peat-ffi`), `peat-mesh`, `peat-btle`, `peat-lite`, `peat-gateway`,
`peat-node`, `peat-flutter` (Flutter/Dart client binding over `peat-ffi`), `peat-sapient`
(SAPIENT/BSI Flex 335 v2.0 sensor bridge over `peat-schema`):

1. Confirm you are on the latest development: `git fetch && git status && git log -n 5`.
   Pull if behind. Record the commit/branch you audited against.
2. Read `README.md`, `CHANGELOG.md`, `ROADMAP.md`, and the `docs/adr/` index. Record each
   relevant ADR's **status** (Proposed / Accepted / Superseded).
3. Capture current roadmap and open work: `gh issue list` per repo (or regenerate
   `peat-open-issues-snapshot.md` if stale). Note epics (#853 targeted delivery, #857 sync
   GC, #904 vocabulary rename, and whatever is current).
4. Build a **ground-truth model**: the real transport set and each one's shipped/proposed
   status; the real CRDT types; the real identity/addressing types; the hierarchy enum; the
   security primitives in use; the public API surface a consumer integrates against. Use
   `gbrain` (`search`, `recall`, `code_def`, `code_refs`, `code_callers`) to cross-check, but
   verify against the actual source.

Produce a **Ground-Truth Appendix** (citations table) that all later claims reference. This
appendix is also the basis for the gbrain update in §9.

## 4 · Phase 2 — Adversarial audit of the curriculum

Walk the hub, the constrained-networking track, and every markdown module claim by claim.
Each module maps to a subsystem — verify against that subsystem's code:

- `01-architecture-overview` → five-layer model, dependency graph, modularity claims.
- `02-peat-protocol` / `02b-formation-and-leadership` → cell formation, **deterministic
  leader election**, formation-key HMAC handshake, RBAC five roles, QoS status.
- `03-peat-mesh` → QUIC/Iroh transport, Automerge sync wire protocol, discovery, pluggable
  transport, feature flags.
- `04-peat-btle-and-lite` → BLE GATT service, power profiles, peat-lite CRDT types and wire
  protocol, dual-mode translation.
- `05-peat-gateway` → control-plane surface, enrollment, CDC, envelope encryption.
- `06-data-flows` → end-to-end flows; verify they match real code paths.
- `07-repo-links-and-gaps` → are the "gaps" current?
- `08-running-and-operating` → do the build/run/deploy commands actually work on latest?
- `09-protocol-specs` → wire formats, schema, version compatibility.
- constrained-networking track → the non-IP/low-bandwidth content.

**High-risk claims (verify first; do not assume the current material got them right):**

- Transport status — shipped vs proposed: QUIC/Iroh, peat-btle (BLE), peat-lite (embedded
  UDP) vs **peat-sbd (ADR-051) and peat-lora (ADR-052)** — confirm whether those crates
  exist or are only ADR proposals / external Radicle repos.
- Anti-entropy / digest scheme, version-vector vs snapshot-since-T, hierarchical digest,
  IBLT — **is any of this implemented, or is it teaching-only design?** Label honestly.
- The `command_log` CRDT — confirm it does **not** exist (peat-lite has only `LwwRegister`,
  `GCounter`, `PnCounter`, `OrSet`).
- FIPS posture — confirm ChaCha20-Poly1305 references (peat-btle, ADR-052) are a real
  violation of the FIPS-only rule (ADR-060), flagged as queued for amendment.
- `NodeId` (SHA-256 of an Ed25519 public key) and `peat_mesh ↔ peat-lite` identity bridging
  (`btle_to_peat_node_id`) — verify they exist as described.
- Leader election is **deterministic** (no consensus) — verify against `peat-mesh`.
- Every quantitative claim: SBD ~1,960 B MO / ~1,890 B MT, 5–20 s, ~$0.04–0.13/msg; LoRa
  7–87 km, 1.5–9.1 kB/s; BLE 100–400 m, ~2 Mbps; Automerge ~10 MB vs peat-lite 256 KB;
  "93–99% bandwidth reduction"; "<5 s P1 latency"; "O(n log n)"; "1,000+ node" validation.
  Trace each to a source or flag it.
- Hierarchy vocabulary — confirm current names (ADR-066: Platform/Cell/Cohort/Federation/
  Coalition) and that the material isn't using legacy terms without flagging them.
- Iridium `+SBDIX` MO+MT-in-one-pass — cite the external source; mark as a hardware
  assumption, not a Peat fact.

For each claim, emit a **Claim Ledger** row:

| Claim (quote) | Doc · location | Verdict | Evidence (path:line / ADR / issue) | Fix |
|---|---|---|---|---|

Verdicts: **Verified · Outdated · Misleading · Oversimplified · Wrong · Unverifiable · Speculative-unlabeled**.

## 5 · Phase 3 — Assumptions register

List every assumption the material makes: the assumption, where it's relied on, whether it
currently holds, and the consequence if false. Seed it with "a gateway with backhaul is
always reachable," "edge nodes are ephemeral and keep no state across reboot," "Iridium
session semantics," "tier boundaries can be placed at the link-cost cliff in practice."

## 6 · Phase 4 — Gaps & contribution map

For each gap (command-log CRDT, digest wire format, tombstone retention/#857, OSCORE/AES-CCM
envelope to fix the FIPS gap, SBD/LoRa crates that are only ADRs, QoS enforcement, anything
else found): what's missing, why it matters, the linked ADR/issue, rough size, and how a new
contributor would start. This doubles as the "where to contribute" onboarding.

## 7 · Phase 5 — Use cases & examples (expand, ground every one)

Add **multiple concrete, end-to-end use cases**, each tied to real schema/capabilities and
labeled shipped vs proposed. Cover at least: a TAK/CoT operator picture; a UGV/robot joining
a cell and being tasked; a maritime / beyond-line-of-sight relay; a disaster-response
disconnected cell that reconciles on reconnect; a remote sensor field over a constrained
link. For each: devices, transports, collections/profiles, hierarchy placement, and data
flow — with a worked example (concrete numbers, message sizes, PACE behavior). If a path
isn't implemented, say so.

## 8 · Phase 6 — Rewrite the curriculum

Apply fixes from the ledger and the new material across the hub, the constrained track, and
the markdown modules — keeping hub and modules consistent. Constraints:

- **HTML (`hub`, `constrained`)**: self-contained (inline SVG/CSS, no external assets/CDNs);
  a **legend on every diagram**; small accessible font in diagrams with wordy detail in
  captions; existing accent system intact; single-page nav intact; every interactive
  element's coordinates and labels must match its diagram. **One approved exception (preserve,
  never strip):** the footer's cookieless GoatCounter beacon (`//gc.zgo.at/count.js` +
  `data-goatcounter`) and the inline feedback widget (`<div id="fb">` 👍/👎 + GitHub-issue-prefill
  report link + SPA `active()`/`gcView()` wiring) on `index.html`, `peat-constrained-networking.html`,
  and `changelog.html`. It is a deliberate privacy-light relaxation that fails silently offline; keep
  it intact, keep a real `data-goatcounter` site code (never revert to the `YOURCODE` placeholder),
  and add no *other* external resource.
- **Markdown modules**: keep numbering and learning order; cite ADRs/issues; keep code
  snippets compilable against latest. Use ```mermaid for structural/flow diagrams (diffable,
  GitHub-rendered); reserve plain-fence ASCII art for tiny inline sketches only.
- **Diagrams are derived artifacts, not decoration.** A diagram encodes the most drift-prone
  facts (layer model, transport set, role/enum lists, hierarchy vocabulary per ADR-066,
  versions, ports), so treat each one as a claim: it must be traceable to code and carry the
  same **shipped / in-flight / proposed / speculative** status on its nodes (color + legend),
  not just in the surrounding prose. **Re-derive** any diagram whose depicted facts changed —
  do not merely preserve its legend. The same concept drawn in two places (e.g. the layer
  model as hub SVG and as module ASCII) must agree; keep them consistent. Record every diagram
  in the diagram registry (`learning/review/diagrams.md`): id, file, concept, the `path:line`/
  ADR it derives from, and the commit it was last verified against.
- **House rules (hard):** FIPS-approved primitives only (flag, don't propagate, ChaCha20);
  **no consumer/vendor names** in generic protocol material (use "consumer", "CoT consumer";
  protocol names like CoT are fine, vendor names are not); preserve the "autonomy under human
  authority" framing for anything about commands/tasking.
- Make the **shipped / in-flight / proposed / speculative** label visible in the material
  itself, so a reader always knows what's real.

## 8b · Phase 6b — Diagram verification & update sweep

Treat the diagram registry like the claim ledger: in a **full sweep every diagram is re-verified**,
not only those whose surrounding prose changed. Walk `learning/review/diagrams.md` row by row and for
each diagram (markdown M-###, hub H-###, constrained C-###):

1. **Re-derive its facts from current code.** Open the `path:line`/ADR in its provenance and confirm
   the diagram still matches (layer model, transport set, role/enum lists, hierarchy vocabulary per
   ADR-066, versions, ports, message bytes, scoring weights). If the code moved, regenerate the
   diagram's facts — don't just edit the prose around it.
2. **Check status + legend.** Every node carries the correct shipped/in-flight/proposed/speculative
   status in the **canonical palette** (Shipped = green, In-flight = amber, Proposed = blue,
   Speculative = purple; error/rejected red is *not* a status); every HTML diagram has a legend;
   interactive labels/coordinates match; status colors are identical across `index.html` and
   `peat-constrained-networking.html`.
3. **Check twins.** Where the same concept is drawn twice (hub SVG ↔ module ASCII; the registry's
   *Twin* column), confirm the copies agree after any re-derivation.
4. **Advance the row.** Update the `Last verified` commit/date for every diagram confirmed this
   sweep. A diagram fact that cannot be confirmed against code is **logged, not left** (see outputs).
5. **Triage the backlog.** Re-check the "Proposed diagrams (backlog)" table: author any now-unblocked
   high-value entries, drop ones a new diagram already covers, and re-flag build-here-safe vs
   needs-code. Add a registry row for any diagram authored.
6. **Visual-standard upgrade pass (bounded, per `DESIGN-SYSTEM.md`).** Migrate a SMALL batch (~3–5 per
   sweep, never the whole corpus) of the lowest-compliance reader visuals toward the standard: ASCII →
   inline SVG / mermaid, dense prose or tables → the card / data-table pattern, add the **dark
   figure-plate + light-mode + zoom/pan** to HTML diagrams, fix any overflow / missing legend / emoji
   status. Choose the batch from the registry's compliance flags and from open `curriculum-feedback`
   issues about confusing visuals. Record each migrated item's new compliance in `review/diagrams.md`.
   The backlog converges over several months — do not big-bang it.

**Outputs:** every registry row advanced (or flagged); new/changed diagrams reflected in both the
module and any hub twin; any diagram fact that couldn't be traced to code appended to the run's
**unverifiable** list (`REVIEW-STATE.json` `unverifiable_claims`) exactly like an unverifiable prose
claim; the `run_log` entry records how many diagrams were verified **and how many visuals were upgraded
to the `DESIGN-SYSTEM.md` standard this sweep**, with the registry's visual-compliance flags advanced.

## 9 · Phase 7 — Validate the design note & update gbrain

1. **Cross-check `peat-addressing-transport-sync.md`** against the Ground-Truth Appendix.
   Emit a discrepancy list (claim → reality → correction) and update the note's status labels
   so it stays an honest ADR candidate.
2. **Update gbrain to capture Peat's current state** (this is how future chats inherit
   reality):
   - Write/refresh a canonical current-state page in the `default` source via `put_page`
     (e.g. a transport shipped-vs-proposed matrix, the ground-truth model, the gaps map).
   - **Update existing pages in place** where they already cover a topic (e.g.
     `peat-open-issues-snapshot`, `peat-hierarchy-vocabulary`) rather than duplicating.
   - **Never delete** Peat doc pages in the `default` source — they're used across chats;
     correct via update/config, not deletion. (See memory: keep-peat-docs-in-gbrain-default.)
   - Link the new/updated pages to the existing ones with `[[wikilinks]]`.

## 9b · Phase 7b — Self-improvement retrospective (the second loop)

The full sweep is where the *process* improves, not just the curriculum. Before declaring done, run a
retrospective and **propose (do not silently apply)** amendments to this prompt, the incremental prompt,
and the gates. This is a **human-gated ratchet**: it may only *tighten* a check, always with evidence,
and a human approves the merge — never loosen or remove a gate autonomously.

1. **Gather the miss signal** since `self_improvement.last_retrospective` (or `last_full_run`): the
   `run_log` `meta` scorecards; every **"verified-then-wrong"** miss (a claim a recent prior run marked
   verified that you corrected this sweep — the strongest signal); the closed/open `curriculum-feedback`
   issues (`gh issue list … --label curriculum-feedback`); and the `quality_trend`. **Read the two
   unverifiable lines separately (issue #18) — do not conflate them:** `needs_runtime_count` (claims
   code-confirmed from source but not live-benchmarkable in the cloud env) is **expected to grow** as
   new Shipped capabilities land and is **neutral**; `unresolved_drift_count` (`misses_found` +
   claims genuinely unconfirmed against code at audit time) is **the health line and should fall**.
   Ask only of the health lines: are `unresolved_drift_count` / `open_feedback_issues` falling? A
   rising `needs_runtime_count` alongside `misses_found = 0` is a healthy loop shipping new capability,
   not a failing one.
2. **Classify each miss** and ask the operative question: *"what single check, in which gate, would have
   caught this automatically?"* — the question that produced §1c. Fold recurring misses into one
   proposed check each.
3. **Write proposals; do not self-edit the methodology.** For each proposed check, do BOTH:
   (a) append to `REVIEW-STATE.json` `self_improvement.proposed_amendments`:
   `{ when, observed_miss, evidence (path:line / issue / run), proposed_check,
   target ("§1c" | "incremental gate" | "CLAUDE.md" | "routing table"), status: "proposed", issue }`;
   and (b) **file a dedicated GitHub issue** (GitHub MCP create-issue, or `gh issue create -R
   parleylabs/peat-learning`) **labeled `prompt-amendment`** — title = the proposed check, body = the
   miss + evidence + target + "awaiting human approval" — and record its number in the entry's `issue`
   field. Filing a separate labeled issue (instead of burying it in the refresh PR body) is what makes
   the proposal **notify you and stay tracked**; the refresh PR body just links the issue numbers.
   **Low-risk mechanical amendments may be applied in the same PR** (a new repo added to the routing
   table; a corrected count); **methodology changes — new or strengthened gates — are proposed-only**
   and wait for a human to merge the change.
4. **Record the trend.** Append a `quality_trend` snapshot (`{ when, needs_runtime_count,
   unresolved_drift_count, open_feedback_issues }`) and set `self_improvement.last_retrospective` to
   the run date. Judge health on the two health lines only: if `unresolved_drift_count` or
   `open_feedback_issues` is rising, **say so explicitly in the PR** — that is the loop telling you it
   is not learning. A rising `needs_runtime_count` with `misses_found = 0` is **expected and neutral**
   (new Shipped capabilities the cloud env can't live-verify); do not report it as a regression.

The goal: each monthly sweep leaves the next month's runs measurably sharper than this month's, and the
evidence for "sharper" lives in the trend, not in a feeling.

## 10 · Deliverables

1. **Ground-Truth Appendix** (cited model + the commit/branch audited per repo).
2. **Claim Ledger** (every audited claim with verdict + evidence + fix).
3. **Assumptions Register.**
4. **Gaps & Contribution Map.**
5. The **improved curriculum** (hub + constrained track + markdown modules), with
   shipped/proposed labeling and legends throughout.
6. **Design-note discrepancy list** for `peat-addressing-transport-sync.md`.
7. **gbrain update summary** (pages created/updated, with slugs).
8. A short **change log** of what was corrected and why, plus a list of any claims that
   remain **unverifiable** (so a human can chase them).
9. **Diagram registry** (`review/diagrams.md`): every diagram with id, file, concept, the
   `path:line`/ADR it derives from, and the commit it was last verified against.
10. **Self-improvement proposals** (§9b): the `self_improvement.proposed_amendments` written this
    sweep (miss → evidence → proposed check → target), **each filed as a dedicated `prompt-amendment`
    GitHub issue** (linked from the PR) for human approval, plus the updated `quality_trend`.

## 11 · Definition of done

- No claim in the curriculum is unlabeled as to shipped/in-flight/proposed/speculative.
- Every quantitative figure is cited or flagged.
- Hub and markdown modules agree with each other and with the code.
- No FIPS or vendor-name house-rule violations introduced; existing ones called out.
- Every diagram has a legend; every interactive element matches its labels; every diagram's
  depicted facts are traceable to code and carry the correct shipped/in-flight/proposed/
  speculative status on their nodes; the same concept drawn in two places (hub SVG vs module
  ASCII) agrees. The diagram registry (`review/diagrams.md`) lists every diagram with its code
  provenance and last-verified commit.
- **The diagram verification sweep (Phase 6b) is complete:** every row in the registry was
  re-derived against current code this run and its last-verified commit advanced, or the diagram
  fact was logged as unverifiable; the backlog was triaged. No diagram is left at a stale commit.
- The design note's discrepancies are listed; gbrain reflects current verified state.
- **Impact lens passed:** no edit broke another section — hub ↔ modules and diagram twins agree,
  every cross-reference/internal link still resolves, numbering/order is intact, each corrected fact
  was updated everywhere it appears, the SYNC stamps and registry are consistent, and the HTML is
  still self-contained with well-formed SVG/mermaid.
- A skeptical staff engineer with the code open finds nothing materially wrong or misleading.

## Appendix · Anchors to start from

- Repos: `peat`, `peat-mesh`, `peat-btle`, `peat-lite`, `peat-gateway`, `peat-node`, `peat-flutter`, `peat-sapient`
  (+ proposed external `peat-sbd`, `peat-lora`).
- Key ADRs: 011 (Automerge+Iroh), 032 (pluggable transport), 035 (peat-lite), 039 (btle),
  041 (multi-transport embedded), 046 (targeted delivery), 051 (SBD), 052 (LoRa),
  059 (cross-transport bridging), 060 (FIPS posture), 063 (persistent sync), 066 (hierarchy).
- House rules: `peat/CLAUDE.md` (FIPS-only crypto; no consumer-specific references),
  per-repo `SKILL.md` verification checklists.
- Curriculum: `learning/index.html`, `learning/peat-constrained-networking.html`,
  and `learning/0*.md` modules.
- Reference docs & gbrain: `peat-hierarchy-vocabulary.md`, `peat-open-issues-snapshot.md`,
  `peat-addressing-transport-sync.md`, gbrain `research/quic-iridium-sbd-feasibility`.
