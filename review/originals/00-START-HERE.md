# PEAT — Intern Onboarding & Learning Path

Welcome. This folder is a self-contained learning track for the **PEAT** open-source
mesh protocol. It is written for a software-developer new-grad / intern who has never
seen the codebase before. By the end you should be able to read any of the five repos,
understand how they fit together, and trace a piece of data from a sensor all the way to
a command post.

> **What is PEAT in one sentence?**
> A decentralized mesh protocol that connects heterogeneous systems — phones, servers,
> sensors, embedded devices, AI models — into one coordinated whole, over any transport,
> even when the network is degraded or denied. The team's mental model is *"TCP/IP for
> autonomy"*: it provides reliable synchronization and coordination primitives the way
> TCP/IP provides reliable communication primitives.

PEAT is built by [Defense Unicorns](https://github.com/defenseunicorns) and is Apache-2.0
licensed.

---

## How to use this track

Read the modules **in order**. Each one builds on the last. Every module has:

- **Concept** — the "why" and the mental model.
- **Code walkthrough** — real files, types, and line references you should open in the repo.
- **Try it** — a small, concrete thing to look at or run to make it stick.
- **Checkpoint** — questions you should be able to answer before moving on.

There is also an interactive HTML version of all this — open
[`peat-learning-hub.html`](peat-learning-hub.html) in a browser for a navigable version with
diagrams.

---

## The ordered modules

| # | Module | File | What you'll learn |
|---|--------|------|-------------------|
| 0 | Start Here | `00-START-HERE.md` (this file) | The lay of the land and how to study it |
| 1·5 | The Big Idea (Why PEAT exists) | [`00b-the-big-idea.md`](00b-the-big-idea.md) | The whitepaper's argument: the scaling crisis, the missing coordination layer, the hierarchy insight, open architecture |
| 1 | Architecture Overview | [`01-architecture-overview.md`](01-architecture-overview.md) | The layer models (two lenses), the five repos, and how they depend on each other |
| 2 | The SDK Facade — `peat-protocol` | [`02-peat-protocol.md`](02-peat-protocol.md) | The one crate you depend on; cells, hierarchy, security, QoS, the three phases |
| 2·5 | Forming a Cell & Electing Leaders | [`02b-formation-and-leadership.md`](02b-formation-and-leadership.md) | Code-level deep dive: formation auth handshake, deterministic election, roles, failover |
| 3 | The Network Layer — `peat-mesh` | [`03-peat-mesh.md`](03-peat-mesh.md) | P2P transport (Iroh/QUIC), Automerge CRDT sync, discovery, the node binary |
| 4 | The Edge — `peat-btle` & `peat-lite` | [`04-peat-btle-and-lite.md`](04-peat-btle-and-lite.md) | BLE mesh across phones/MCUs, and `no_std` CRDT primitives for 256 KB devices |
| 5 | The Control Plane — `peat-gateway` | [`05-peat-gateway.md`](05-peat-gateway.md) | Multi-tenant enrollment, CDC, identity federation, envelope encryption |
| 6 | Cross-Cutting Data Flows | [`06-data-flows.md`](06-data-flows.md) | End-to-end traces: a track update, cell formation, hierarchical aggregation |
| 7 | Repo Map, Gaps & Links to Add | [`07-repo-links-and-gaps.md`](07-repo-links-and-gaps.md) | What's local, what's referenced but missing, and what to download next |
| 8 | Running, Deploying & Operating | [`08-running-and-operating.md`](08-running-and-operating.md) | Hands-on quickstart (run a 3-node mesh) + the operator guide: config, deploy, monitor, troubleshoot |
| 9 | The Protocol Specifications | [`09-protocol-specs.md`](09-protocol-specs.md) | The 5 normative IETF-style specs: transport, sync, schema, coordination, security |

### Suggested grouping

- **Orient (Modules 0, 1·5, 1):** Get the big picture. Start Here, then **The Big Idea** (the *why* —
  read the whitepaper distillation before any code), then the Architecture Overview (the *shape*).
- **Core (Modules 2, 2·5, 3):** This is the heart of PEAT. Spend the most time here. `peat-protocol`
  is what an app developer touches; Module 2·5 deep-dives cell formation & leader election; `peat-mesh`
  is what actually moves the bytes.
- **Reach (Modules 4–5):** The extremes of the spectrum — tiny embedded devices on one end,
  enterprise control plane on the other.
- **Synthesize (Modules 6–7):** Tie it together with real data flows, then map the wider repo
  universe so you know where to go next.
- **Apply & Reference (Modules 8–9):** Get hands-on (build & run a real mesh; learn to deploy and
  operate it), then the normative protocol specs as your precise reference.

> **Want to see it work right now?** Skip ahead to [Module 8 §8.1](08-running-and-operating.md) — a
> 3-node mesh on your laptop in ~10 minutes — then come back. Seeing `sync:` lines converge makes the
> rest concrete.

---

## A note on accuracy & sources

This material was built by reading the **actual source** in this folder (not just the READMEs),
cross-checked against `Cargo.toml` dependency declarations and the architecture docs. Where the
codebase and an older doc disagree, this track follows the **code**, and calls out the
discrepancy explicitly (see Module 1 on the dependency-direction change).

Two facts to keep in your head from the start because they trip people up:

1. **You depend on one crate: `peat-protocol`.** It re-exports `peat-mesh` and `peat-schema`,
   so the whole stack comes along. You do **not** normally add `peat-mesh` to your `Cargo.toml`
   directly.
2. **The dependency arrow points "down" from the facade.** `peat-protocol` *depends on*
   `peat-mesh` (the README's current model). An older `ARCHITECTURE.md` (dated 2025-01-07) drew
   it the other way around. The code is the source of truth — Module 1 explains the history.

When in doubt, open the file. Every claim here points you at one.

---

Next: [Module 1 — Architecture Overview »](01-architecture-overview.md)
