# Peat Ground Truth — Merged Authoritative Current-State Model

Synthesized from the six per-repo audits in `learning/review/ground-truth/` (`peat.md`,
`peat-mesh.md`, `peat-btle.md`, `peat-lite.md`, `peat-gateway.md`, `peat-node.md`). Every
curriculum claim must trace to this document. **Code is the source of truth.** Where a repo's
internals were not directly openable in the audit tree, the source citation is tagged
*(via re-export/gbrain)*.

Status labels (used throughout): **Shipped** (in code, tested) · **In-flight** (open issue/PR/epic)
· **Proposed** (ADR in `Proposed` status, no implementation) · **Speculative** (teaching-only design,
not in any repo).

> **The single recurring pattern:** Peat's code is *ahead of* its formal ADRs and *ahead of* its
> own READMEs/specs in some places (FIPS crypto swap, hierarchy enum) and *behind* its docs in
> others (MLS, SBD/LoRa, command_log). Almost every requested foundational ADR (011/032/035/039/
> 060/063/066) is formally **Proposed** even where the code already ships the decision. **Only
> ADR-041 (multi-transport embedded integration) is Accepted.**

---

## 1 · Audited commit per repo

| Repo | Audited HEAD | Crate/pkg version | Origin status |
|---|---|---|---|
| `peat` (umbrella: peat-protocol/schema/transport/persistence/ffi + reserved `peat` facade) | `35d0f11` "raise peat-mesh floor to rc.42 (#990)" | workspace `0.9.0-rc.25`; `peat-ffi` `0.2.7` | up to date (`0 0`), no pull |
| `peat-mesh` | `00ab0c9` "bump to 0.9.0-rc.42 (#260)" | `0.9.0-rc.42` (README stale: advertises 0.3.2/0.1.0) | up to date, clean |
| `peat-btle` | `3d70f48` (HEAD = origin/main) | `0.4.0` | up to date |
| `peat-lite` | `7a8a8fb` "rename hierarchy tiers to abstract vocabulary (ADR-066) (#28)" | `0.2.5`, edition 2021, MSRV 1.70 | even with origin |
| `peat-gateway` | `8d16824` "add dependabot config (#134)" | `0.1.0`; pins `peat-mesh =0.9.0-rc.1` (~40 RCs stale, intentional) | up to date |
| `peat-node` | `4e1b5c8` "deterministic iroh identity (v0.4.3) (#162)" | `0.4.3` | **origin 5 commits ahead** (v0.4.4/v0.4.5 tagged, NOT audited — unverified delta) |

**`peat` umbrella structure:** the `peat` crate itself is a **reserved-name placeholder with zero
deps**. Real code lives in workspace members: `peat-protocol` (core: formation, hierarchy, QoS,
CoT, discovery, distribution; **re-exports peat_mesh + peat_schema** so a consumer depends on
peat-protocol alone), `peat-schema` (Protobuf/`prost`), `peat-transport` (HTTP/REST Axum + TAK/CoT
TCP bridge), `peat-persistence`, `peat-ffi` (UniFFI + JNI). `peat-protocol`'s `security/*` and
`network/iroh_transport` modules are **thin `pub use peat_mesh::…` re-export shims**. `peat-discovery`
was retired under peat#919; discovery now lives in `peat_mesh::discovery`.

---

## 2 · Transport shipped/proposed matrix

