# PEAT Curriculum — Diagram Registry

Every diagram in the curriculum, treated as a **code-derived claim**. A diagram encodes the
most drift-prone facts (layer model, transport set, role/enum lists, hierarchy vocabulary per
ADR-066, version stamps, ports) — and that drift is invisible to a prose diff because the facts
live inside SVG coordinates and ASCII box-art. This registry is the mechanism that makes diagram
drift detectable the way `claim-ledger.md` makes claim drift detectable.

**How the refresh uses this file** (see `PROMPT-peat-curriculum-refresh.md` §2/§4/§8): when a
code repo moves, look up every diagram whose **Provenance** falls in the changed area, pull it
into the affected set, **re-derive** it (don't just preserve its legend), keep its node **status
colors** (shipped/in-flight/proposed/speculative) correct, verify hub-SVG ↔ module-ASCII
agreement for the same concept, then advance its **Last verified** entry.

**Conventions.** Markdown structural/flow diagrams use ```mermaid (diffable, GitHub-rendered);
plain-fence ASCII art is for tiny inline sketches only. HTML diagrams are hand-authored inline
SVG and must stay self-contained (no external renderer/CDN). The same concept may appear twice
(hub SVG + module ASCII) — those copies must agree; the **Twin** column notes the pairing.

**Last verified** baseline: `2026-06-18` full sweep, at the per-repo commits recorded in
`REVIEW-STATE.json` → `audited_commits`. Update per-row as diagrams are re-derived.

> **Provenance status — read this.** The **Provenance** column records the `path:line`/ADR each
> diagram *cites in the curriculum*; it was transcribed from the docs, **not independently
> re-verified against the PEAT source in the session that created this registry** (the six code
> repos were not checked out then). The `2026-06-18` date is when a full sweep — running with the
> code present — last verified these facts, per `REVIEW-STATE.json`. Re-verification against
> source happens on the next refresh run that has the sibling repos cloned. Treat the
> provenance as a pointer to check, not a proof already performed.

---

## Markdown modules

| ID | Location | Concept | Type | Provenance | Status shown | Twin | Last verified |
|---|---|---|---|---|---|---|---|
| M-001 | `00b-the-big-idea.md:82` | The missing coordination layer in the stack | ascii | illustrative (whitepaper argument) | n/a | — | 2026-06-18 |
| M-002 | `00b-the-big-idea.md:236` | Three authority axes: RBAC Role / CellRole / AuthorityLevel | ascii | `peat-protocol/src/models/role.rs` | — | — | 2026-06-18 |
| M-003 | `01-architecture-overview.md:106` | Lens A — crate/packaging 5-layer model | ascii | `ARCHITECTURE.md` (verified vs code) | — | H-002 | 2026-06-18 |
| M-004 | `01-architecture-overview.md:174` | Lens B — local change → peer state | mermaid | `DEVELOPER_GUIDE.md §3.2` | — | H-001 | 2026-06-18 |
| M-005 | `01-architecture-overview.md:181` | Lens B legend | ascii | teaching | — | — | 2026-06-18 |
| M-006 | `01-architecture-overview.md:297` | Cargo dependency graph (facade points down) | mermaid | `Cargo.toml` (verified) | optional edges | H-003 | 2026-06-18 |
| M-007 | `02-peat-protocol.md:108` | Three phases as a flow | mermaid | `peat-protocol/src/lib.rs`; `hierarchy/maintenance.rs:227,252,312` | — | — | 2026-06-18 |
| M-008 | `02-peat-protocol.md:267` | Routing rule `is_route_valid` | ascii | `router.rs:101+` | — | — | 2026-06-18 |
| M-009 | `02-peat-protocol.md:496` | HierarchyLevel enum tiers | ascii | `peat-mesh/src/beacon/types.rs:56-67`; ADR-066 | — | — | 2026-06-18 |
| M-010 | `02-peat-protocol.md:532` | Phases ↔ src/ module layout | ascii | `peat-protocol/src/` | — | — | 2026-06-18 |
| M-011 | `02b-formation-and-leadership.md:72` | Hierarchy enum + sizing table | ascii | `beacon/types.rs:50-54` doc comments | — | M-009 | 2026-06-18 |
| M-012 | `02b-formation-and-leadership.md:97` | Formation lifecycle state machine | mermaid | `coordinator.rs` | Ready/AwaitingApproval/Failed | — | 2026-06-18 |
| M-013 | `02b-formation-and-leadership.md:145` | Formation handshake (ALPN `peat/formation-auth/1`) | ascii | `FORMATION_AND_LEADERSHIP.md` | — | — | 2026-06-18 |
| M-014 | `02b-formation-and-leadership.md:170` | Handshake HMAC challenge-response sequence | mermaid | `formation_handshake.rs` | — | H-009 | 2026-06-18 |
| M-015 | `02b-formation-and-leadership.md:238` | Leader election state machine (2s hb, ~6s timeout) | ascii | `leader_election.rs:192-242` | — | M-016 | 2026-06-18 |
| M-016 | `02b-formation-and-leadership.md:265` | Election state diagram | mermaid | `leader_election.rs:238-240`; ADR-068 | — | M-015 | 2026-06-18 |
| M-017 | `03-peat-mesh.md:110` | Discovery → connection | ascii | `discovery/*`; `peer_connector.rs` | — | M-018 | 2026-06-18 |
| M-018 | `03-peat-mesh.md:129` | Discovery flowchart (mDNS/K8s/static) | mermaid | `discovery/*`; `peer_connector.rs` | — | M-017 | 2026-06-18 |
| M-019 | `03-peat-mesh.md:187` | Sync message type wire bytes | ascii | `automerge_sync.rs:92-110`; ADR-034/040 | — | — | 2026-06-18 |
| M-020 | `03-peat-mesh.md:206` | CRDT/negentropy sync sequence | mermaid | `negentropy_sync.rs`; ADR-040 (#435) | — | — | 2026-06-18 |
| M-021 | `03-peat-mesh.md:290` | Transport trait + ConnectionState | ascii | `peat-mesh/src/transport/mod.rs` | — | — | 2026-06-18 |
| M-022 | `04-peat-btle-and-lite.md:85` | BleAdapter trait + platform matrix | ascii | `peat-btle/src/platform/mod.rs` | iOS Beta | — | 2026-06-18 |
| M-023 | `04-peat-btle-and-lite.md:125` | GATT sync sequence | mermaid | `sync/protocol.rs`; `gatt/protocol.rs:68-77` | — | — | 2026-06-18 |
| M-024 | `04-peat-btle-and-lite.md:135` | GATT Write/Indicate legend | ascii | `peat-btle/docs/sync` | — | — | 2026-06-18 |
| M-025 | `04-peat-btle-and-lite.md:348` | Edge dependency flow (acyclic) | ascii | `Cargo.toml:47,174`; ADR-059 Amend.4 | — | — | 2026-06-18 |
| M-026 | `04-peat-btle-and-lite.md:353` | Dependency-direction legend | ascii | Module 1 §1.6 | — | — | 2026-06-18 |
| M-027 | `05-peat-gateway.md:211` | CDC sinks (NATS/Webhook/Kafka) | mermaid | `engine.rs:78-80`; `models.rs:80-84` | Kafka In-flight | H-010 | 2026-06-18 |
| M-028 | `06-data-flows.md:28` | Trace A: sensor → command post | ascii | Module 6 §6.1 | leg-by-leg | M-029 | 2026-06-18 |
| M-029 | `06-data-flows.md:62` | Trace A sequence | mermaid | `peat-lite/protocol/`; `transport/lite.rs`; `cot/` | — | M-028 | 2026-06-18 |
| M-030 | `06-data-flows.md:143` | Trace B: discovery → cell (score weights) | ascii | `leader_election.rs:101-106`; `coordinator.rs` | — | — | 2026-06-18 |
| M-031 | `06-data-flows.md:201` | Trace C: up/down/lateral hierarchy flow | ascii | `hierarchy/`; `command/` | — | — | 2026-06-18 |
| M-032 | `06-data-flows.md:272` | System architecture (gateway off the data path) | ascii | Module 6 §6.4 | peat-sbd/peat-lora Proposed | H-002 | 2026-06-18 |
| M-033 | `08-running-and-operating.md:34` | Quickstart 3-node topology | ascii | Module 3 §3.4 | — | — | 2026-06-18 |

## HTML — `index.html` (hub; mirrors the modules)

| ID | Location | Concept | Type | Provenance | Twin | Last verified |
|---|---|---|---|---|---|---|
| H-001 | `index.html:385` | Lens B in motion — local change → peer state | svg | mirrors M-004 | M-004 | 2026-06-18 |
| H-002 | `index.html:437` | Repo constellation / layer model (incl. peat-node) | svg | mirrors M-003/M-032 | M-003 | 2026-06-18 |
| H-003 | `index.html:548` | Dependency graph (facade points down) | svg | mirrors M-006 | M-006 | 2026-06-18 |
| H-004 | `index.html:670` | peat-protocol surface / phases | svg | mirrors Module 2 | — | 2026-06-18 |
| H-005 | `index.html:707` | (Module 2/2b deep-dive figure) | svg | mirrors Module 2b | — | 2026-06-18 |
| H-006 | `index.html:827` | peat-mesh sync / discovery | svg | mirrors Module 3 | — | 2026-06-18 |
| H-007 | `index.html:854` | (Module 3 figure) | svg | mirrors Module 3 | — | 2026-06-18 |
| H-008 | `index.html:936` | BLE / lite edge | svg | mirrors Module 4 | — | 2026-06-18 |
| H-009 | `index.html:1054` | Gateway / formation security | svg | mirrors Module 2b/5 | M-014 | 2026-06-18 |
| H-010 | `index.html:1113` | Gateway CDC / control plane | svg | mirrors M-027 | M-027 | 2026-06-18 |
| H-011 | `index.html:1393` | Repo map / what to clone next | svg | mirrors Module 7 | — | 2026-06-18 |

## HTML — `peat-constrained-networking.html` ("Off the Grid" track)

| ID | Location | Concept | Type | Provenance | Status shown | Last verified |
|---|---|---|---|---|---|---|
| C-001 | `peat-constrained-networking.html:184` | Link-tier fallback through transports (PACE) | svg | ADR-032 pluggable transport | — | 2026-06-18 |
| C-002 | `peat-constrained-networking.html:230` | Name fixed while address swaps | svg | addressing/transport/sync note | — | 2026-06-18 |
| C-003 | `peat-constrained-networking.html:261` | Three-layer stack: addressing/transport/sync | svg | `peat-addressing-transport-sync.md` | — | 2026-06-18 |
| C-004 | `peat-constrained-networking.html:295` | Out-of-order postcards converge (CRDT) | svg | Automerge convergence | — | 2026-06-18 |
| C-005 | `peat-constrained-networking.html:341` | PACE ladder primary→emergency | svg | PACE model | — | 2026-06-18 |
| C-006 | `peat-constrained-networking.html:374` | Edge lite dialect → bridge → Automerge | svg | `peat-mesh` lite-bridge; peat-lite | Shipped | 2026-06-18 |
| C-007 | `peat-constrained-networking.html:410` | One satellite pass: data up, missing down | svg | ADR-051 (SBD) | Proposed | 2026-06-18 |
| C-008 | `peat-constrained-networking.html:452` | Yank-the-hub: broker dark, PEAT re-homes | svg | leader election / failover | Shipped | 2026-06-18 |
| C-009 | `peat-constrained-networking.html:490` | Reachability vs actual sync toggle | svg | discovery vs sync distinction | — | 2026-06-18 |
| C-010 | `peat-constrained-networking.html:560` | System layout with selected role ringed | svg | CellRole / hierarchy | — | 2026-06-18 |
| C-011 | `peat-constrained-networking.html:620` | Dense local coord, summary crosses cost cliff | svg | hierarchical aggregation | — | 2026-06-18 |
| C-012 | `peat-constrained-networking.html:694` | Roadmap: built / in flight / to design | svg | ADR-051/052; epics | shipped/in-flight/proposed | 2026-06-18 |

---

## Maintenance notes

- **Status colors are mandatory on diagram nodes**, not just legends — a Proposed transport
  (satellite/LoRa) or In-flight sink (Kafka CDC) must be visually distinct from Shipped. The
  constrained track already does this with `st-spec`/pill classes; carry the same convention
  into the hub SVGs as they are touched.
- **Twin pairs must agree.** When a concept's facts change, update both the markdown ASCII and
  the hub SVG (M-003↔H-002, M-004↔H-001, M-006↔H-003, M-014↔H-009, M-027↔H-010,
  M-028↔M-029, M-015↔M-016, M-017↔M-018).
- **Highest-value diagrams to add** (currently prose-only): a clearer
  gateway-observes-from-the-side picture (gateway is NOT in the data path), the
  sensor→command-post end-to-end trace labeled shipped-vs-proposed leg by leg, and the
  negentropy reconcile round-trip. Add to this registry when authored.
</content>
