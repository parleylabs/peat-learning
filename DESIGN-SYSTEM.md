# Peat Learning — visual design standard

The canonical visual standard for the curriculum. It is the adapted *Visual Explainer* skill, **bounded
by our two hard constraints**, and it is enforced by the visual gate on every refresh (see the prompts'
gate list) and applied incrementally by the monthly upgrade pass. A live reference rendering lives in
`DESIGN-SYSTEM.html`.

> **Principle: adopt the design system, not the CDN.** We take the quality bar — a real component
> system, accessible semantics, light+dark, "no generic aesthetic" — and reject the delivery mechanism
> (runtime Mermaid/Chart.js from a CDN). Everything is **pre-rendered and inline** so the pages work
> offline and air-gapped.

## 1 · The two hard constraints (these override the skill)

1. **Self-contained & air-gap-safe.** No `<script src>`, no CDN, no external fonts, no runtime fetch.
   The *Off the Grid* track is about denied networks and any page may be opened air-gapped.
   - Diagrams are authored as ` ```mermaid ` in the markdown (GitHub renders them) **and pre-rendered to
     inline SVG** for the HTML. Charts are pre-rendered inline SVG / CSS — never a runtime chart lib.
   - All CSS/JS is inline. Brand fonts (Montserrat/Roboto) are **self-hosted woff2**, never a font CDN.
   - **The only permitted external call is the cookieless GoatCounter beacon** (the existing, signed-off
     carve-out). A CDN `<script src>` is a **gate failure**.
2. **Parley Labs brand, always.** Keep our tokens and the **status palette** below — never the skill's
   default Instrument-Serif / generic / violet aesthetic.

## 2 · Theming & tokens (light + dark)

Define everything via CSS custom properties; ship **both** schemes via `@media (prefers-color-scheme)`.
Pages are dark-first today; light mode is added incrementally (monthly pass).

Core tokens: `--bg --surface --surface2 --border --text --dim --green(#1cb57d) --orange(#e84700)`.
**Status palette (canonical, do not invent colors):**

| State | Color | Use |
|---|---|---|
| **Shipped** | green `#3fb950` | in code, tested |
| **In-flight** | amber `#d29922` | open issue/PR/epic |
| **Proposed** | blue `#58a6ff` | ADR Proposed, no code |
| **Speculative** | purple `#a371f7` | teaching-only |
| *error / rejected* | red `#f85149` | **not a status** — a separate signal |

Apply status as **badges/dots, never emoji**. Use semi-transparent fills (`color-mix` / 8-digit hex) +
`currentColor` text so a component survives both themes.

## 3 · Components (pick the right one)

- **`.card`** — entities/components; variants `hero`, `accent` (left border), recessed. Text-heavy
  architecture or plans → CSS-grid card sets, not walls of prose.
- **Data-table pattern** — **any table with 4+ rows or 3+ columns**: wrap in `.tablewrap`
  (`overflow-x:auto`), sticky `thead`, constrained columns, `min-width:0` on flex/grid children,
  `overflow-wrap:break-word`. No horizontal page scroll, ever.
- **KPI cards** — single metrics (e.g. `quality_trend`: unverifiable-claim count, open feedback).
- **Callout** — one accented box for the single most important caveat per section.
- **Status badge** — the four states above.

## 4 · Diagrams (the load-bearing rules — extends `CLAUDE.md` § Diagrams)

Diagrams are **code-derived claims, not decoration**, registered in `review/diagrams.md`, re-derived
from `path:line`/ADR each time the code moves, and carry the status palette + a legend. That discipline
is unchanged. The visual standard **adds**:

- **Format.** Markdown → ` ```mermaid ` (diffable, GitHub-rendered). HTML → **pre-rendered inline SVG**
  (generated from the same mermaid source). Plain ASCII art is for *tiny* inline sketches only and is a
  **migration target** — prefer a mermaid twin so both copies are real, diffable diagrams.
- **Authoring hygiene (mermaid source).** `flowchart TD` for anything with 5+ nodes / branching;
  `LR` only for 3–4-node linear flows. **Max ~10–12 nodes** per diagram — beyond that, split into a
  *hybrid* (mermaid overview + a CSS card grid). `<br/>` for multi-line labels (never `\n`); quote
  labels; alphanumeric node IDs.
- **Light mode = dark figure-plate (decision A, 2026-06-26).** A diagram **always sits on a fixed dark
  surface card**, in both light and dark page modes. This keeps every existing inline-SVG's hardcoded
  colors valid — **zero rework of current diagrams** — and reads as a deliberate dark figure. (Full
  theme-adaptive SVGs are optional, opt-in per registry row, never required.)
- **Legend on every diagram**, status palette exact, hub-SVG ↔ module-mermaid **twins must agree**.
- **Interactivity.** Wrap HTML diagrams in a shell with **zoom / pan / reset / expand** — **inline JS
  only**, no CDN. Additive; legacy static SVGs remain valid until migrated.

## 5 · Accessibility & robustness

Semantic HTML (`<table>`, `<nav>`, `<section>`, headings in order); ARIA labels on diagrams/SVG;
respect `prefers-reduced-motion` (gate all motion behind `@media (prefers-reduced-motion: no-preference)`);
no emoji for status; overflow protection (`min-width:0`, `overflow-wrap:break-word`, scroll wide content
inside its own container). The main point must be obvious in the first viewport.

## 6 · The visual gate (enforced every refresh)

A new/changed reader page or diagram passes only if **all** hold:
1. **Self-contained** — no CDN/`<script src>`/external font/runtime fetch (GoatCounter beacon excepted).
2. **Diagram** — pre-rendered inline SVG (HTML) / mermaid source (md); ≤12 nodes or hybrid; status
   palette + legend; dark figure-plate; zoom/pan present (HTML); registry row advanced.
3. **Tables** — 4+ rows use the data-table pattern; no horizontal page overflow.
4. **Theme** — works in light and dark.
5. **Accessibility** — semantic, reduced-motion respected, no emoji status.
6. **Brand** — Parley Labs tokens + the canonical status palette; no generic aesthetic.

## 7 · Rollout (incremental, never big-bang)

Set the standard → **gate every new/changed visual against it now** → the **monthly full sweep migrates
a bounded batch** of legacy visuals (ASCII → SVG, dense prose → cards/tables, add light-mode plates),
tracking compliance in `review/diagrams.md` → the standard **self-sharpens via the ratchet**:
`curriculum-feedback` on confusing visuals + the §9b retrospective filing `prompt-amendment` issues.
Existing pages stay valid throughout; nothing is rewritten all at once.
