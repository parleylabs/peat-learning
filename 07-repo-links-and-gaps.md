# Module 7 — Repo Map, Capability Gaps & Links to Add

**Goal:** two maps in one. First, *what code lives where* — which repos are in this folder, which
are referenced but not checked out, and the full clone list so the team can pull what's missing.
Second — and more important for a skeptical reader — *what does not exist yet*: the capability gaps
behind the documentation, each labeled and tied to a real issue or ADR. A repo map alone is a
download list; the capability map is what tells you whether the system does what the docs claim.

**Status labels used throughout** (the same ones the rest of the curriculum uses):

- **Shipped** — in code, on the audited HEAD, covered by tests.
- **In-flight** — an open issue, PR, or epic is moving it; partially present.
- **Proposed** — an ADR exists in `Proposed` status; no implementation.
- **Speculative** — design discussed for teaching, not even an ADR-complete proposal anywhere.

Everything below was checked against the audited HEADs in
[`learning/review/ground-truth.md`](review/ground-truth.md): `peat` `68e9c3c`, `peat-mesh` rc.43,
`peat-btle` 0.4.0, `peat-lite` 0.2.5, `peat-gateway` 0.1.0, `peat-node` 0.4.7. The operating
principle is **code over everything**: where a README, a spec, or a months-old guide disagrees with
the source on the audited HEAD, the source wins.

---

## 7.1 What's local (in this folder)

These six directories are present and are what the rest of this track covers. The fifth column gives
each one's role in a sentence so you know why you'd open it.

| Local dir | Upstream | What it is |
|-----------|----------|------------|
| `peat/` (umbrella: `peat-protocol`, `peat-schema`, `peat-transport`, `peat-persistence`, `peat-ffi`, `examples/`, `spec/`, `docs/adr/`) | https://github.com/defenseunicorns/peat | The umbrella workspace. The `peat` crate itself is a **reserved-name placeholder with zero dependencies**; the real code lives in the workspace members. A consumer depends on **`peat-protocol`**, which re-exports `peat_mesh` and `peat_schema`. |
| `peat-mesh/` | https://github.com/defenseunicorns/peat-mesh | The networking and CRDT engine: Iroh/QUIC transport, Automerge sync, negentropy reconciliation, the crypto primitives, discovery. `peat-protocol`'s `security/*` and `network/iroh_transport` modules are thin `pub use peat_mesh::…` re-export shims. |
| `peat-btle/` | https://github.com/defenseunicorns/peat-btle | The Bluetooth Low Energy mesh transport: GATT service, MTU chunking, relay, hand-rolled `no_std` CRDTs. Opt-in behind the `bluetooth` feature. |
| `peat-lite/` | https://github.com/defenseunicorns/peat-lite | The embedded/microcontroller tier: a 16-byte-header UDP wire codec and small CRDTs for ESP32-class devices. Bridged into the mesh by peat-mesh's `lite-bridge` feature. |
| `peat-gateway/` | https://github.com/defenseunicorns/peat-gateway | The enterprise **control plane** — binary, library, web UI, and Helm/Zarf/UDS packaging. It is **not a mesh node and not in the data path**; it observes and manages. Server-side, not an SDK crate. |
| `peat-node/` | https://github.com/defenseunicorns/peat-node | The deployable **production node**: a gRPC / Connect / gRPC-Web sidecar that embeds `peat-mesh` + `peat-protocol` and exposes them on one port. Iroh QUIC only — no BLE path. (See §7.2 for how it differs from the in-`peat-mesh` demo binary.) |

> **Note on registries.** Whether `peat-protocol`, `peat-schema`, `peat-btle`, and `peat-lite` are
> *published* to crates.io, and `peat-ffi` to Maven Central, is **not verified in this audit**. The
> crates exist and carry versions, but the peat-mesh README advertises stale versions (0.3.2 against
> a shipped rc.43), so registry-version trust is shaky. Check the actual registry before relying on a
> published version; do not assume the README is current.

---

## 7.2 Referenced but NOT present locally — pull these down

These are PEAT repos referenced in code comments, `Cargo.toml`s, READMEs, or ADRs that are **not**
checked out here. Listed roughly by how useful they'd be to someone onboarding.

