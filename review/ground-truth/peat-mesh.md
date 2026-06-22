# Ground Truth — peat-mesh

**Audited HEAD:** `71fc3d5` ("chore: bump to 0.9.0-rc.43"), branch `main` (advanced from `00ab0c9`
on the 2026-06-19 incremental — see the dated delta at the end of this file). `git fetch` ran clean;
working tree not modified.

**Crate version:** `0.9.0-rc.43` (`Cargo.toml:3`). Note: README still advertises `0.3.2` /
`0.1.0` in its install snippets (`README.md:29,46`) — stale.

Status labels per the §1 contract: **Shipped** (in code, tested), **In flight** (open
issue/PR), **Proposed** (ADR, no impl), **Speculative** (teaching-only).

---

## 1. Transports — what is actually implemented

The pluggable seam is the `MeshTransport` trait (`src/transport/mod.rs:375`) plus the extended
`Transport` trait that adds capability advertisement (`src/transport/capabilities.rs:27`). The
`TransportType` enum (`src/transport/capabilities.rs:54-71`) lists eight categories: `Quic`,
`BluetoothClassic`, `BluetoothLE`, `WifiDirect`, `LoRa`, `TacticalRadio`, `Satellite`,
`Custom(u32)`. **The enum is a taxonomy, not an implementation inventory** — most variants have
no backing transport.

Concrete impls actually present (grep `impl .* (Mesh)?Transport for`, excluding test mocks):

