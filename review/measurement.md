# Measuring the curriculum's impact

This note records **how we measure whether the PEAT learning curriculum is actually helping**, what
we collect, and the privacy posture. It exists so the team — and any future refresh run — knows the
instrumentation is deliberate, not accidental, and knows where to look for the signal.

## Why we measure

The curriculum's whole purpose is to get new engineers productive on PEAT faster and more accurately.
We can't tell whether it's working from the inside. So the three reader-facing pages carry lightweight
instrumentation whose only job is to answer two questions: **which material gets read**, and **whether
readers found it helpful or wrong**. That tells us where to spend the next refresh — a module nobody
reaches, or one that draws repeated 👎 and corrections, is a module to rewrite.

## What we collect

- **Page and per-module views** — via a cookieless [GoatCounter](https://www.goatcounter.com) beacon.
  Because `index.html` and `peat-constrained-networking.html` are single-page apps, the beacon also
  counts each section as it's opened (paths like `/index.html#m3`), so we see *which modules* land,
  not just that the hub was hit. `changelog.html` counts as a single page.
- **Helpfulness signal** — a "Was this helpful?" 👍 / 👎 control in every page footer. Each click
  fires a GoatCounter **event** named `/feedback/<section>/up` or `/feedback/<section>/down`, giving a
  per-module helpful/not-helpful tally.
- **Corrections and suggestions** — the footer's "Report an error / suggest an edit" link (and the 👎
  button) opens a **prefilled GitHub issue** on `parleylabs/peat-learning`, auto-filled with the page,
  the section, the URL, and a "what's wrong / what would make it better" template, labeled
  **`curriculum-feedback`**. Reader corrections therefore land as trackable issues in the same repo
  the refresh routine already works from, and can feed the next pass directly.

## Privacy posture

- **Cookieless, no PII.** GoatCounter sets no cookies and stores no personal data, so the pages need
  **no consent banner**. We collect page paths, referrers, and the 👍/👎 events — not identities.
- **Air-gap safe.** The beacon is a single async request that **fails silently with no network**, so a
  downloaded or air-gapped copy of any page still renders and works exactly as before; it simply
  doesn't report. This is why the analytics snippet is an *approved exception* to the otherwise-strict
  "fully self-contained, no external script" rule (see the carve-out in `CLAUDE.md` and both refresh
  prompts). No *other* external resource is permitted.

## Where to look

- **Dashboard:** `https://parley-peat.goatcounter.com` (views per page/module, referrers, and the
  `/feedback/...` events). Site code `parley-peat`.
- **Reader corrections:** GitHub issues on `parleylabs/peat-learning` filtered by the
  `curriculum-feedback` label.

## How to read the signal (and its limits)

- Treat view counts as a **relative** signal — module A vs B, week over week — not an exact headcount.
  Some privacy extensions block even cookieless analytics, so the absolute numbers undercount.
- The **👍/👎 ratio and the corrections issues are the higher-fidelity read** of "is this helpful and
  correct." A module with traffic but a poor thumbs ratio, or a recurring correction, is the priority
  for the next refresh.
- Acting on it: each refresh run (or a human review) should glance at the dashboard's worst-performing
  modules and the open `curriculum-feedback` issues, and route them into the rewrite the same way a
  code-drift finding is routed (see `PROMPT-peat-curriculum-refresh.md`).

## Governance for refresh runs

The beacon + feedback widget (`<div id="fb">`, the GoatCounter `<script>`, and the SPA
`active()`/`gcView()` wiring) must be **preserved verbatim** on every rewrite — it is whitelisted in
`CLAUDE.md` and both prompts. Do not strip it as a self-containment violation, and **never reset a real
`data-goatcounter` site code back to the `YOURCODE` placeholder**. The instrumentation can be swapped
to a self-hosted analytics backend (e.g. Umami) later by changing only the one beacon line.

_Instrumentation added 2026-06-23._
