# Peat — Gaps & Contribution Map

What does not exist yet, why it matters, the linked ADR/issue, a rough size, and how a new
contributor would start. Synthesized from the six ground-truth audits and the 14 ledgers. This
doubles as the "where to contribute" onboarding for a defense-prime engineer.

Status legend: **Speculative** (design only, not even an ADR-complete proposal) · **Proposed** (ADR
exists, no impl) · **In-flight** (open issue/PR/epic) · **Doc-fix** (code is right, docs are wrong).
Size: **trivial** (<1 day) · **small** (days) · **medium** (1–3 wks) · **large** (1+ months).

---

## 1 · `command_log` CRDT — the tasking primitive that doesn't exist

- **Gap.** There is no `command_log` / `CommandLog` CRDT anywhere (peat-protocol, peat-mesh,
  peat-lite, peat-btle, peat-node all grep clean). Commands today are ordinary JSON documents in a
  `commands` collection (`peat-node/proto/sidecar.proto:342-373`).
- **Why it matters.** peat-lite's data types are safe for *facts and counts* (LwwRegister, GCounter)
  but **none is safe for orders** — a tasking collection needs an append-only, causally-ordered,
  authority-gated CRDT so a command can't be silently lost, reordered, or merged into ambiguity.
  This is the missing half of capability-based tasking for autonomy-under-human-authority.
- **Status.** **Speculative** (the type has to be designed). The *targeted-delivery* half is real and
  In-flight (ADR-046).
- **ADR/issue.** ADR-046 (targeted message delivery, Proposed); epic **#853** (+#854-#859); binary
  distribution epic **#780** (+#774-#778); `CapableScope` is reserved-but-rejected in peat-node
  (`sidecar.proto:543-549`).
- **Size.** **large** — needs a CRDT design (likely an OR-add-only log with Lamport ordering +
  signed entries + RBAC write-gating), a wire encoding for both Automerge and the peat-lite/peat-btle
  embedded tiers, and ack/timeout semantics.