| Repo / component | GitHub | Why it's referenced | Status / priority |
|------------------|--------|---------------------|-------------------|
| **peat-sim** | https://github.com/defenseunicorns/peat-sim | The network simulator — multi-node and partition testing without hardware. It is **where the scale-validation work lives**: epics #724/#725/#726/#727 target 900 / 1.2K / 10K nodes. | **In-flight (scale validation).** Existence as the sim/validation vehicle is corroborated by the #724 epic; the standalone repo URL is not independently verified here. High priority. |
| **peat-node** | https://github.com/defenseunicorns/peat-node | Already cloned (see §7.1). The production node sidecar. | **Shipped.** A real, distinct, actively-developed repo — *not* a repackaging of anything. See the box below. |
| **peat-registry** | https://github.com/defenseunicorns/peat-registry | UDS registry replication for disconnected/DDIL environments — ADR-054 (`054-UDS-registry-replication-ddil.md`, on disk). Relevant to gateway and air-gapped deployment. | **Status unverified.** ADR-054 exists; the gateway's mesh wire-tier enum is "explicitly modeled on peat-registry artifact routing," so the concept is real, but the repo and its implementation status are not audited here. Medium priority. |
| **peat-sdk-go** | https://github.com/defenseunicorns/peat-sdk-go | A Go SDK, useful if any consumer service is written in Go. | **Proposed / Planned.** The peat-mesh README lists Go/Python/Kotlin SDKs as **Phase 3 (Future)**. Treat as roadmap, not a clonable shipped repo, until confirmed. Low priority. |
| **peat-android / android demo** | A demo lives **locally** at `peat/examples/android-peat-demo` (confirmed present); a standalone repo is referenced in peat-mesh issue #828 | Android consumer integration and user-acceptance testing. | **In-repo example is present and is the one to start from.** Whether a separate standalone repo exists is unconfirmed. Medium priority for mobile work. |

> **peat-node is a real component — drop the old "is this a repackaging?" hedge.**
>
> An earlier version of this module flagged peat-node as possibly just a redeploy of the
> `peat-mesh-node` binary that lives inside peat-mesh (`peat-mesh/src/bin/peat-mesh-node.rs`,
> confirmed present). That framing understated a major piece of the system. The two are categorically
> different:
>
> - **`peat-mesh-node`** (inside peat-mesh) is a small demo / reference binary for bringing up a mesh
>   by hand.
> - **`peat-node`** (its own repo, audited at v0.4.7) is the **production sidecar**: it embeds
>   peat-mesh + peat-protocol and exposes them as a gRPC / Connect / gRPC-Web API on a single port. It
>   is the Kubernetes sidecar pattern's node, it ships a Helm chart plus Zarf and UDS bundles, and it
>   is the UDS Remote Agent integration target. The proto defines **28 RPCs** and `service.rs`
>   implements **27** (the "25 RPCs" you may see in older docs is outdated). It speaks **Iroh QUIC
>   only** — a `docs/DESIGN.md` diagram labels "BLE" aspirationally, but there is no BLE code path in
>   peat-node.
>
> When the production-deployment story matters, peat-node is the component, not the demo binary.

---

## 7.3 External (non-PEAT) dependencies worth a bookmark

Not repos to clone, but the upstreams whose docs you'll read while working in PEAT:

- **Iroh** (QUIC P2P) — https://github.com/n0-computer/iroh — the heart of the `peat-mesh` transport.
  Pinned with the `tls-aws-lc-rs` provider so the TLS stack uses FIPS-approved AES, not `ring`.
- **Automerge** (CRDT) — the document model; the sync-protocol paper is https://arxiv.org/abs/2012.00472
  (cited in `peat-mesh/src/storage/automerge_sync.rs`).