| Transport | Type | Impl site | Status | Notes |
|---|---|---|---|---|
| **Iroh / QUIC** | `Quic` | `src/transport/iroh_mesh.rs:508` (`MeshTransport`), `:691` (`Transport`) | **Shipped** | Primary mesh transport. ADR-062 Phase 2 (peat#926). iroh `=1.0.0-rc.1` pinned, `tls-aws-lc-rs` (FIPS), no n0 relay by default. |
| **BLE / GATT** | `BluetoothLE` | `src/transport/btle.rs:208` (`MeshTransport`), `:342` (`Transport`); struct `PeatBleTransport<A: BleAdapter>` `:120` | **Shipped (feature `bluetooth`)** | Delegates to `peat-btle` (`>=0.4.0,<0.4.1`, currently a git `patch.crates-io` rev, `Cargo.toml:227`). |
| **Peat-Lite UDP bridge** | (lite, not a `TransportType` variant) | `src/transport/lite.rs:841` (`MeshTransport for LiteMeshTransport`) | **Shipped (feature `lite-bridge`)** | UDP bridge to ESP32/embedded Lite nodes; OTA + telemetry. ADR-035. |
| **UDP bypass** | n/a | `UdpBypassChannel` `src/transport/bypass.rs:930` | **Shipped (primitive, not a `MeshTransport`)** | It is a UDP send/recv *channel* (ADR-042), NOT a `MeshTransport`/`Transport` impl. README listing "UDP bypass" alongside BLE as a "pluggable transport" (`README.md:15`) overstates it — it is a lower-level bypass channel. |
| LoRa | `LoRa` | none | **Proposed (ADR-052, peat repo)** | Appears only as an enum variant and as a `preference_order` default entry (`src/transport/manager.rs:149`) and in route/selection tests. No `impl Transport`. `TransportManager::get_transport(LoRa)` returns `None` in tests (`manager.rs:1317`). |
| Satellite / SBD | `Satellite` | none | **Proposed (ADR-051, peat repo)** | Enum variant only. No code. No `peat-sbd` / `Iridium` / `+SBDIX` reference anywhere in `src/` or `Cargo.toml` (grep clean). |
| Tactical radio, WiFi Direct, Bluetooth Classic | resp. variants | none | **Speculative/Proposed** | Enum variants only; WiFiDirect also in default `preference_order` (`manager.rs:147`) but has no impl. |

**Verdict on the high-risk transport claim:** QUIC/Iroh, BLE (peat-btle), and the Peat-Lite UDP
bridge are shipped. **peat-sbd (ADR-051) and peat-lora (ADR-052) are ADR proposals only — no
crate, no module, no `impl` in this repo.** The `TransportType::{LoRa,Satellite}` enum members
exist so the routing/selection layer can be configured for them, but selecting one resolves to
"no transport registered."

### n0 relay posture (FIPS/air-gap relevant)
Default iroh endpoint uses `Endpoint::empty_builder` — **no third-party relay, no n0 DNS pkarr**
(`src/lib.rs:19-36`, `src/transport/iroh_mesh.rs:14-34`). The `relay-n0-hosted` feature
(off by default, `Cargo.toml:192`) opts into `*.iroh.network` relays. As of **rc.42** the choice
is also a *runtime* arg: `relay_policy_builder(enable_n0_relay)` (CHANGELOG rc.42 / PR #259,
commit `be763dc`). Correct posture for tactical/air-gapped deployments. (peat#833)

---

## 2. CRDT types

Two distinct CRDT surfaces:

1. **Automerge backend (default feature `automerge-backend`)** — `automerge = 0.9.0`
   (`Cargo.toml:127`). The mesh stores arbitrary Automerge documents; the CRDT semantics are
   Automerge's (RGA/observed-remove text & maps), not an enumerated set. Sync uses the Automerge
   sync protocol over Iroh QUIC streams (`src/storage/automerge_sync.rs:1-25`, cites
   arxiv 2012.00472). **Shipped.**

2. **Peat-Lite bridge CRDTs (feature `lite-bridge`)** — the typed CRDT enum `CrdtType` is
   **imported from `peat-lite`**, not defined here (`src/transport/lite.rs:69-70`, re-exported at
   `src/transport/mod.rs:77`). The variants the bridge encodes/decodes are **`LwwRegister`,
   `GCounter`, `PnCounter`, `OrSet`** (codec fns `decode_orset` `lite.rs:1339`, `CrdtType::PnCounter`
   `:1481`, `CrdtType::OrSet` `:1486`, `CrdtType::GCounter` `:1490`, `CrdtType::LwwRegister` `:1496`;
   `OrSetElement` struct `:86`). **There is NO `command_log` / `CommandLog` CRDT** — grep clean.
   This confirms the contract's expected finding. **Shipped (these four types).**

`FULL_CRDT` constant is re-exported (`src/transport/mod.rs:79`).

---

## 3. Identity / addressing

Two layered identifier types — easy to conflate:

- **`transport::NodeId`** (`src/transport/mod.rs:107`) is a **thin `String` newtype**
  (`pub struct NodeId(String)`). It is NOT itself a hash of a key — it is whatever string a
  transport assigns (iroh EndpointId string, a hex DeviceId, etc.).
- **`security::DeviceId`** (`src/security/device_id.rs:33`) is `[u8; 16]` =
  **first 16 bytes of SHA-256(Ed25519 verifying-key bytes)** (`from_public_key`
  `device_id.rs:39-47`). 32 hex chars. `From<DeviceId> for NodeId` produces the hex string
  (`device_id.rs:109-113`); `TryFrom<&NodeId> for DeviceId` parses it back (`:115-121`).

**Verdict on the "NodeId = SHA-256 of an Ed25519 public key" claim:** Substantially correct but
imprecise. The identity *derivation* is `DeviceId = SHA-256(Ed25519 pubkey)[..16]` (truncated to
128 bits), and a `NodeId` *can be* the hex of that DeviceId. The mesh `NodeId` type itself is a
generic string and does not enforce that derivation. Any doc that says "the NodeId is the SHA-256
of the Ed25519 key" should specify: it is the **first 16 bytes** of that hash, surfaced via
`DeviceId`, and `NodeId` is the transport-level string form.

`btle_to_peat_node_id` bridging: **not found in this repo** under that name (grep clean). The
cross-transport identity bridging lives behind the `Translator` trait
(`src/transport/translator.rs`, ADR-059) and `BleTranslator` (`src/transport/btle_translator.rs:60`).
Any `btle_to_peat_node_id` helper would be in `peat-btle`, not here — flag for the peat-btle audit.

---

## 4. Hierarchy enum (ADR-066 vocabulary)

`HierarchyLevel` (`src/beacon/types.rs:56-67`):

```
Node = 0, Cell = 1, Cohort = 2, Federation = 3, Coalition = 4
```

with `parent()` / `child()` chaining (`types.rs:71-86`). Sizing comments: Cell 4–13 nodes,
Cohort 2–4 cells, Federation 2–4 cohorts (`types.rs:59-65`).

**Discrepancy vs ADR-066:** the contract's expected current vocabulary is
**Platform/Cell/Cohort/Federation/Coalition**. This repo's leaf level is still named **`Node`**,
not **`Platform`**. Cell/Cohort/Federation/Coalition match. So the rename (peat#904 vocabulary
epic) is **partially landed in peat-mesh**: the upper four tiers match ADR-066, but `Node` →
`Platform` has NOT been applied here. Curriculum must not present "Platform" as the leaf term for
peat-mesh code without flagging that the code says `Node`.

`NodeRole` (`src/hierarchy/mod.rs:18-26`): `Leader`, `Member`, `Standalone` (default). This is the
intra-level role, distinct from RBAC roles (RBAC lives in the `peat` core, not here).

---

## 5. Leader election — deterministic, no consensus

`DynamicHierarchyStrategy::determine_role` (`src/hierarchy/dynamic_strategy.rs:190-224`) is a
**deterministic capability-based scoring** function, NOT a consensus/voting protocol:

- `calculate_leadership_score` (`:100-138`) = weighted sum of mobility (Static>SemiMobile>Mobile),
  resource availability (1 − CPU%, 1 − mem%), battery%, ×1.1 if `can_parent`, ×(1 + parent_priority/255).
- A node becomes `Leader` iff `my_score >= best_peer_score * (1 + hysteresis)` (`:217-223`),
  else `Member`; `Standalone` when no same-level peers (`:201-204`).

No Raft/Paxos, no quorum, no vote messages — each node independently computes the same ordering
from observed beacons. **Verdict: deterministic election claim is correct. Shipped.**
Strategies: `StaticHierarchyStrategy`, `DynamicHierarchyStrategy`, `HybridHierarchyStrategy`
(`src/hierarchy/mod.rs:10-12`).

---

## 6. Crypto primitives — FIPS posture (ADR-060) ALREADY MET in source

**This is the most important correction for the curriculum.** The README and the source diverge:

- **README (`README.md:16,173`) still lists "ChaCha20-Poly1305 encryption" and "X25519 key
  exchange."** STALE.
- **Actual source primitives** (`src/security/encryption.rs:1-45`):
  - **AES-256-GCM** (`aes-gcm = 0.10`, `Cargo.toml:47`; `Aes256Gcm` `encryption.rs:38-41`) — AEAD,
    NIST SP 800-38D.
  - **ECDH on NIST P-256** (`p256 = 0.13` features `ecdh`, `Cargo.toml:52`; `encryption.rs:43`) —
    NIST SP 800-56A. (`EncryptionKeypair` `encryption.rs:72`.)
  - **HKDF-SHA256** (`hkdf`/`sha2`, `encryption.rs:42,45`) — NIST SP 800-56C / 800-108.
  - **Ed25519** (`ed25519-dalek = 2`, `Cargo.toml:44`) for identity/signing.
  - **HMAC-SHA256** challenge-response for formation-key auth (`src/security/formation_key.rs:13,34,49`).
- `Cargo.toml:46-52` documents the swap explicitly: "Was `chacha20poly1305 = 0.10`; swapped to
  `aes-gcm`" and "Was `x25519-dalek = 2`; swapped to `p256`" under the FIPS posture, dated
  2026-05-18 (ADR-049 Phase 5, ADR-060 §5).
- TLS provider is `aws-lc-rs`, NOT `ring` (`Cargo.toml:121,136-141`) — `ring` is not FIPS 140-3
  validated; the iroh QUIC stack and kube-rs both select `tls-aws-lc-rs` (peat#923).

**Verdict on the FIPS/ChaCha20 high-risk claim:** In **peat-mesh source**, ChaCha20-Poly1305 and
X25519 have been **removed** and replaced with FIPS-approved AES-256-GCM + ECDH-P256. There is
**no FIPS violation in peat-mesh's encryption module.** The ChaCha20 violation the contract warns
about lives in **peat-btle and ADR-052 (LoRa)** — sibling repos/ADRs — not here. The only residual
"ChaCha20" mentions in peat-mesh are in the deprecation comments in `Cargo.toml` and `encryption.rs`
documenting the swap. Curriculum claiming peat-mesh "uses ChaCha20-Poly1305" (and the README) is
**Outdated/Wrong** as of rc.42.

Open follow-up (In flight): **peat-mesh#126** — benchmark AES-256-GCM + ECDH-P256 on
ARM-without-crypto-extensions; `encryption.rs:13-24` warns AES-GCM in software on pre-ARMv8-crypto
cores is ~5–10× slower than ChaCha20 was, and ECDH-P256 ~3–5× slower than X25519 for session setup.
This perf envelope is **unverified** (no benchmark numbers in repo).

---

## 7. Public API a consumer integrates against

Top-level re-exports (`src/lib.rs:74-107`):

- **Facade:** `PeatMesh`, `PeatMeshBuilder`, `MeshConfig`, `MeshState`, `MeshStatus`,
  `PeatMeshEvent`, `MeshError` (`mesh.rs`; builder/state API `mesh.rs:154-456`). Lifecycle:
  `start()`/`stop()`/`state()`/`status()`/`subscribe_events()`; injectors `set_transport`,
  `set_transport_manager`, `set_hierarchy`, `set_discovery`, `set_device_keypair`,
  `set_formation_key`, `set_beacon_broadcaster` (`mesh.rs:224-456`).
- **Document layer:** `Node` (`src/node/mod.rs:43`) over a `DataSyncBackend` — `publish`,
  `publish_with_origin`, `get`, `query`, `delete`, `observe`→`ChangeStream` (`node/mod.rs:67-149`).
  `sync::InMemoryBackend` is a re-exported test/embedded backend (`lib.rs:99`).
- **Transport seam:** `MeshTransport`, `MeshConnection`, `Transport`, `TransportManager`,
  `TransportType`, `TransportCapabilities`, `NodeId`, `PeerEvent`, `ConnectionHealth`,
  `DisconnectReason` (`lib.rs:93-96`, `transport/mod.rs`, `transport/capabilities.rs`).
- **Hierarchy/topology/routing/beacon/qos** modules all re-exported (`lib.rs:75-92`).
- **Broker (feature `broker`):** Axum HTTP/WS service (`src/broker/`), incl. OTA routes.

Quick-start in README (`README.md:51-64`) uses `PeatMeshBuilder::new().with_config(...).build()`
— note the README example uses `.build().await?` whereas `mesh.rs` exposes `PeatMesh::new(config)`
+ `start()`. The exact builder ergonomics should be re-verified against `mesh.rs` before quoting
the README snippet verbatim (it may be aspirational/older).

`DataSyncBackend` is the integration trait; issue **#106** (In flight) tracks exposing the
Automerge+Iroh primitives as a turnkey `DataSyncBackend` adapter — i.e. today a consumer wires the
backend themselves.

---

## 8. Other shipped capabilities (verified present)

- **Negentropy set reconciliation** — `src/storage/negentropy_sync.rs` (ADR-040, issue #435);
  claims O(log n) rounds, stateless sessions, 32-byte (SHA-256) doc IDs (`negentropy_sync.rs:1-40`).
  `negentropy = 0.5` (`Cargo.toml:154`). **Shipped.**
- **Automerge sync over Iroh QUIC** — `src/storage/automerge_sync.rs`. **Shipped.**
- **Streaming large-blob transfer** — bounded-memory, incremental SHA256, resumable checkpoints;
  profiles `datacenter()` 1 MiB/60s, `tactical()` 256 KiB/30s, `edge()` 64 KiB/10s
  (`README.md:81-85`; `StreamingTransferConfig`, ADR-055). **Shipped.** (Resume re-reads skipped
  bytes — known limitation, issue **#55**, In flight.)
- **CRDT compaction** — per-collection, sync-mode-aware (`LatestOnly` compacted, `FullHistory`
  never); env-driven `PEAT_COMPACTION_*`; off by default (`README.md:88-106`). **Shipped.**
- **Discovery** — mDNS (`mdns-sd 0.19`), static config, Kubernetes EndpointSlice (feature
  `kubernetes`), hybrid (`src/discovery/`). **Shipped.**
- **QoS** — bandwidth allocation, TTL, retention, sync modes, garbage collection, preemption,
  eviction (`src/qos/*`, ADR-0013). **Shipped.**
- **Topology** — partition detection, autonomous operation, metrics (`src/topology/*`). **Shipped.**
- **OTA firmware push to Lite/ESP32** — stop-and-wait reliable transfer, SHA256, optional Ed25519
  signing (`src/transport/lite_ota.rs`, ADR-047; HTTP API `README.md:197-208`). **Shipped (feature
  `lite-bridge`).**
- **Cross-transport document bridging** — `Translator` trait + `BleTranslator` + fan-out
  (`src/transport/{translator,fanout,btle_translator}.rs`, ADR-059). **Shipped (feature `bluetooth`).**

---

## 9. Quantitative claims — verifiable vs not

**Verifiable in this repo:**
- Streaming profiles: 1 MiB/60s, 256 KiB/30s, 64 KiB/10s (`README.md:81-85`).
- Cell sizing 4–13 nodes, Cohort 2–4 cells, Federation 2–4 cohorts (`src/beacon/types.rs:59-65`)
  — these are doc-comment design intents, not enforced/benchmarked invariants.
- Circuit-breaker defaults: failure_threshold 5, open_timeout 5 s (CHANGELOG rc.37/rc.38).
- Negentropy "O(log n) rounds" — claimed in module doc (`negentropy_sync.rs:3`); algorithmic, not
  independently benchmarked here.
- rc.41 blob-stall fix: transient-overload worst case "~180 s → ~31 s" — a lab-observed figure
  stated in CHANGELOG, reproduced from a named convergence sweep, not a CI-asserted number.

**NOT verifiable in this repo (no source/benchmark — flag in curriculum):**
- SBD ~1,960 B MO / ~1,890 B MT, 5–20 s, ~$0.04–0.13/msg — no SBD code here (Proposed).
- LoRa 7–87 km, 1.5–9.1 kB/s — no LoRa code here (Proposed).
- BLE 100–400 m, ~2 Mbps — not in peat-mesh; would live in peat-btle.
- "Automerge ~10 MB vs peat-lite 256 KB" — no such comparison in source.
- "93–99% bandwidth reduction" — grep clean; not in this repo.
- "<5 s P1 latency" — not asserted in code.
- "1,000+ node validation" — no such test/benchmark in this repo (largest referenced lab is the
  7-node failover lab, CHANGELOG rc.37).
- AES-256-GCM/ECDH-P256 ARM performance envelope — explicitly unbenchmarked (issue #126).
- `TransportCapabilities::quic()` advertises 100 Mbps / 10 ms latency (`capabilities.rs:150-160`) —
  these are **hardcoded advertisement defaults**, not measured values; do not cite as PEAT
  performance facts.

---

## 10. ADR index (repo-local, `docs/adr/`, 4-digit form)

`0001` k8s/istio deploy, `0002` peat-mesh extraction, `0003` peer discovery, `0004` pluggable
transport, `0005` E2E encryption/key mgmt, `0006` membership certs/tactical trust, `0007` consumer
interface adapters, `0008` Zarf/UDS, `0009` SDK integration, `0010` DHT peer discovery, `0011`
cross-transport bridging, `0012` android CI toolchain, `0013` QoS priority sync fanout.
README Roadmap marks Phase 2 (MLS, DHT, full PACE failover, Zarf/UDS) and Phase 3 (Go/Python/Kotlin
SDKs, signed CRDT ops, formal benchmark suite) as **Planned/Future** = Proposed
(`README.md:180-189`). Ecosystem ADRs (032/035/042/049/051/052/059/060/066) live in sibling
`peat/docs/adr/`, cited in this repo's Cargo.toml comments and commits.

---

## 11. Assumptions / decisions logged (no questions asked, per §1b)

- **DECISION:** Treated `UdpBypassChannel` as a primitive channel, not a `MeshTransport`, because no
  `impl MeshTransport for UdpBypassChannel` exists (grep clean). README's "UDP bypass (pluggable
  transport)" framing is recorded as an overstatement, not a shipped transport.
- **DECISION:** Recorded `btle_to_peat_node_id` as absent from peat-mesh (grep clean) and deferred
  it to the peat-btle audit rather than asserting it does not exist anywhere.
- **DECISION:** Did not build/run `cargo test` (audit is read-only ground-truth extraction;
  contract says do not modify the working tree). Test *presence* is established from inline
  `#[cfg(test)]` modules and CHANGELOG regression-test names; pass/fail not independently re-run.
- **ASSUMPTION:** `CrdtType` variant set taken from peat-mesh's encode/decode call sites; the
  authoritative enum definition is in peat-lite and should be confirmed in that repo's audit.

---

## 2026-06-19 incremental delta (`00ab0c9` → `71fc3d5`, rc.42 → rc.43)

Four commits. New ground truth, all read against `71fc3d5`:

- **Distribution/file-transfer impl relocated INTO peat-mesh (peat#992).** `src/storage/mod.rs:24-37`
  now owns `blob_announce`, `file_distribution`, `model_distribution` (feature `automerge-backend`),
  with the comment "relocated from peat-protocol per peat#992 — peat-mesh is the canonical iroh
  consumer; transport-specific impl belongs here." The peat-protocol modules of the same name are now
  3-line `pub use peat_mesh::storage::…::*` re-exports. **Shipped.**
- **`DistributionScope` targeting (`storage/file_distribution.rs:98`, `resolve_targets:873`).**
  `AllNodes` (default) and `Nodes { node_ids }` resolve to `blob_store.known_peers()` (the latter
  filtered to known peers). `Formation { formation_id }` and `Capable { min_gpu_gb, cpu_arch,
  min_storage_mb }` are **stubbed** — each logs a `warn!` ("not yet implemented") and falls back to
  all `known_peers`. So targeting is direct-dial-bound. **Shipped (AllNodes/Nodes); In-flight/stub
  (Formation/Capable).**
- **Provider gossip — `peat/blob-announce/1` ALPN (`storage/blob_announce.rs`, peat-mesh#262).**
  `CAP_BLOB_ANNOUNCE_ALPN = b"peat/blob-announce/1"`, `ANNOUNCE_VERSION = 1`, `DEFAULT_ANNOUNCE_TTL =
  3` relay hops, `MAX_HASHES = 4096`, `MAX_ADDRS = 16`. Gossips holdings into the `BlobPeerIndex`
  that `fetch_blob` consults, carrying holder socket addrs so a needer can dial a non-adjacent holder;
  acquiring re-announces. Rides a dedicated ALPN (not a new `SyncMessageType`) **on purpose** — the
  Automerge sync decoder hard-errors on an unknown tag, so a new tag would break mixed-version sync;
  an unknown ALPN just never opens the stream. `classify_announce` gates trust (first-party / relayed
  / forged-origin) against provider/address poisoning. 8 unit + 3 real-iroh e2e tests. **Shipped.**
- **ADR-071 Phase-1 seam (`storage/file_distribution.rs`).** `NeedEvaluator` trait (`needs(&doc)`),
  `CollectionSubscriptionNeed` impl over a shared `Arc<RwLock<HashSet<String>>>`, `collection:
  Option<String>` on `DistributionDocument` (None = directed-only, backward compatible),
  `with_need_evaluator(...)` opt-in (default None), pure `should_deliver(is_directed_target,
  needed_by_interest)` and `can_skip_permanently(...)` gates. **Inert by default**: publish writes
  `collection: None` (TODO at ~`:963`), no evaluator attached unless a consumer opts in. ADR-071 is
  **Proposed**; the seam is **Shipped-but-inert**.
- **Sync fix #261:** inbound-accepted peers are now registered into the blob store's `known_peers`
  (`sync/automerge_backend.rs`, `with_inbound_peer_registry`), so a node you only accepted a dial from
  becomes targetable for distribution — not just one you dialed. **Shipped.**