- **How to start.** Read ADR-046 and the #853 epic; study `peat-protocol/src/command/{coordinator,
  routing,conflict_resolver,timeout_manager}.rs` (the command-delivery plumbing exists) and the
  `ConflictPolicy` enum (`command.proto:91-99`); prototype the log CRDT in peat-lite alongside the
  existing `CannedMessageAckEvent` hybrid as a model.

## 2 · Anti-entropy digest / constrained-link sync wire format

- **Gap.** The constrained track describes version-vector "reading logs", snapshot-since-T,
  hierarchical digests, and IBLT difference-sketches. **None is implemented.** The shipped reconciliation
  is **negentropy set reconciliation over Iroh QUIC** (`peat-mesh/src/storage/negentropy_sync.rs`,
  `automerge_sync.rs`). peat-lite has no sync engine at all.
- **Why it matters.** Negentropy assumes an interactive QUIC channel (multiple round trips). A
  satellite/LoRa link with 5–20 s one-way latency and a ~1,960 B frame needs a *single-burst* digest
  scheme — the design the track teaches — but it has to be built before any SBD/LoRa transport is
  useful for reconciliation.
- **Status.** **Speculative** (the digest schemes) over **Shipped** negentropy (the QUIC path).
- **ADR/issue.** ADR-040 (Nostr lessons → negentropy, shipped, #435); ADR-063 (persistent
  multiplexed sync streams, Proposed, #935 / peat-mesh#175); design note
  `peat-addressing-transport-sync.md`.
- **Size.** **large** — a new wire format + a single-RTT reconciliation algorithm for a non-interactive
  link.
- **How to start.** Read `negentropy_sync.rs` to see what *is* there; read the design note's digest
  section; define the wire format as an ADR before coding (it depends on §3 below to have a transport
  to carry it).

## 3 · SBD and LoRa transport crates (peat-sbd, peat-lora)

- **Gap.** `peat-sbd` (Iridium SBD) and `peat-lora` (LoRa) **do not exist** — no crate, no module, no
  Cargo entry. `TransportType::{Satellite, LoRa}` are enum variants that resolve to "no transport
  registered" (`peat-mesh/src/transport/manager.rs:1317`).
- **Why it matters.** They are the entire premise of the constrained/"Off the Grid" track and the
  PACE ladder. Without them, beyond-line-of-sight and long-range tactical comms are aspirational.
- **Status.** **Proposed** (ADR-051 SBD, ADR-052 LoRa).
- **ADR/issue.** ADR-051; ADR-052 (**which references ChaCha20-Poly1305 — a FIPS conflict that must be
  resolved before any implementation**, see §4). ADR-032 (pluggable transport seam, Proposed but the
  `MeshTransport`/`Transport` trait ships).
- **Size.** **large** each — implement the `MeshTransport`/`Transport` traits
  (`peat-mesh/src/transport/mod.rs:375`, `capabilities.rs:27`), framing for the constrained MTU,
  and the digest scheme from §2.
- **How to start.** Study a shipped transport impl (`iroh_mesh.rs` or `btle.rs`) as the template;
  fix ADR-052's cipher to AES-256-GCM/an object-security profile first; build against the trait seam.

## 4 · FIPS module validation + the ADR-052/README ChaCha20 residue (OSCORE/AES-CCM fix)

- **Gap.** Two distinct gaps. (a) Shipped crypto uses FIPS-**approved** algorithms (AES-256-GCM,
  ECDH-P256) in **pure-Rust RustCrypto crates — not a CMVP-validated module.** (b) The ChaCha20
  residue: **ADR-052 (LoRa, Proposed) specifies ChaCha20-Poly1305**, and the peat-mesh/peat-btle
  **READMEs and pre-FIPS ADRs 006/044/048/049 still advertise it**.
- **Why it matters.** A defense prime requires a CMVP-validated boundary for FIPS 140-3. And shipping
  any LoRa transport on ADR-052's current cipher would introduce a *real* shipped FIPS violation.
- **Status.** (a) **In-flight** (peat-btle#75: migrate `src/security/*` to `aws-lc-rs`). (b)
  **Proposed/Doc-fix** — fix ADR-052 and the stale READMEs; the object-security (OSCORE/AES-CCM)
  envelope is a proposal.
- **ADR/issue.** ADR-060 (encryption tiers, Proposed but §5 implemented); peat-btle#75; ARM-crypto
  perf bench peat-mesh#126; peat-gateway#55 (zeroize decrypted genesis).
- **Size.** (a) **medium** (swap RustCrypto → aws-lc-rs behind the existing security trait). (b)
  **small** (doc-fix the READMEs/ADRs) + **medium** (design the OSCORE/AES-CCM object-security layer
  for constrained links — `peat-lite`'s `DOC_FLAG_ENCRYPTED` is reserved+encoder-rejected, ready to
  wire).
- **How to start.** For (a) read `peat-btle/src/security/{mesh_key,peer_key,peer_session}.rs` and the
  `aes-gcm`→aws-lc-rs swap pattern already documented in peat-mesh `encryption.rs`/`Cargo.toml`
  comments. For (b) edit the READMEs to match shipped AES-256-GCM (see §10).

## 5 · QoS enforcement (TTL / bandwidth / "<5 s P1" preemption)

- **Gap.** The QoS *framework* ships — 5 classes (Critical=1…Bulk=5), bandwidth allocation, TTL,
  eviction, GC (`peat-protocol/src/qos/`, `peat-mesh/src/qos/`, ADR-019), and peat-node orders relay
  fanout by class. But **cross-class wire-level preemption is not enforced in v1** (a Critical bundle
  does not pause an in-flight Bulk transfer — `peat-node/proto/sidecar.proto:551-578`), and the
  "<5 s P1" latency is a **target**, not a validated SLA.
- **Why it matters.** "Priority 1 latency <5 s end-to-end" is a headline claim; without enforcement it
  is configuration, not a guarantee.
- **Status.** **In-flight** (framework Shipped, enforcement In-flight).
- **ADR/issue.** ADR-019; PRD-004 (attachment preemption, referenced in `sidecar.proto`).
- **Size.** **medium** — wire-level preemption in the sync/transfer path.
- **How to start.** Read `peat-node/src/fanout.rs` (QoS-priority fanout) and the v1 caveats in
  `sidecar.proto:551-656`; trace how `QoSClass::for_collection` maps collections to classes.

## 6 · Tombstone retention / sync-reliability GC

- **Gap.** Tombstone GC knobs exist (peat-node `--tombstone-ttl-hours`, 168h default, 300s GC) but
  per-collection deletion-policy enforcement through peat-mesh is **not wired** (needs
  `AutomergeBackend::set_deletion_policy()`); several sync-reliability bugs are open.
- **Why it matters.** After long offline windows, what re-syncs vs. what is GC'd determines whether a
  reconnecting node converges correctly — central to DDIL operation.
- **Status.** **In-flight.** (Note: the curriculum's cited "#857 sync GC" is **wrong** — #857 is
  ADR-046 Phase-4 selectors.)
- **ADR/issue.** peat-node **#136** (tombstone GC); sync-reliability **#850/#829/#873**;
  collection-config ADR-016 (peat-node#55).
- **Size.** **medium**.
- **How to start.** Read `peat-node/src/node.rs:218-231` (collection-config deletion policy persisted
  but not enforced) and the GC knobs at `:64-71`.

## 7 · Hierarchy vocabulary rename (Node↔Platform; military→abstract)

- **Gap.** The workspace mixes vocabularies: peat-mesh/peat-protocol use `Node` as the leaf;
  peat-btle uses legacy `Platform/Squad/Platoon/Company`; peat-schema proto still ships
  `SquadSummary`/`squad_id`; ADR-066 wants `Platform/Cell/Cohort/Federation/Coalition`; ADR-068
  debates the base-unit name.
- **Why it matters.** A reader (and a contributor) sees four vocabularies coexisting; CohortSummary
  may actually be SquadSummary in proto.
- **Status.** **In-flight** (ADR-066 + ADR-068 both Proposed; rename partially landed).
- **ADR/issue.** ADR-066, ADR-068; epics **#904** (workspace rename), **#968** (converge on Node),
  **#970** (ADR-068 follow-ups).
- **Size.** **medium** (mechanical rename across peat-btle + peat-schema proto, with wire-compat care).
- **How to start.** Read `peat-mesh/src/beacon/types.rs:56-67` (the target enum) and
  `peat-btle/src/lib.rs:495-525` (the legacy one); follow #904.

## 8 · gRPC / control-plane authentication (the unauthenticated surfaces)

- **Gap.** peat-node's gRPC surface is **unauthenticated** (#38). peat-gateway's NATS ingress AuthZ
  is a **permissive stub** (#99), with no broker ACLs (#97), no NATS auth/TLS (#124/#125), and **no
  proof-of-possession at enrollment** (pubkey validated for length only). The admin API is fully open
  when `PEAT_ADMIN_TOKEN` is unset.
- **Why it matters.** For an air-gapped/enterprise deployment these are the security boundary; the
  control plane is feature-rich but **not production-secure yet**.
- **Status.** **In-flight.**
- **ADR/issue.** peat-node#38; peat-gateway#99/#97/#124/#125/#55; enrollment PoP (no issue found —
  open one).
- **Size.** **medium** each.
- **How to start.** peat-node: `src/service.rs` + the #38 ADR. peat-gateway: `src/ingress/mod.rs:282-293`
  (the loud permissive-stub boot log) and `src/api/enroll.rs:273-285` (length-only pubkey check).

## 9 · MLS group key agreement (forward secrecy)

- **Gap.** MLS (RFC 9420) is documented as a shipped Layer-4 security mechanism but **does not exist**
  (no `openmls`/`mls-rs`). Group-key rotation today is leader-distribution.
- **Why it matters.** Forward secrecy for cell group keys is a real requirement; the doc overclaims it.
- **Status.** **Proposed** (ADR-044, which also carries pre-FIPS ChaCha20 references).
- **ADR/issue.** ADR-044.
- **Size.** **large** — must select a FIPS-compatible MLS ciphersuite and integrate group key
  lifecycle.
- **How to start.** Read ADR-044; the formation-key rotation path (`formation_key.rs`) is the current
  baseline to extend.

## 10 · Doc-fixes (code is right, docs are wrong — fast wins for a first PR)

| Doc fix | Where | Size |
|---|---|---|
| README RBAC roles "Operator/Supervisor" → real `Leader/Member/Observer/Commander/Admin` | peat README.md:262 | trivial |
| READMEs say ChaCha20/X25519 → AES-256-GCM/ECDH-P256 | peat-mesh README:16,173; peat-btle README:196,218,242,282 | trivial |
| peat-mesh README version 0.3.2/0.1.0 → rc.42 | peat-mesh README:29,46 | trivial |
| peat-node "25 RPCs" → 28 defined / 27 implemented | peat-node README:132, sidecar.proto header | trivial |
| GATT service UUID consistency (`0xA1B2` is code-of-record; README `0xF47A` + `gatt/mod.rs` `f47ac10b…` stale) | peat-btle README:187, src/gatt/mod.rs:23 | trivial |
| peat-lite README adds missing `Document` (0x07) message type; un-collapse OTA 0x10-0x16 | peat-lite README:59 | trivial |
| peat-btle per-peer E2EE overhead 46B (README) vs 44B (`security/mod.rs:71`) — reconcile | peat-btle README:298-304 vs src/security/mod.rs:71 | small |
| Spec 001/005 Device ID 32B → 16B; spec peat-lite/BLE wire formats; spec Role enum | peat/docs/spec/{001,005} | small |
| Document the AES/P-256 crypto swap in peat-btle CHANGELOG (currently only in Cargo.toml comments) | peat-btle CHANGELOG.md | trivial |

- **Why it matters.** These are exactly the errors a skeptical staff engineer catches with the code
  open. Fixing them is the highest-ROI onboarding contribution and stops the stale docs from
  re-infecting future curriculum.
- **Status.** **Doc-fix.**
- **How to start.** Each row cites its file:line; the code is authoritative.

---

## Cross-cutting: where a new contributor gets the most leverage

1. **Doc-fixes (§10)** — a first PR that makes the docs match shipped code; low risk, high trust.
2. **FIPS module validation (§4a, peat-btle#75)** — the single most important enterprise-readiness
   item; well-scoped.
3. **QoS enforcement (§5)** and **control-plane auth (§8)** — turn feature-rich-but-not-secure
   subsystems into production-ready ones.
4. **The constrained-comms stack (§2, §3)** — the largest greenfield, but the entire "Off the Grid"
   value proposition; start by fixing ADR-052's cipher (§4b) and writing the digest-format ADR.