| Transport | Mechanism | Status | Where implemented | Evidence |
|---|---|---|---|---|
| **QUIC / Iroh** | TLS 1.3 over QUIC, Automerge sync; default transport | **Shipped** | peat-mesh, re-exported by peat-protocol, embedded by peat-node | `peat-mesh/src/transport/iroh_mesh.rs:508,691`; `peat-protocol/src/network/iroh_transport.rs:25`; ADR-062 Phase 2 (peat#926) |
| **BLE / GATT (peat-btle)** | Cross-platform BLE mesh, MTU chunking, relay | **Shipped, opt-in** (feature `bluetooth`) | peat-btle (`BluetoothLETransport`), bridged by peat-mesh | `peat-btle/src/transport.rs:159-324`; `peat-mesh/src/transport/btle.rs:208`; ADR-039 |
| **peat-lite UDP bridge** | Bridge to embedded/ESP32 Lite nodes; OTA + telemetry | **Shipped, opt-in** (feature `lite-bridge`) | peat-mesh `LiteMeshTransport`; codec in peat-lite; sockets in peat-lite `firmware/` | `peat-mesh/src/transport/lite.rs:841`; `peat-lite/src/protocol/*`; ADR-035 |
| **HTTP / REST (Axum)** | External integration bridge; gateway/broker admin APIs | **Shipped** | peat-transport, peat-mesh `broker`, peat-gateway | `peat-transport/Cargo.toml`; `peat-mesh/src/broker/`; `peat-gateway/src/api/mod.rs:35-93` |
| **TAK / CoT TCP** | TLS bridge to TAK Server wire protocol | **Shipped** | peat-transport | `peat-transport/src/tak/`; ADR-020/028/029 |
| **UDP bypass** | Low-level send/recv channel (NOT a `MeshTransport`) | **Shipped primitive** | peat-mesh `UdpBypassChannel` | `peat-mesh/src/transport/bypass.rs:930`; ADR-042. README's "UDP bypass (pluggable transport)" **overstates** it |
| **peat-sbd (Iridium SBD satellite)** | — | **Proposed only** | nowhere — no crate, no module, no `Cargo` entry | ADR-051 (Proposed); `Satellite` enum variant only in `peat-mesh/src/transport/capabilities.rs:54-71`; grep clean for `iridium`/`+SBDIX` |
| **peat-lora (LoRa)** | — | **Proposed only** | nowhere — `LoRa` enum variant resolves to "no transport registered" | ADR-052 (Proposed; **carries ChaCha20-Poly1305 → FIPS conflict**); `peat-mesh/src/transport/manager.rs:149`, returns `None` at `:1317` |

**Notes.** The `TransportType` enum lists eight categories (`Quic, BluetoothClassic, BluetoothLE,
WifiDirect, LoRa, TacticalRadio, Satellite, Custom(u32)`) — **a taxonomy, not an implementation
inventory.** Only QUIC/Iroh, BLE, and the peat-lite UDP bridge are real `MeshTransport` impls.
WifiDirect and LoRa appear in the default `preference_order` but have no backing impl. **Multi-transport
PACE failover is README Phase-2 Planned = Proposed**, not shipped. **n0 hosted relay is OFF by
default** (`relay-n0-hosted` feature; `Endpoint::empty_builder`, no n0 DNS/pkarr) — correct for
air-gapped/tactical builds; runtime toggle `relay_policy_builder(enable_n0_relay)` added rc.42;
disabling-default tracked by peat#833.

**Cross-transport bridging (ADR-059, Proposed):** the `Translator` trait + `BleTranslator` ship in
peat-mesh (feature `bluetooth`). peat-btle 0.4.0 **dropped its `peat-mesh` back-edge** (Amendment 4
Slice 4.b) — peat-btle now exposes only codec primitives (`translator-codec`) and has **zero
peat-mesh dependency**. Amendment 4 cycle-break tracked by peat#828 (open).

---

## 3 · CRDT types

| Surface | Types | Status | Where |
|---|---|---|---|
| **Document CRDT (core mesh)** | **Automerge** documents (RGA lists/text, observed-remove maps); single CRDT family | **Shipped** | peat-mesh `automerge-backend` (default), `automerge = 0.9.0`; peat-node stores JSON docs as Automerge |
| **peat-lite core library CRDTs** | `LwwRegister<T>`, `GCounter<const N=32>` (+ hybrid `CannedMessageStore` LWW, `CannedMessageAckEvent` OR-set+LWW) | **Shipped** | `peat-lite/src/{lww,counter,canned}.rs` |
| **peat-lite wire-type bytes** | `LwwRegister=0x01, GCounter=0x02, PnCounter=0x03, OrSet=0x04` | **Wire codes only** | `peat-lite/src/protocol/crdt_type.rs:6-11` |
| **peat-lite `PnCounter`** | two G-Counters | **Firmware-only** (workspace-excluded `firmware/`), NOT in the published library | `peat-lite/firmware/src/crdt/pn_counter.rs:14` |
| **peat-lite `OrSet`** | — | **No struct anywhere** — reserved wire byte only | `peat-lite` grep clean |
| **peat-btle CRDTs** | `LwwRegister<T>`, `GCounter`, `EmergencyEvent` (OR-set ACK map), `Peripheral`, `Position`, `HealthStatus`, `VectorClock`; `ChatCRDT` (deprecated, `legacy-chat`) | **Shipped** | `peat-btle/src/sync/crdt.rs`. Hand-rolled byte-packed `no_std` CRDTs — sibling set, **distinct from peat-lite's** |
| **`command_log` CRDT** | — | **Speculative — DOES NOT EXIST** in any repo | grep clean in peat-protocol, peat-mesh, peat-lite, peat-btle, peat-node. Commands are ordinary JSON docs in a `commands` collection (`peat-node/proto/sidecar.proto:342-373`) |

**Anti-entropy / reconciliation:** the shipped mechanism is **`negentropy` set reconciliation**
(claims O(log n) rounds, stateless sessions, 32-byte SHA-256 doc IDs — *algorithmic claim, not
benchmarked*) over the Automerge sync protocol on Iroh QUIC streams (`peat-mesh/src/storage/
negentropy_sync.rs`, `automerge_sync.rs`, cites arxiv 2012.00472; ADR-040 / issue #435; `negentropy =
0.5`). **Re-exported as `NegentropySync` via `peat-protocol/src/storage/negentropy_sync.rs`.**
**Version-vector, snapshot-since-T, hierarchical-digest, and IBLT schemes are NOT implemented** —
they are teaching-only design (peat.md §2; peat-lite has *no* sync engine, no anti-entropy, no leader
election at all).

---

## 4 · Identity / addressing

There are **four different identity schemes** across the stack — a "one derivation used everywhere"
claim is false:

| Layer | Identity type | Derivation | Evidence |
|---|---|---|---|
| **peat-mesh crypto identity** | `security::DeviceId` = `[u8; 16]` (32 hex chars) | **first 16 bytes (128 bits) of SHA-256(Ed25519 verifying-key)** | `peat-mesh/src/security/device_id.rs:33,39-47`; re-exported `peat-protocol/src/security/device_id.rs:6` |
| **peat-mesh transport identity** | `transport::NodeId` = `pub struct NodeId(String)` | a transport-assigned string (iroh EndpointId string, or hex of a DeviceId via `From<DeviceId>`) — NOT itself a hash | `peat-mesh/src/transport/mod.rs:107`, `device_id.rs:109-121` |
| **iroh seed / NodeId** | iroh `SecretKey` | **HKDF-SHA-256** (v2 salt/info) since rc.14 (replaced legacy `SHA-256("peat-iroh-key-v1:"‖seed)`) — **wire-visible NodeId break at rc.13→rc.14**; pre-rc.14 static-peer TOML must be regenerated | `peat-protocol/src/network/iroh_transport.rs:11-19` |
| **peat-node network identity** | iroh `EndpointId` (raw Ed25519 public key — **no SHA-256 wrap**); `--node-id` is a separate human UUID label / HKDF info input | deterministic `HKDF-SHA256(IKM=shared_key, info="iroh:"+node_id)`; pinned by known-answer test; 16-byte IKM floor | `peat-node/src/crypto.rs:119-132`, `src/identity.rs:38-74` |
| **peat-btle NodeId** | `u32` (4 bytes) | **first 4 bytes of BLAKE3(Ed25519 pubkey) LE** (or from BLE MAC, or literal). 32-bit ⇒ birthday-collision risk ~77k nodes | `peat-btle/src/lib.rs:367-371`, `src/security/identity.rs:139-148` |
| **peat-lite NodeId** | `u32`, `#[repr(transparent)]` (4 bytes) | **bare integer — no key derivation.** From BLE MAC or provisioning; `<1000 nodes` collision stated as design assumption | `peat-lite/src/node_id.rs:9-34` |

`btle_to_peat_node_id` bridging (hypothesized in the design note): **not found by that name** in
peat-mesh (grep clean) or peat-btle. Cross-transport identity bridging lives behind the `Translator`
trait / `BleTranslator`. The cross-crate identity hop (u32 ↔ DeviceId) is non-trivial and partly
unverified.

**Spec drift:** spec 001/005 say `PeerId`/`DeviceId` = "first **32 bytes** of SHA-256" — the shipped
`DeviceId` is **16 bytes**. The constrained track's "**256-bit NodeId**" is fabricated.

---

## 5 · Hierarchy vocabulary & roles (mid-rename, internally inconsistent)

**ADR-066 (abstract hierarchy vocabulary) is Status: Proposed.** Intended 5 tiers:
**Platform / Cell / Cohort / Federation / Coalition** (mapping Squad→Cell, Platoon→Cohort,
Company→Federation, Battalion→Coalition). **ADR-068 (Node base-unit) is also Proposed** (proposes
the leaf name). The rename is **mid-flight and the workspace mixes vocabularies**:

| Where | Leaf-tier enum | Vocabulary state |
|---|---|---|
| `peat-mesh/src/beacon/types.rs:56-67` | `HierarchyLevel { Node=0, Cell=1, Cohort=2, Federation=3, Coalition=4 }` | leaf is **`Node`, NOT `Platform`**; upper four match ADR-066. Sizing comments: Cell 4–13 nodes, Cohort 2–4 cells, Federation 2–4 cohorts (design-intent, not enforced) |
| `peat-protocol/src/security/authorization.rs:331-343` | `HierarchyLevel { Node, Cell, Cohort, Federation, Coalition }` | also **`Node`**, already migrated upper tiers; doc-comment notes "was previously squad/platoon/company" |
| `peat-btle/src/lib.rs:495-525` | `HierarchyLevel { Platform=0, Squad=1, Platoon=2, Company=3 }` | **fully LEGACY military vocabulary** — does NOT track ADR-066 |
| `peat-schema/proto/` | `CellSummary`, `CohortSummary`, `FederationSummary`, `CoalitionSummary` (no `Squad`/`squad_id`) | **renamed** to abstract vocabulary (`hierarchy.proto:24,71,72,122,172`) — only `peat-btle/src/lib.rs` is still fully legacy |
| `peat-lite` | (no hierarchy enum at all) | n/a — codec + CRDTs only |
| `peat-node/proto/sidecar.proto:272-302` | `Cell` message (no `HierarchyLevel` enum of its own) | on the **new** vocabulary; enum is upstream |

Epics **#904** (workspace-wide military→abstract rename), **#968** (converge base unit on Node,
ADR-068), **#970** (ADR-068 follow-ups) track the churn. **No shipped enum uses `Platform` as the
leaf.** Any doc presenting "Platform/Cell/Cohort/Federation/Coalition" as the *canonical current*
enum is wrong on two counts: the enum says `Node`, and ADR-066 is Proposed.

**Three distinct role/authority axes — must not be conflated:**

- **RBAC `Role`** (`peat-protocol/src/security/authorization.rs:50-64`): **`Leader, Member, Observer,
  Commander, Admin`** (5). **README §Layer-3 (README.md:262) wrongly says "Observer, Member, Operator,
  Leader, Supervisor"** — `Operator` and `Supervisor` are NOT in the enum (a high-risk factual error
  that the spec 005 enum and several modules repeat). Authorization model deferred pending Layer-1
  device identity (peat#941, open).
- **`CellRole`** (capability axis, `peat-protocol/src/models/role.rs:14-29`): **`Leader, Sensor,
  Compute, Relay, Strike, Support, Follower`** (7). `Support` **is** a variant.
- **`AuthorityLevel`** (human-machine teaming ladder, `peat-schema/proto/node.proto:61-67`):
  `UNSPECIFIED, OBSERVER, ADVISOR, SUPERVISOR, COMMANDER` (5). This is likely where README's
  "Supervisor"/"Observer" leaked in. The whitepaper's six-level Root/Cluster/Formation/Group/Team/Node
  ladder is **whitepaper-only**, not in code.

`CapabilityType` (`peat-schema/proto/capability.proto:11-19`): `UNSPECIFIED, SENSOR, COMPUTE,
COMMUNICATION, MOBILITY, PAYLOAD, EMERGENT`. **There is no `Weapon` type** (weapons are under
`PAYLOAD`); the curriculum invented a `Weapon` row.

**Leader election is deterministic, no consensus** (the high-risk claim, **confirmed correct**).
Two distinct deterministic implementations exist:
- peat-mesh `DynamicHierarchyStrategy::determine_role` (`src/hierarchy/dynamic_strategy.rs:190-224`):
  weighted score of mobility, (1−CPU%), (1−mem%), battery%, ×1.1 can_parent, ×parent_priority; node
  becomes `Leader` iff `my_score >= best_peer_score*(1+hysteresis)`. No Raft/Paxos/votes; each node
  independently computes the same ordering from observed beacons. Strategies Static/Dynamic/Hybrid.
- peat-protocol `cell/leader_election.rs:101-106`: weighted score **`compute*0.30 + communication*0.25
  + sensors*0.20 + power*0.15 + reliability*0.10`**, lexicographic node-id tiebreak (`:118-124`). This
  is the **cell-formation** election and matches spec 004's weights. **The mesh layer does NOT use
  these weights** — two different scoring functions; do not present a single global algorithm.

**Formation handshake = HMAC challenge-response:** `FORMATION_HANDSHAKE_ALPN = b"peat/formation-auth/1"`,
30 s timeout, pre-shared formation key proven via **HMAC-SHA-256**, key never crosses the wire,
constant-time compare (`subtle`) (`peat-protocol/src/network/formation_handshake.rs:49`;
`peat-mesh/src/security/formation_key.rs`). **Shipped.** Routing: cell leaders route upward; non-leaders
cannot cross-cell or reach zone (`peat-mesh/src/routing/router.rs`: `route` :246, Up/Down/Lateral :362-433).

---

## 6 · Crypto primitives & FIPS posture

**The most important correction for the curriculum: the code already migrated off ChaCha20-Poly1305
+ X25519 to FIPS-approved AES-256-GCM + ECDH-P256, dated 2026-05-18 (peat-mesh rc.12, ADR-060 §5).
The READMEs are stale; the code is clean.**

| Function | Algorithm in use | FIPS | Evidence |
|---|---|---|---|
| Identity / signatures | **Ed25519** | FIPS 186-5 (CMVP coverage uneven) | `ed25519-dalek = 2` across all repos |
| Symmetric AEAD | **AES-256-GCM** | SP 800-38D | peat-mesh `encryption.rs:38-41`; peat-btle `mesh_key.rs`/`peer_session.rs`; peat-gateway `crypto/`; peat-node at-rest `crypto.rs` |
| Key agreement | **ECDH P-256** (swapped from X25519, rc.12) | SP 800-56A | peat-mesh `encryption.rs:43` (`p256 = 0.13`); peat-btle `peer_key.rs:33` |
| KDF | **HKDF-SHA-256** | SP 800-56C / 800-108 | iroh seed derivation; peat-btle `mesh_key.rs:124-128`; peat-node `crypto.rs` |
| MAC / formation auth | **HMAC-SHA-256** | FIPS 198-1 | `formation_handshake.rs:49`; `formation_key.rs` |
| Hash | **SHA-256** | FIPS 180-4 | `sha2 = 0.10` everywhere |
| TLS provider | **`aws-lc-rs`** (NOT `ring`) | `ring` is NOT FIPS-validated | peat-mesh `Cargo.toml:121,136-141`; peat-protocol dev-dep iroh pinned `tls-aws-lc-rs` |
| Password hashing | Argon2 (Phase-3 user auth) | n/a (not a FIPS KDF) | `argon2 = 0.5` |
| peat-btle NodeId hash | **BLAKE3** | **NOT FIPS** — but addressing-only, not a security boundary | `peat-btle/src/security/identity.rs:141` |

**Residual FIPS gaps (honest caveats):**
- **Algorithm choice is FIPS-approved; the modules are NOT CMVP-validated.** The `aes-gcm`/`p256`
  crates are pure-Rust RustCrypto implementations, not certified cryptographic modules. There is
  **no `peat-btle#75` and no `aws-lc-rs` BLE-crypto migration** (grep over peat-btle = zero hits); the
  `aws-lc-rs` elsewhere is the rustls/iroh TLS provider, unrelated to the BLE AEAD/key-agreement. For a
  real FIPS 140 boundary the path is the KMS/Vault HSM backends (peat-gateway); the local-KEK path uses
  non-validated (FIPS-approved-algorithm) software AES.
- **The only ChaCha20 FIPS *conflict* in code-adjacent form is in PROPOSED ADRs**, not shipped code:
  **ADR-052 (peat-lora, Proposed)** references ChaCha20-Poly1305 — design-only, no crate. Pre-FIPS
  ADRs 006/044/048/049 and the **READMEs** (peat-mesh `:16,173`; peat-btle `:196,218,242,282`) still
  advertise ChaCha20/X25519 — stale docs, queued for amendment, not shipped behavior.
- **MLS (RFC 9420) is documented-as-shipped but NOT IMPLEMENTED** (high-risk). README §Layer-4 claims
  ciphersuite `MLS_128_DHKEMP256_AES128GCM_SHA256_P256`; grep for `openmls|mls-rs|rfc.?9420|DHKEMP256`
  is empty; no `openmls`/`mls-rs` in any Cargo.lock. MLS is **Proposed (ADR-044)**, which itself
  carries pre-FIPS ChaCha20 references. Must be relabeled Proposed.
- **P-384 is not in code** — only ECDH P-256. Spec/curriculum "P-256/384" is unsupported.
- ARM-without-crypto-extensions performance envelope (AES-GCM ~5–10× slower than ChaCha20 was;
  ECDH-P256 ~3–5× slower than X25519) is **unbenchmarked** — In-flight (peat-mesh#126).
- peat-gateway: decrypted genesis bytes not zeroized after `open()` — peat-gateway#55, In-flight.

---

## 7 · Per-repo control-plane / data-plane reality

- **peat-gateway is a control plane, NOT a node and NOT a data plane.** It implements **zero mesh
  transports**, does not sync CRDTs, and does not sit in the peer-to-peer path. It *observes and
  manages*: external mesh nodes register `Arc<dyn MeshBrokerState>` handles into the gateway's
  read-only API (`src/api/formations.rs:28-30`). Shipped: HTTP REST admin API (axum 0.7), NATS
  JetStream CDC sink + control-plane ingress, HTTP webhook CDC sink, OIDC token introspection
  (RFC 7662), AWS KMS / Vault Transit / local AES-256-GCM envelope encryption (PENV v1, the
  best-tested subsystem), Postgres/redb backends, Helm/Zarf/UDS packaging. **Kafka CDC sink is a TODO
  stub** (`engine.rs:78-80`, does not deliver — In-flight); **Redis Streams does not exist** (no
  `CdcSinkType::Redis` — Proposed/absent). Gateway tier enum `MeshTier{Authority, Infrastructure,
  Endpoint}` maps to mesh wire tiers `Enterprise/Regional/Tactical` (3 of 4 — `Edge` unreachable);
  **neither set is the ADR-066 hierarchy.** Honest gaps: ingress AuthZ is a **permissive stub**
  (#99), no NATS broker ACLs (#97), no NATS auth/TLS (#124/#125), no proof-of-possession at
  enrollment (pubkey validated for length only), `Strict` enrollment policy unimplemented (always
  403s). Admin API is **fully open when `PEAT_ADMIN_TOKEN` is unset**.
- **peat-node is a real, separate, actively-developed gRPC/Connect/gRPC-Web sidecar** (NOT a
  repackaging of any `peat-mesh-node` binary). It embeds peat-mesh + peat-protocol and exposes them
  on a single port. **It speaks Iroh QUIC only — no BLE** (a `docs/DESIGN.md` diagram labels "BLE"
  aspirationally; there is no BLE code path). Discovery: mDNS, Kubernetes EndpointSlice, static
  peering (+ offline `derive-id`). AES-256-GCM encryption at rest (opt-in). QoS-priority relay fanout
  (Critical=1…Bulk=5; `commands`/`contact-reports`/`alerts`→Critical). **The gRPC surface is
  currently unauthenticated** (#38, In-flight). RPC count: README says "25 RPCs" but the proto
  defines **27** / `service.rs` implements all **27** (full parity) — the "25" figure is outdated. v1 honesty caveats:
  attachment priority does NOT enforce wire-level preemption; `DistributionStatus` is sender-side
  only; `CapableScope` is reserved-but-rejected; collection-config deletion policy persisted but not
  yet enforced.

---

## 8 · Key verified numbers (and what is NOT verifiable)

**Verified against code/spec:**
- Streaming transfer profiles: datacenter 1 MiB/60 s, tactical 256 KiB/30 s, edge 64 KiB/10 s
  (peat-mesh `StreamingTransferConfig`, ADR-055).
- peat-lite wire: 16-byte fixed header (MAGIC `"Peat"`, version 1, type, flags 2B LE, node_id 4B LE,
  seq 4B LE); `DEFAULT_PORT = 5555`; multicast `239.255.72.76`; `MAX_PACKET_SIZE = 512`
  (`MAX_PAYLOAD 496`); OTA chunk 448B; unsigned canned msg 22B / signed 86B / sig 64B; ACK event 24B
  base +12B/ACK (max 792B); Document field caps collection 255B / doc_id 65535B / body 65535B; TTL
  defaults LWW 300s, GCounter/PnCounter 3600s, OrSet never. Message types incl. **`Document=0x07`**
  (omitted from peat-lite README).
- peat-btle: BLE capability constants 250 kB/s base / 125 kB/s Coded-S8 (400 m) / 500 kB/s Le2M
  (50 m); 100 m base range; relay `DEFAULT_MAX_HOPS=7`, 300 s seen-cache TTL; `ChunkHeader` 8B
  (`sync/protocol.rs`); 30 s partial-reassembly timeout; mesh-wide crypto overhead 30B (2 marker +
  12 nonce + 16 tag); 16-byte `PeatBeacon` body (full AD ~40B, **requires extended advertising — does
  NOT fit legacy 31B**); GATT service UUID **code-of-record `a1b2c3d4…` / 16-bit `0xA1B2`** (README
  `0xF47A` and `gatt/mod.rs` `f47ac10b…` are stale; spec's `0x1826` matches none — it's the standard
  Fitness Machine UUID). **These BLE rate/range numbers are declared capability constants, NOT
  measured.**
- QoS: 5 classes P1 ~500ms/40% bandwidth, P2 ~5s/30%, P3 ~60s/20%, P4 ~300s/8%, P5 none/2%
  (`peat-protocol/src/qos/mod.rs:41-45`; `QoSClass` Critical=1…Bulk=5). **These are configured policy
  targets, not enforced/validated SLAs** — enforcement is In-flight.
- Tombstone defaults: 168h/7-day retention, 300s GC interval, 1000 batch (peat-node; spec 002 7-day).
- Reconnect watchdog: 5s interval, 5s→120s exponential backoff (peat-node, distinct from spec
  001's transport-layer 100ms→30s ±10% jitter).
- `PEAT_CONNECTION_RECYCLE_SECS` default 60 (iroh memory workaround, ~4–6s outage/min, set `=0` for
  demos) — recycler issue is **#892** (NOT #435; #435 is negentropy/set-reconciliation).
- ADR count in `peat/docs/adr/`: **75** `.md` files (curriculum's "~60" is outdated).
- peat-gateway loadtest (self-reported, debug build): enroll ~137 req/s, read path ~10,239 req/s.
- Whitepaper validation: 2-node <1s/100%, 12-node 26s, 24-node 54s (6.1s mean), **single-machine
  simulation to 1000 nodes with a 1023 hard limit (Linux bridge)**; replication-op reduction 79% at
  24 nodes, CRDT diff-sync 60–95%, hierarchical target "95%+"; TRL 4–5 (whitepaper positioning).

**NOT verifiable in any repo (external hardware specs or unbenchmarked estimates — must be flagged):**
- **"93–99% bandwidth reduction"** — a marketing rounding NOT stated verbatim in the experiments
  whitepaper (which reports 79% / 60–95% / "95%+"). Grep clean in mesh/lite/gateway/node.
- **"1,000+ node validation"** — a single-machine simulation ceiling (1023 cap), NOT field-proven;
  open epics #724/#725/#726/#727 still target 900/1.2K/10K. Largest *code-side* lab referenced is the
  7-node failover lab (CHANGELOG rc.37).
- **"<5s P1 latency"** — a target; QoS enforcement is In-flight, not asserted in code.
- **"O(n log n) vs O(n²)"** — structurally supported by the whitepaper connection table (full mesh
  ~9,120 vs hierarchical ~384 at scale), an analytical/design claim, not a runtime benchmark. The
  whitepaper's *other* figure (500,000 vs ~4,000 at depth 4) is a looser model that disagrees ~10×.
- **SBD ~1,960B MO / ~1,890B MT, 5–20s, ~$0.04–0.13/msg** — Iridium hardware limits (gbrain
  `quic-iridium-sbd-feasibility`), peat-sbd is Proposed-only.
- **LoRa 7–87 km, 1.5–9.1 kB/s** — external LoRa hardware specs; peat-lora is Proposed-only.
- **Automerge ~10 MB vs peat-lite 256 KB** — the 256 KB is a real *design target* (ADR-035, not
  enforced/measured — no allocator cap, no static-RAM assertion); the "~10 MB" Automerge footprint
  is unverified; the derived "~40×" rests on two soft inputs.
- peat-btle battery hours (~6/12/20/36h), "18–24h", "3–4h competitor", "<5% / 20%+ duty cycle" —
  uncited README marketing; UltraLow (36h) profile has **no code path** (not in the `PowerProfile`
  enum: `Aggressive/Balanced/LowPower` only).
- Iridium `+SBDIX` MO+MT-in-one-pass — an external hardware fact, not a Peat fact.
- `TransportCapabilities::quic()` advertises 100 Mbps / 10 ms — **hardcoded advertisement defaults**,
  not measured.
- Integration-depth LOC/latency (Shallow ~500–1000 LOC/~500ms+, Medium ~2000–5000/<50ms, Deep
  ~5000+/<5ms) — author estimates, not measured.

---

## 9 · ADR status table (requested set + key extras)

| ADR | Title | Status | Note |
|---|---|---|---|
| 011 | CRDT+Networking (Ditto vs Automerge/Iroh) | **Proposed** | Automerge/Iroh is the shipped choice; Ditto backend removed |
| 032 | Pluggable transport abstraction | **Proposed** | `MeshTransport`/`Transport` trait seam ships |
| 035 | Peat-Lite embedded nodes | **Proposed** | peat-lite codec + CRDTs ship |
| 039 | Peat-BTLE mesh transport | **Proposed** | peat-btle 0.4.0 ships |
| **041** | **Multi-transport embedded integration** | **Accepted** | the only Accepted ADR in the set |
| 046 | Targeted message delivery | **Proposed** | epic #853 (+#854-#859) In-flight; #768/#769 promote |
| 051 | Peat-SBD satellite transport | **Proposed** | no crate |
| 052 | Peat-LoRa long-range radio | **Proposed** | no crate; **ChaCha20 → FIPS conflict** |
| 059 | Cross-transport document bridging | **Proposed** | Translator codec ships; Amendment 4 cycle-break #828 |
| 060 | Encryption tiers (FIPS posture) | **Proposed** | **§5 already implemented in code** (rc.12 swap) |
| 063 | Persistent multiplexed sync streams | **Proposed** | rc.25 floor cites ADR-063 / peat-mesh#175 (#935) |
| 066 | Abstract hierarchy vocabulary | **Proposed** | partially landed; leaf still `Node` not `Platform` |
| 068 | Node base-unit vocabulary | **Proposed** | #968/#970 track convergence on Node |
| 040 | "Nostr protocol lessons" → negentropy | (shipped lesson) | on-disk title is `040-nostr-protocol-lessons.md`; negentropy is the applied lesson, issue #435 |
| 044 | MLS group key agreement | **Proposed** | **NOT implemented**; carries pre-FIPS ChaCha20 |
| 042 | UDP bypass | (shipped primitive) | `UdpBypassChannel` |
| 047 | OTA firmware push | (shipped) | `lite_ota.rs`, feature `lite-bridge` |
| 055 | Streaming transfer / gateway control plane | (shipped) | profiles; gateway |
| 016 | Collection-lifecycle config | (shipped, partly enforced) | peat-node#55 |
| 065 | Version-negotiated signing | **Proposed** | not confirmed implemented |

Triage epic **#695** ("triage 22 Proposed ADRs before public release").

**Spec files (`peat/docs/spec/`)** are all **Draft (2025-01-07)** — EXCEPT spec **005-security was
amended 2026-05-18 (rev 0.2.0) to AES-256-GCM** and is now FIPS-clean. Spec values that the code
contradicts: Device ID 32B (code: 16B), Role enum Observer/Member/Operator/Leader/Supervisor (code:
Leader/Member/Observer/Commander/Admin), peat-lite wire magic `0xCAFE`/CRC-16 (code: `"Peat"`, no
CRC), BLE GATT `0x1826` (code: `0xA1B2`), leader-election weights 0.30/0.25/… (mesh uses a different
formula). The IRTF DINRG draft `peat/spec/draft-peat-protocol-00.md` exists alongside the per-layer
working specs.

---

## 10 · Open epics / issues (anchors, HEAD `35d0f11`)

- **#853** ADR-046 Targeted Message Delivery epic (+#854-#859; dup #546; specs #770/#771/#772);
  **#780** targeted binary distribution (+#774-#778).
- **#904** military→abstract vocabulary rename (workspace-wide); **#968** converge base unit on Node
  (ADR-068); **#970** ADR-068 follow-ups.
- **#935** ADR-063 persistent multiplexed sync streams (peat-mesh#175); **#932** migrate
  formation_handshake to `&dyn QuicMeshConnection`.
- **#724** Peat-Sim scaling validation 900/1.2K/10K (+#725/#726/#727).
- **#833** disable n0 relay default; **#846** strip ATAK consumer-name refs; **#850/#829/#873**
  sync-reliability bugs; **#828** ADR-059 Amendment 4 cycle-break; **#892** connection-recycle.
- **#592** membership certificates/enrollment (+#588-#591); **#547** ADR-045 Zarf/UDS; **#950**
  ADR-064 ARM64 CI; **#695** triage 22 Proposed ADRs; **#941** authorization model deferred.
- peat-mesh: **#106** turnkey DataSyncBackend; **#55** blob resume; **#126** ARM crypto bench.
- peat-btle: **#26** chat-send originate; **#73** reconnect re-delivery; **#45** peer_link_info.
  (No `#75` / `aws-lc-rs` issue exists; the source is already FIPS-clean RustCrypto `aes-gcm`/`p256`.)
- peat-gateway: **#99** ingress AuthZ; **#97** broker ACLs; **#124/#125** NATS auth/TLS; **#55**
  zeroize genesis; **#53** Postgres CI; **#119** DLQ replay API.
- peat-node: **#38** gRPC auth; **#53** targeted-delivery proto; **#100** ReconnectionManager;
  **#17** WebSocket; **#161** iroh stable; **#14** identity provisioning; **#136** tombstone GC.

(The prompt's anticipated "#857 sync GC" did not match: **#857 is ADR-046 Phase-4 selectors**, not
tombstone GC; tombstone GC knobs live in peat-node#136.)
