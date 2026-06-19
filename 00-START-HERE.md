# PEAT — Engineer Onboarding & Learning Path

Welcome. This folder is a self-contained learning track for the **PEAT** open-source
mesh protocol. It is written for an engineer who has never seen the codebase before —
accessible enough to follow without deep Rust or distributed-systems background, but
exact enough to survive reading with the source open in another window. By the end you
should be able to read any of the core repos, understand how they fit together, and
trace a piece of data from a sensor toward a command post.

> **What is PEAT in one sentence?**
> A decentralized mesh protocol that coordinates heterogeneous systems — phones,
> servers, sensors, embedded devices — into one synchronized whole over a **pluggable
> transport layer**, and keeps working when the network is degraded or denied.
>
> The team's mental model is *"TCP/IP for autonomy"*: PEAT aims to provide reliable
> synchronization and coordination primitives the way TCP/IP provides reliable
> communication primitives. Treat that as the project's framing, not a delivered SLA —
> the convergence layer (Automerge + negentropy) ships and is the strongest part of the
> system, but priority/QoS *enforcement* (the "<5 s for Priority-1 traffic" target) is
> still in flight, not validated. We will be precise about that distinction throughout.

PEAT is built by [Defense Unicorns](https://github.com/defenseunicorns) and is
Apache-2.0 licensed.

---

## The one label that runs through everything: shipped vs. proposed

This curriculum makes a hard promise: **for every capability it describes, it tells you
whether the capability actually exists.** A defense engineer evaluating PEAT needs to
know what they can deploy today versus what is a design on paper. We use four labels,
and you will see them on individual claims throughout the modules:

| Label | Meaning |
|---|---|
| **Shipped** | In code, on the audited commit, with tests. You can run it. |
| **In-flight** | Real work underway — an open issue, PR, or epic. Partly there. |
| **Proposed** | An ADR exists in `Proposed` status. No implementation yet. |
| **Speculative** | A design invented for teaching or discussion. Not anywhere — not even an ADR-complete proposal. |

Two consequences worth internalizing before you read another line:

- **Transports.** Three transports ship: **QUIC/Iroh** (the default), **BLE** (peat-btle,
  opt-in `bluetooth` feature), and the **peat-lite embedded UDP bridge** (opt-in
  `lite-bridge`). Satellite (Iridium SBD, ADR-051) and LoRa (ADR-052) are **Proposed
  only — no crate, no module, no code.** The internal `TransportType` enum lists eight
  categories, but that is a *taxonomy*, not an inventory of what is implemented. When you
  read "any transport" anywhere, read "the pluggable transport seam, with three real
  backends today."
- **Crypto.** PEAT's code already migrated to FIPS-approved primitives — **AES-256-GCM**,
  **ECDH P-256**, **Ed25519**, **HKDF-SHA-256**, **HMAC-SHA-256** (peat-mesh rc.12,
  2026-05-18). Several READMEs and a few *Proposed* ADRs still mention ChaCha20-Poly1305
  and X25519; those are **stale docs, not shipped behavior**, and are queued for
  amendment. The algorithms are FIPS-approved, but the implementations are pure-Rust
  RustCrypto crates, *not a CMVP-validated module* — a real FIPS 140 boundary runs through
  the gateway's KMS/Vault HSM path. We will keep that distinction sharp.

---

## How to use this track

Read the modules **in order**. Each one builds on the last. Every module has:

- **Concept** — the "why" and the mental model.
- **Code walkthrough** — real files, types, and line references you should open in the repo.
- **Try it** — a small, concrete thing to look at or run to make it stick.
- **Checkpoint** — questions you should be able to answer before moving on.

There is also an interactive HTML version of all this — open
[`index.html`](index.html) in a browser for a navigable version
with diagrams. It is fully self-contained — every diagram is inline SVG and no external
resources are loaded, so it works offline and on air-gapped machines.

---

## The ordered modules

| # | Module | File | What you'll learn |
|---|--------|------|-------------------|
| 0 | Start Here | `00-START-HERE.md` (this file) | The lay of the land and how to study it |
| 1·5 | The Big Idea (Why PEAT exists) | [`00b-the-big-idea.md`](00b-the-big-idea.md) | The whitepaper's argument: the scaling crisis, the missing coordination layer, the hierarchy insight, open architecture |
| 1 | Architecture Overview | [`01-architecture-overview.md`](01-architecture-overview.md) | The layer models (two lenses), the core repos, and how they depend on each other |
| 2 | The SDK Facade — `peat-protocol` | [`02-peat-protocol.md`](02-peat-protocol.md) | The one crate you depend on; cells, hierarchy, security, QoS, the three phases |
| 2·5 | Forming a Cell & Electing Leaders | [`02b-formation-and-leadership.md`](02b-formation-and-leadership.md) | Code-level deep dive: formation auth handshake, deterministic election, roles, failover |
| 3 | The Network Layer — `peat-mesh` | [`03-peat-mesh.md`](03-peat-mesh.md) | P2P transport (Iroh/QUIC), Automerge CRDT sync, discovery — the library that moves the bytes |
| 4 | The Edge — `peat-btle` & `peat-lite` | [`04-peat-btle-and-lite.md`](04-peat-btle-and-lite.md) | BLE mesh across phones/MCUs, and `no_std` CRDT primitives for very small devices |
| 5 | The Control Plane — `peat-gateway` | [`05-peat-gateway.md`](05-peat-gateway.md) | Multi-tenant enrollment, change-data-capture, identity federation, envelope encryption |
| 6 | Cross-Cutting Data Flows | [`06-data-flows.md`](06-data-flows.md) | End-to-end traces: a track update, cell formation, hierarchical aggregation |
| 7 | Repo Map, Gaps & Links to Add | [`07-repo-links-and-gaps.md`](07-repo-links-and-gaps.md) | What's local, what's referenced but missing, and what to download next |
| 8 | Running, Deploying & Operating | [`08-running-and-operating.md`](08-running-and-operating.md) | Hands-on quickstart + the operator guide: config, deploy, monitor, troubleshoot |
| 9 | The Protocol Specifications | [`09-protocol-specs.md`](09-protocol-specs.md) | The normative IETF-style specs: transport, sync, schema, coordination, security |

### Suggested grouping

- **Orient (Modules 0, 1·5, 1):** Get the big picture. Start Here, then **The Big Idea**
  (the *why* — read the whitepaper distillation before any code), then the Architecture
  Overview (the *shape*).
- **Core (Modules 2, 2·5, 3):** This is the heart of PEAT. Spend the most time here.
  `peat-protocol` is what an app developer touches; Module 2·5 deep-dives cell formation
  and leader election; `peat-mesh` is what actually moves the bytes.
- **Reach (Modules 4–5):** The extremes of the spectrum — tiny embedded devices on one
  end, the enterprise control plane on the other.
- **Synthesize (Modules 6–7):** Tie it together with real data flows, then map the wider
  repo universe so you know where to go next.
- **Apply & Reference (Modules 8–9):** Get hands-on (build and run a real mesh; learn to
  deploy and operate it), then the normative protocol specs as your precise reference.

> **Want to see it work right now?** Skip ahead to
> [Module 8 §8.1](08-running-and-operating.md) for a local multi-node mesh on your
> laptop, then come back. Seeing `sync:` lines converge makes the rest concrete. (The
> exact node count and timing in that quickstart are validated in Module 8, where the
> commands are run; treat any figure here as a forward pointer, not a measured promise.)

---

## The repos you'll be reading

PEAT is not one repository — it is a small constellation. The curriculum and the
ground-truth audit behind it cover **six code repos**. Knowing the boundaries up front
saves confusion later, because some pieces people expect to be together actually live
apart.

| Repo | What it is | Role |
|---|---|---|
| **`peat`** (umbrella) | A Cargo workspace. The crates that matter live inside it: **`peat-protocol`** (the SDK facade — formation, hierarchy, QoS, CoT, discovery, distribution), `peat-schema` (Protobuf), `peat-transport` (HTTP/REST + TAK/CoT bridge), `peat-persistence`, `peat-ffi` (mobile bindings). The bare `peat` crate itself is a reserved-name placeholder with no dependencies. | The application-developer surface |
| **`peat-mesh`** | The P2P networking library: Iroh/QUIC transport, Automerge CRDT sync, negentropy reconciliation, discovery, the pluggable-transport seam. It is a *library* exposing `PeatMesh`/`Node` types — **not** a deployable binary. | The network layer |
| **`peat-btle`** | Cross-platform BLE mesh (Linux/BlueZ, macOS, iOS *(Beta)*, Android AAR, ESP32/NimBLE). Hand-rolled `no_std`-friendly CRDTs. | The Bluetooth edge |
| **`peat-lite`** | A `no_std` codec + CRDT library for microcontrollers, targeting devices with as little as ~256 KB RAM (a *design budget* from ADR-035 — there is no allocator cap or static-RAM assertion enforcing it). | The microcontroller edge |
| **`peat-gateway`** | The control plane: tenant enrollment, change-data-capture, OIDC identity federation, envelope encryption. **It is not a mesh node and not in the data path** — it observes and manages, it does not sync CRDTs or relay traffic. | The control plane |
| **`peat-node`** | A separate, actively-developed gRPC/Connect sidecar binary that embeds `peat-mesh` + `peat-protocol` and exposes them on one port. **This is the deployable node** people mean when they say "run a node." It speaks Iroh QUIC only (no BLE). | The deployable node |

A couple of these boundaries trip up almost everyone:

- The thing you *deploy and run* is **`peat-node`**, the sidecar — not `peat-mesh` (a
  library) and not the gateway (control plane). Module 3 covers the library; Module 8
  covers running the node.
- **`peat-gateway` is a control plane, not a relay.** It watches the mesh through
  read-only handles that mesh nodes register into it; it never carries your documents.
  When a use case says "data reaches the command post," the relaying is done by a
  peat-mesh *node*, with the gateway observing from the side.

---

## Five concrete things you can build with PEAT today

Abstract protocol descriptions are hard to hold onto. Here are five end-to-end use
cases, each labeled by what actually ships. Module 6 traces them in code; Module 7 maps
the gaps. The pattern to notice: **the synchronization backbone is solid and shipped; the
exotic transports and the autonomy-tasking primitive are not yet built.**

1. **TAK/CoT operator picture** *(mostly Shipped).* Dismounted teams on phones (BLE),
   a vehicle running a peat-node sidecar, and a Cursor-on-Target client at the command
   post. Positions sync intra-cell over BLE, bridge to the mesh over QUIC, and get
   encoded to CoT XML by `peat-transport`'s TAK bridge (ADR-020/028/029, **Shipped**).
   The whole path ships — with the caveat that QoS *preemption* and the "<5 s"
   Priority-1 latency are targets, not enforced guarantees yet.

2. **A robot joining a cell and being tasked** *(join Shipped; tasking In-flight/Proposed).*
   A UGV running peat-node discovers a cell (mDNS / static peering / Kubernetes
   EndpointSlice — all **Shipped**), proves a pre-shared formation key via an
   **HMAC-SHA-256 challenge-response** (the key never crosses the wire), and is assigned
   a role by deterministic capability scoring. The *tasking* half is younger: a command
   today is an ordinary JSON document in a `commands` collection — there is **no
   `command_log` CRDT** (that is **Speculative**), and *targeted* delivery to a specific
   role is **ADR-046 (Proposed), epic #853 (In-flight)**. Throughout, **autonomy operates
   under human authority**: a mission-critical capability requires human approval at
   formation.

3. **Maritime / beyond-line-of-sight relay** *(relay legs Shipped; satellite Proposed).*
   A vessel relays a coastal sensor cell's summary to shore over QUIC, falling back to
   Iridium SBD when line-of-sight drops. The in-range QUIC legs are **Shipped and real**.
   The satellite fallback is **Proposed (ADR-051) — no crate** — and the single-burst
   digest scheme it would need is **Speculative** (the shipped reconciler, negentropy,
   assumes an interactive QUIC channel and would not work over a one-shot ~1,960-byte
   satellite frame as-is).

4. **Disaster-response disconnected cell** *(Shipped — the strongest story).* A team in
   a comms-denied building forms a partitioned cell, elects a leader locally (deterministic
   scoring, **no quorum, so no split-brain stall**), accumulates state offline, then
   reconnects and converges automatically. Negentropy reconciles the document sets and
   transfers only the genuinely-missing deltas; Automerge merges deterministically. This
   offline-first convergence is the backbone of PEAT and is **fully Shipped** — with one
   caveat: BLE-path reconnect re-delivery of pending state is **In-flight (peat-btle#73)**;
   the QUIC/peat-node path is the robust one.

5. **Remote sensor field over a constrained link** *(embedded legs Shipped; long-range
   Proposed).* Battery ESP32 sensors run peat-lite firmware, report over UDP to a bridge
   node (peat-mesh `lite-bridge`), which aggregates them into the full Automerge mesh over
   QUIC. A sensor reading fits in one datagram (16-byte header + payload, ≤ 512 B total).
   OTA firmware push to the sensors is **Shipped** (ADR-047). A LoRa long-range variant of
   the first leg is **Proposed (ADR-052)**, not built.

The "sensor all the way to a command post" trace you'll learn to follow is real and
teachable — just remember which legs ship (the QUIC/BLE/UDP core) and which are design
(satellite/LoRa), and that the command-post leg goes through a relaying *mesh node*, not
the control-plane gateway.

---

## A note on accuracy & sources

This material was built by reading the **source across the PEAT repos** — not just the
READMEs — and cross-checking it against `Cargo.toml` dependency declarations, the ADRs in
`peat/docs/adr/`, and the open-issue tracker. Where a crate's internals live in a sibling
repo rather than the umbrella tree, that is noted. **The operating rule is code over
docs:** when the code, an ADR, a doc, or a README disagree, the code wins, and the
discrepancy is called out rather than smoothed over. This matters here specifically,
because PEAT's READMEs and several specs lag the code — on crypto (they still say
ChaCha20), on role names, on version numbers, and on wire details. Quoting a stale doc is
exactly how an error survives, so the curriculum quotes the code.

Two facts to keep in your head from the start, because they trip people up:

1. **You depend on one crate: `peat-protocol`.** It re-exports `peat-mesh` and
   `peat-schema` (`peat-protocol/src/lib.rs:105-106`), so the whole stack comes along.
   An *application developer* does **not** normally add `peat-mesh` to their `Cargo.toml`
   directly. (Embedders like `peat-node` and `peat-gateway` do depend on `peat-mesh`
   directly — but that is the integrator's path, not the app developer's.)
2. **The dependency arrow points "down" from the facade.** `peat-protocol` *depends on*
   `peat-mesh` — its `security/*` and Iroh-transport modules are thin re-export shims over
   `peat_mesh` (`peat.md` ground truth §0). An older `ARCHITECTURE.md` drew the arrow the
   other way; the code is the source of truth, and Module 1 walks through that history.

When in doubt, open the file. Every claim here points you at one.

---

Next: [Module 1 — Architecture Overview »](01-architecture-overview.md)
