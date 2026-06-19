# Prompt — Incremental refresh of the PEAT learning curriculum (scheduled)

> The **cheap, recurring** companion to `PROMPT-peat-training-doc-improvement.md`. That prompt
> is the full deep review (run it on first setup and on a periodic full sweep). This one is
> **delta-driven**: it only re-examines what git history changed since the last audited commit,
> and only rewrites the documents those changes affect. Designed to run unattended on a
> schedule. It inherits every rule (honesty, code-over-everything, shipped/in-flight/proposed/
> speculative labeling, FIPS-only crypto, no vendor names, the §1b Autonomy Contract) from
> `PROMPT-peat-training-doc-improvement.md` — read that file's §1, §1b, §8, §11 first; this
> prompt only specifies the *incremental* behavior.

---

## 0 · Mission

Keep the PEAT learning curriculum in `learning/` synchronized with the code, cheaply, by
auditing only what changed since the last run. Never re-run the full review unless explicitly
asked (`full` mode) or the periodic full-sweep is due.

Inputs: `learning/REVIEW-STATE.json` (records `audited_commits` per repo and `last_full_run`),
the six repos, and the curriculum docs. Source of truth is always the code.

## 1 · Phase 0 — Read state & detect drift

1. Read `learning/REVIEW-STATE.json`. Extract `audited_commits[repo].head` for each of
   `peat, peat-mesh, peat-btle, peat-lite, peat-gateway, peat-node`, and `last_full_run`.
2. For each repo: `git -C <repo> fetch` then `git -C <repo> rev-parse origin/HEAD` (or the
   tracked branch). Compute the delta: `git -C <repo> log --oneline <audited>..origin/HEAD`
   and `git -C <repo> diff --name-only <audited>..origin/HEAD`. **Do not pull/modify the tree**
   unless your run mode is the local in-place mode (then a fast-forward of the working copy is
   allowed; never force).
3. **If no repo moved** → write a "no drift" entry to the run log in `REVIEW-STATE.json`
   (timestamp from `args.now`, see §6), touch nothing else, and exit. This is the common case
   and should cost almost nothing.
4. **Full-sweep check**: if `args.full` is set, or `last_full_run` is older than the
   full-sweep interval (default 30 days, from `args.fullSweepDays`), STOP this incremental
   path and instead run the full `PROMPT-peat-training-doc-improvement.md` end to end, then set
   `last_full_run = args.now`.

## 2 · Phase 1 — Scope the affected documents

Map changed paths to curriculum docs (use judgment; this is a default routing, not a cage):

| Changed in… | Re-audit these docs |
|---|---|
| `peat/peat-protocol/**`, `peat/peat-schema/**` | `02-peat-protocol.md`, `02b-formation-and-leadership.md`, `06-data-flows.md`, `09-protocol-specs.md` |
| `peat/docs/adr/**` | every doc + `peat-addressing-transport-sync.md` that cites the changed ADR (grep for the ADR number) |
| `peat-mesh/**` | `03-peat-mesh.md`, `peat-constrained-networking.html` |
| `peat-btle/**` | `04-peat-btle-and-lite.md`, `peat-constrained-networking.html` |
| `peat-lite/**` | `04-peat-btle-and-lite.md`, `peat-constrained-networking.html` |
| `peat-gateway/**` | `05-peat-gateway.md` |
| `peat-node/**` | `06-data-flows.md`, `08-running-and-operating.md`, `09-protocol-specs.md` |
| any `README.md`/`CHANGELOG.md`/`ROADMAP.md`/`Cargo.toml` | `00-START-HERE.md`, `00b-the-big-idea.md`, `01-architecture-overview.md`, `07-repo-links-and-gaps.md` |

Always also reconcile: `learning/index.html` (the hub; it mirrors the modules) and
`07-repo-links-and-gaps.md`. Build the **affected-docs set** from the union.

## 3 · Phase 2 — Targeted ground-truth refresh

For each **changed repo only**: re-establish reality for the parts that moved (read the changed
files + their context, confirm symbols with gbrain `code_def`/`code_refs`, note ADR status
changes). Update that repo's `learning/review/ground-truth/<repo>.md` and the relevant sections
of `learning/review/ground-truth.md`. Record the new audited commit. Leave unchanged repos'
ground-truth files alone.

## 4 · Phase 3 — Targeted audit & rewrite

For each doc in the affected-docs set: re-audit only the claims touching the changed areas
(verdicts per the main prompt §4), then apply fixes **in place**, preserving all house rules
(visible shipped/in-flight/proposed/speculative labels; FIPS-only; no vendor names; legends on
every diagram for HTML; numbering/order for markdown). Do not rewrite untouched sections.
Keep the hub consistent with any module it mirrors.

## 5 · Phase 4 — gbrain refresh (local mode only)

Update the gbrain `default` source to match: refresh `peat-current-state` and
`peat-open-issues-snapshot` via `put_page` (update in place, **never delete**), reflecting the
new commits and any status changes. Skip this phase in cloud/CI mode (no local gbrain).

## 6 · Phase 5 — Persist state & log

Update `learning/REVIEW-STATE.json`:
- `audited_commits[repo].head` → the new HEADs for changed repos.
- append to a `run_log` array: `{ when: args.now, mode, repos_changed, docs_touched,
  claims_changed, notes }`. (Timestamps come from `args.now` — the scheduler passes the date;
  do not invent one.)
- if a full sweep ran, set `last_full_run = args.now`.

Append (do not overwrite) a dated delta section to `learning/review/CHANGELOG-review.md`
summarizing what changed this run and any new unverifiable claims.

**Update the reader-facing freshness surfaces** (these are inline so the pages stay
self-contained — never make them fetch a manifest at runtime):
- In `learning/index.html` AND `learning/peat-constrained-networking.html`, replace the
  contents between `<!--SYNC-->` and `<!--/SYNC-->` in the footer stamp with the new date,
  run type (full sweep / incremental / no drift), and the per-repo short commits.
- In `learning/changelog.html`: update the same `<!--SYNC-->…<!--/SYNC-->` block and the
  per-repo commit `tag`s in the "Current sync" card, and **prepend one `<tr>`** immediately
  after `<!-- CHANGELOG-ROWS-START -->` (Date · Type pill [`full`/`incr`/`man`] · Scope ·
  Summary). Use the same date from `args.now`; never invent a date.

## 7 · Output mode

- **`local` (default):** rewrite docs in place, update gbrain, update state files. Requires the
  local repo clones + gbrain MCP.
- **`ci` / `cloud`:** do all the above except gbrain; do the work on a branch
  (`curriculum-refresh/<date>`), and open a PR with the diff + the run-log summary instead of
  editing the working `main`. (Requires the curriculum to live in a pushed git repo.)

Mode comes from `args.mode` (default `local`).

## 8 · Definition of done (incremental)

- Either a clean "no drift" log entry, or: only affected docs changed, each correctly relabeled
  shipped/in-flight/proposed/speculative, no house-rule violations, hub consistent with modules.
- `REVIEW-STATE.json` advanced to the new commits with a run-log entry.
- `CHANGELOG-review.md` has a new dated delta section; nothing silently overwritten.
- Anything that couldn't be verified is logged, not hidden.
</content>