- **negentropy** (set reconciliation) — the shipped sync optimization. It is the lesson applied from
  **ADR-040, whose on-disk title is `040-nostr-protocol-lessons.md`** (issue #435); "negentropy set
  reconciliation" is what PEAT took *from* that ADR, not the ADR's own title. Its "O(log n) rounds"
  property is an **algorithmic / module-doc claim, not an independently measured benchmark.**
- **redb** — the pure-Rust embedded key-value store used for persistence (peat-mesh's
  `automerge-backend` feature and peat-gateway's default state backend).
- **BLE platform backends behind `peat-btle`:** **bluer** (Linux BlueZ), **objc2-core-bluetooth**
  (Apple), **NimBLE / esp-idf** (ESP32), and a **Windows / WinRT** backend that is **in-flight and
  untested** (code present, not validated per the peat-btle README) — so it is not yet on equal
  footing with the other three.
- **UniFFI** — generates the Kotlin/Swift bindings in `peat-ffi` and the mobile crates.
- **Axum** — the HTTP/WS framework behind peat-mesh's broker, peat-gateway's API, and
  peat-transport's REST bridge.
- **TAK / Cursor-on-Target / MIL-STD-2525** — the tactical interop standards the CoT layer targets.
  CoT is a public protocol name; MIL-STD-2525 is the symbology standard CoT references, not a PEAT
  module.

---

## 7.4 Suggested clone command (for whoever sets up the workspace)

To complete the picture alongside what's already here:

```bash
# from the directory that contains peat/, peat-mesh/, peat-node/, etc.
for r in peat-sim peat-registry peat-sdk-go; do
  git clone https://github.com/defenseunicorns/$r.git
done
# Optional, for deployment/packaging work — the platform peat-gateway bundles into:
# git clone https://github.com/defenseunicorns/uds-core.git   # the UDS platform
# git clone https://github.com/defenseunicorns/pepr.git        # Kubernetes operator framework used by UDS
```

`peat-node` is intentionally not in the loop above — it is already a checked-out, first-class repo
(§7.1), not a mirror to confirm. For Android work, start from the in-repo example at
`peat/examples/android-peat-demo` and only chase a standalone repo if the team confirms one exists.
`peat-sdk-go` is a Phase-3 roadmap item; clone it only once you've confirmed the repo is real.

---

## 7.5 Primary sources inside the repo (and how much to trust each)

These in-repo documents are your foundations. The crucial habit for a skeptical reader: **rank them
by recency against code.** The order of authority is **code > current ADRs > specs > the developer
guide and README**. The guide and the specs are valuable for the "why" and the precise contracts,
but several of them predate the audited HEADs and have known drift — so read them with the source
open.

- **[`peat/docs/guides/developer/DEVELOPER_GUIDE.md`](../peat/docs/guides/developer/DEVELOPER_GUIDE.md)**
  — the onboarding guide: environment setup, runtime architecture, core concepts, crate reference,
  testing, mobile, edge AI, and "Extending Peat." This learning track is a guided path through it.
  **Caveat:** it is a **2025-12-08 snapshot that predates every audited HEAD** (peat-mesh rc.43, the
  rc.12 FIPS crypto swap dated 2026-05-18, the ADR-066 hierarchy rename still in flight). Where the
  guide and the code differ, **the code wins** — quoting the guide without checking the source is how
  the known stale-doc errors (wrong RBAC role names, ChaCha20 crypto, legacy hierarchy terms) get
  laundered back in.
- **[`peat/docs/ARCHITECTURE.md`](../peat/docs/ARCHITECTURE.md)** — the crate/layer model and security
  overview (older, 2025-01-07). Good for the packaging lens; check it against the manifests. The
  document already flags itself as old, which is the right posture.
- **[`peat/docs/guides/developer/FORMATION_AND_LEADERSHIP.md`](../peat/docs/guides/developer/FORMATION_AND_LEADERSHIP.md)**
  — the code-level how-to for cell formation, the formation-key HMAC handshake, and leader election
  (the source for **Module 2·5**). **This file is present in this checkout** — read it directly. (An
  earlier note here told you to `git pull` because the file might be missing; that was wrong, and it's
  removed.)
- **[`peat/docs/guides/QUICKSTART.md`](../peat/docs/guides/QUICKSTART.md)** &
  **[`peat/docs/guides/operator/OPERATOR_GUIDE.md`](../peat/docs/guides/operator/OPERATOR_GUIDE.md)**
  — the hands-on run-a-mesh walkthrough and the operator/deploy reference (the source for **Module 8**).
- **[`peat/docs/spec/`](../peat/docs/spec/)** — five normative IETF-style specs
  (001 transport · 002 sync · 003 schema · 004 coordination · 005 security) — the source for
  **Module 9**. The precise wire-format contract. **Caveat:** all five are **Draft (2025-01-07)**
  *except* **005-security, amended 2026-05-18 (rev 0.2.0) to AES-256-GCM** and now FIPS-clean. Several
  spec values are contradicted by the code — Device ID width (spec 32 bytes, code 16), the RBAC role
  enum, peat-lite and BLE wire formats, the leader-election weights. Treat the specs as drafts to
  reconcile against code, not as settled truth.
- **[`peat/docs/whitepaper/`](../peat/docs/whitepaper/)** — the conceptual/positioning narrative
  (chapters 01–09), distilled in **Module 1·5 (The Big Idea)**. It is the source of the validated lab
  metrics (the 2/12/24-node runs and the single-machine simulation to ~1,000 nodes). It has its own
  build system (`make html`/`pdf`/`docx`) and large appendices: `10b-spec-appendix.md` and
  `11-adr-appendix.md` (the concatenated ADRs). Read the narrative chapters for the "why"; treat the
  appendices as a reference. (An earlier "~47k lines" figure for the ADR appendix was unsourced and is
  dropped.)

## 7.6 The ADR archive — your deepest primary source

`peat/docs/adr/` holds **75 Architecture Decision Records (and growing)** — confirmed by a file count,
not the "~60" an earlier draft estimated; open issue #695 ("triage 22 Proposed ADRs before public
release") shows the count trending up. `peat-mesh/docs/adr/` (14) and `peat-btle/docs/adr/` (6) hold
more. When you want to know *why* something is the way it is, these beat any summary — including this
one.

**An essential reading habit: check each ADR's status, because PEAT's code is frequently ahead of its
ADRs.** Almost every foundational ADR below is formally **Proposed** even though the code already
ships the decision. **ADR-041 (multi-transport embedded integration) is the only Accepted ADR in the
core set.** High-value starting ADRs:

- **ADR-007 / ADR-011** — the CRDT backend choice (Automerge + Iroh). The choice is **Shipped**, but
  **ADR-011 is formally Proposed** — a clear example of code-ahead-of-ADR.
- **ADR-006 / ADR-048** — security architecture and membership certificates. Membership-cert /
  enrollment work is open epic #592. **ADR-006 predates the FIPS rule and still carries ChaCha20
  references** — queued for amendment, not shipped behavior. (The shipped crypto is AES-256-GCM; see
  Module 9 and the gap in §7.8 below.)
- **ADR-019** — QoS classification. The framework is **Shipped**; cross-class enforcement is
  **In-flight** (Module 2/3 cover the nuance).
- **ADR-040** — `040-nostr-protocol-lessons.md`. Negentropy set reconciliation is the lesson PEAT
  applied from it (issue #435), **Shipped**.
- **ADR-049** & peat-mesh **ADR-0002** — the peat-mesh extraction; explains why the dependency arrow
  flips (networking/CRDT/crypto live in external peat-mesh, and peat-protocol re-exports them).
- **ADR-059** — cross-transport document bridging; explains the peat-btle ↔ peat-mesh cycle break.
  **ADR-059 is Proposed**; the *shipped* part is the `Translator` codec, and peat-btle 0.4.0 dropped
  its peat-mesh back-edge (the cycle break is tracked under peat-mesh #828).
- **ADR-060 / ADR-062** — FIPS crypto posture and HKDF key derivation. **ADR-060 is Proposed, but its
  §5 decisions already ship** (X25519 → ECDH-P256 and ChaCha20 → AES-256-GCM, landed in peat-mesh
  rc.12, dated 2026-05-18). This is the single most important code-ahead-of-doc case for a security
  reviewer: the algorithms in the code are FIPS-approved even where the READMEs still say otherwise.

---

## 7.7 The gaps that aren't repos: capability gaps

A repo map can make the system look complete — every box has a download link. The harder question a
skeptical reader asks is *which described capabilities are actually built.* Several headline
capabilities are **not shipped**, and a reader who greps the code will find that out fast. Naming them
here is the honest version of "what's missing," and each one doubles as a place to contribute (the
full contribution map, with sizes and starting points, is in
[`learning/review/gaps-contributions.md`](review/gaps-contributions.md)).

| Capability | Status | What's really there | Anchor |
|---|---|---|---|
| **`command_log` CRDT** (an append-only, causally-ordered, authority-gated type for orders) | **Speculative** | Does not exist in any repo. Commands today are ordinary JSON documents in a `commands` collection (`peat-node/proto/sidecar.proto:342-373`). The missing half of capability-based tasking — under **autonomy under human authority**, an order must not be silently lost, reordered, or merged into ambiguity, and no shipped CRDT guarantees that for orders. | ADR-046 (Proposed); epic #853 |
| **Targeted message delivery** (send a command only to `role:strike`, not the whole cell) | **In-flight** | Today a command propagates to the cell and receivers filter; `CapableScope` is reserved-but-rejected in peat-node v1. | ADR-046 (Proposed); epic #853 (+#854–#859); binary-distribution epic #780 |
| **Satellite (SBD) and LoRa transports** | **Proposed** | No crate, no module. `TransportType::{Satellite, LoRa}` are enum variants that resolve to "no transport registered" (`peat-mesh/src/transport/manager.rs:1317`). The entire constrained / "Off the Grid" PACE story rests on transports that aren't built. | ADR-051 (SBD); ADR-052 (LoRa) |
| **Single-burst anti-entropy digest** for one-shot constrained links | **Speculative** | The shipped reconciliation is **negentropy over interactive Iroh QUIC** (multiple round trips). A satellite/LoRa link (one ~1,960-byte frame, 5–20 s one-way) needs a single-burst digest scheme; version-vector, snapshot-since-T, hierarchical-digest, and IBLT designs are teaching-only, not implemented. | ADR-040 (negentropy, Shipped); ADR-063 (Proposed, #935) |
| **MLS group key agreement** (forward secrecy, RFC 9420) | **Proposed** | Documented as a shipped Layer-4 mechanism, but there is no `openmls`/`mls-rs` anywhere. Group-key rotation today is leader distribution. A security overclaim until relabeled. | ADR-044 (Proposed; itself carries pre-FIPS ChaCha20) |
| **QoS end-to-end enforcement** ("<5 s P1", cross-class preemption) | **In-flight** | The QoS framework ships (5 classes, allocation, eviction) and peat-node orders relay fanout by class, but a Critical bundle does **not** pause an in-flight Bulk transfer in v1, and "<5 s P1" is a target, not a validated SLA. | ADR-019; peat-node v1 caveats (`sidecar.proto:551-578`) |
| **Tombstone retention / sync-reliability GC** | **In-flight** | GC knobs exist in peat-node (168 h / 7-day default), but per-collection deletion-policy enforcement through peat-mesh is not wired, and several sync-reliability bugs are open. This governs what re-syncs after a long offline window. | peat-node #136; sync bugs #850/#829/#873 (**not** #857 — #857 is ADR-046 Phase-4 selectors) |
| **Control-plane and node authentication** | **In-flight** | peat-node's gRPC surface is unauthenticated (#38); peat-gateway's ingress AuthZ is a permissive stub (#99) with no broker ACLs (#97) or NATS auth/TLS (#124/#125), and enrollment validates a pubkey for length only (no proof-of-possession). The admin API is fully open when `PEAT_ADMIN_TOKEN` is unset. Feature-rich, but not production-secure yet. | peat-node #38; peat-gateway #99/#97/#124/#125 |
| **Hierarchy vocabulary rename** (military → abstract; leaf `Node` vs `Platform`) | **In-flight** | The workspace mixes vocabularies: peat-mesh and peat-protocol use **`Node`** as the leaf; peat-btle is fully legacy `Platform/Squad/Platoon/Company`; peat-schema proto still ships `SquadSummary`. ADR-066 (the abstract vocabulary) and ADR-068 (the base-unit name) are both **Proposed**. No shipped enum uses `Platform` as the leaf. | ADR-066, ADR-068; epics #904, #968, #970 |

## 7.8 The FIPS posture in one place (the most important caveat)

The crypto situation deserves its own note because it is both better and subtler than the READMEs
suggest, and it is exactly what a defense-prime security auditor will probe:

- **The shipped algorithms are FIPS-approved.** As of the rc.12 swap (2026-05-18), the code uses
  **AES-256-GCM** (symmetric AEAD), **ECDH P-256** (key agreement), **Ed25519** (signatures),
  **HKDF-SHA-256** (KDF), **HMAC-SHA-256** (formation auth), and **SHA-256** (hashing). The TLS stack
  uses `aws-lc-rs`, not `ring`. This is a genuine improvement over the docs, which still describe
  ChaCha20-Poly1305 and X25519.
- **But "FIPS-approved algorithm" is not "FIPS-validated module."** The `aes-gcm` and `p256` crates
  are pure-Rust RustCrypto implementations, **not CMVP-validated cryptographic modules.** An auditor
  will reject "FIPS 140-3" for software that isn't in a validated module. For a real FIPS 140 boundary
  the path is the KMS/Vault HSM backends in peat-gateway; migrating peat-btle's crypto to `aws-lc-rs`
  is **In-flight (peat-btle #75)**.
- **The one real FIPS *conflict* is in a Proposed ADR, not shipped code.** **ADR-052 (peat-LoRa,
  Proposed) specifies ChaCha20-Poly1305.** Since LoRa is a constrained link, the right fix before any
  implementation is an object-security envelope (an OSCORE / AES-CCM profile) rather than propagating
  a non-FIPS cipher. Until then, **flag ChaCha20 wherever it appears in the ADRs and stale READMEs;
  never carry it forward into new material.**
- **Also not in code:** P-384 (only P-256 ships), and the ARM-without-crypto-extensions performance
  envelope is unbenchmarked (peat-mesh #126).

---

## Checkpoint

- Which six repos are local, and what is each one's role in a sentence?
- Why is peat-node a first-class component rather than a mirror of `peat-mesh-node`?
- Name three capabilities the docs describe that are **not Shipped**, and give each its status label
  and a place to contribute.
- A reviewer asks "is PEAT's crypto FIPS-validated?" — what's the precise, honest answer?

---

That's the track. Loop back to [Module 0](00-START-HERE.md) any time, or open the interactive
[`index.html`](index.html) for the navigable version.
