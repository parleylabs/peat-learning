# Ground-truth audit — `peat` (umbrella workspace)

**Repo:** `github.com/defenseunicorns/peat` (the `./peat` subdirectory of the umbrella working tree).
**HEAD audited:** `68e9c3c` — `chore: bump workspace to 0.9.0-rc.26` (advanced from `35d0f11` on the
2026-06-19 incremental — see the dated delta at the end of this file).
**Origin status:** `git fetch` ran; advanced 3 commits this run. **No pull/working-tree change beyond fetch.**
**Workspace version:** `0.9.0-rc.26` (`Cargo.toml [workspace.package].version`); peat-mesh dependency
floor `>=0.9.0-rc.43` (`Cargo.toml:287`). `peat-ffi` versions independently at `0.2.7`.

This file is the citations-backed reality model. Every later curriculum claim must trace here.
Labels: **Shipped** (in code/tested) · **In-flight** (open issue/epic) · **Proposed** (ADR in Proposed status, no impl) · **Speculative** (teaching-only).

---

## 0 · What this repo is (and is not)

`peat` is the **umbrella workspace**, not a single crate. The `peat` crate itself is a **reserved-name placeholder** with zero dependencies (`peat/Cargo.toml`: *"Reserved name for the future top-level facade crate… Depend on the per-component peat-* crates for now."*). The real code lives in workspace members:

| Member crate | Role | Status |
|---|---|---|
| `peat-protocol` | Core: cell formation, hierarchy, QoS, security re-exports, CoT, discovery, distribution | Shipped (WIP, `0.9.0-rc.28`) |
| `peat-schema` | Protobuf wire definitions (`prost`) | Shipped |
| `peat-transport` | HTTP/REST (Axum) + TAK/CoT TCP bridge | Shipped |
| `peat-persistence` | Storage abstraction over peat-mesh | Shipped |
| `peat-ffi` | UniFFI + JNI Kotlin/Swift bindings | Shipped (`0.2.9`) |
| `peat` | Reserved facade | Placeholder, no deps |

**Retired:** `peat-discovery` removed under peat#919 (members comment in `peat/Cargo.toml`). Discovery now lives in `peat_mesh::discovery`.

**The networking, CRDT engine, and crypto are NOT in this repo.** They live in the **external** `peat-mesh` crate (`Cargo.lock`: `peat-mesh 0.9.0-rc.43`), which `peat-protocol` pulls in and **re-exports**. peat-protocol's `security/*` modules (`device_id.rs`, `encryption.rs`, `formation_key.rs`, `keypair.rs`) are **thin `pub use peat_mesh::security::…::*` shims** (e.g. `peat-protocol/src/security/encryption.rs:7`). Likewise `IrohTransport` is a re-export shim (`peat-protocol/src/network/iroh_transport.rs:25` → `pub use peat_mesh::network::iroh_transport::*`).

> **Assumption logged:** peat-mesh / peat-btle / peat-lite source is *not* present in this clone (they are external crates.io deps). Claims about their internals below are corroborated via gbrain's indexed copies + the comments/tests in this repo's re-export shims, not by reading their source in this tree. Where a peat-mesh internal could not be opened directly it is marked **(via re-export/gbrain)**.

---

## 1 · Transports — shipped vs proposed

Cargo.lock contains exactly three transport crates: `peat-mesh 0.9.0-rc.42`, `peat-btle 0.4.0`, `peat-lite 0.2.5`. **`peat-sbd` and `peat-lora` appear in NO `Cargo.toml` and NO `Cargo.lock`** (grep across all manifests returned empty).

| Transport | Mechanism | Status | Evidence |
|---|---|---|---|
| **QUIC / Iroh** | `IrohTransport`, TLS 1.3 over QUIC, Automerge CRDT sync | **Shipped** | `peat-protocol/src/network/iroh_transport.rs:25` (re-export of `peat_mesh::network::iroh_transport`); default feature `automerge-backend` (`peat-protocol/Cargo.toml`); `Cargo.lock peat-mesh 0.9.0-rc.42` |
| **BLE mesh (peat-btle)** | Cross-platform BLE (BlueZ/CoreBluetooth/NimBLE/Android/iOS) | **Shipped, opt-in** | `bluetooth` feature → `peat-btle` (`peat-protocol/Cargo.toml [features]`); `Cargo.lock peat-btle 0.4.0`; ADR-039 |
| **peat-lite (embedded UDP)** | `no_std` UDP-gossip wire protocol for MCUs | **Shipped, opt-in** | `lite-transport` feature → `peat-mesh/lite-bridge` (`peat-protocol/Cargo.toml`); `Cargo.lock peat-lite 0.2.5`; ADR-035 |
| **HTTP / REST (Axum)** | External integration bridge | **Shipped** | `peat-transport/Cargo.toml` (axum, tower); README:184 |
| **TAK / CoT TCP** | TLS bridge to TAK Server wire protocol | **Shipped** | `peat-transport/Cargo.toml` (`tokio-rustls`, `src/tak/`); ADR-020/028/029 |
| **peat-sbd (Iridium SBD satellite)** | — | **Proposed only** | ADR-051 **Status: Proposed**; NOT in any Cargo manifest/lock |
| **peat-lora (LoRa long-range)** | — | **Proposed only** | ADR-052 **Status: Proposed**; NOT in any Cargo manifest/lock; *also references ChaCha20-Poly1305 → FIPS conflict (see §5)* |

**Cross-transport bridging (ADR-059):** the `Translator` trait + `TransportManager` machinery is **partially shipped / in-flight**. The `Translator` trait lives in peat-mesh; `peat-transport`'s `mesh-translator` feature provides a CoT `Translator` impl (`peat-transport/Cargo.toml [features]`). ADR-059 itself is **Proposed**, with Amendment 4 cycle-break work tracked under peat#828 (open). The peat-btle ↔ peat-mesh back-edge was structurally dissolved (peat-btle 0.4.0 dropped the `mesh-translator` feature) — see the long resolved-comment block in `peat/Cargo.toml`.

**n0 hosted relay:** off by default (`relay-n0-hosted` feature, `peat-protocol/Cargo.toml`); tactical builds must not phone home. Disabling n0 as default is tracked by open issue **#833**. Runtime toggle added recently (`TransportConfigFFI.enable_n0_relay`, commit `338ffed`).

---

## 2 · CRDT types

- **Core / mesh:** Automerge documents over Iroh (`automerge = "0.9.0"`, `peat-protocol/Cargo.toml`; `automerge-backend` default feature). This is the "document CRDT" tier. **Shipped.**
- **peat-lite primitives (via re-export/gbrain):** `LwwRegister`, `GCounter`, `PnCounter`, `OrSet` only — a pure `no_std` leaf crate whose only dep is `heapless`. **Shipped.** (gbrain `firmware/readme`, `peat-modularity-leaf-crates`.)
- **`command_log` CRDT: does NOT exist.** No `command_log` / `CommandLog` symbol in `peat-protocol/src/` (grep empty); peat-lite ships only the four primitives above. Any curriculum reference to a command-log CRDT is **Speculative**.
- **Anti-entropy / set reconciliation:** `negentropy` (set-reconciliation, ADR-040, issue #435) is wired, re-exported as `NegentropySync`/`ReconcileResult`/`SyncItem` from peat-mesh (`peat-protocol/src/storage/negentropy_sync.rs:2`, `storage/mod.rs:92`). Automerge's own sync protocol carries the differential sync. **Shipped (via re-export).** Version-vector / snapshot-since-T / hierarchical-digest / IBLT schemes described in design notes are **not** separately implemented here — Automerge + negentropy are the actual mechanisms.

---

## 3 · Identity / addressing

- **`DeviceId` = SHA-256(Ed25519 public key), truncated to 16 bytes.** Defined canonically in peat-mesh; re-exported via `peat-protocol/src/security/device_id.rs:6` (`pub use peat_mesh::security::device_id::*`). The re-export tests prove the contract: `from_public_key` is deterministic, distinct keys → distinct ids, `to_hex().len() == 32` ⇒ **16-byte id** (`device_id.rs:38-40`), `from_public_key` derived from the Ed25519 `verifying_key()`. **Shipped.** (Note: README §Layer-1 calls it "device ID is the SHA-256 hash of its public key" — accurate, but it is the 16-byte truncation, not the full 32-byte digest.)
- **iroh NodeId / SecretKey:** `IrohTransport::from_seed*` derives the iroh `SecretKey` via **HKDF-SHA-256** (v2 salt/info) since rc.14, replacing legacy `SHA-256("peat-iroh-key-v1:" || seed)` (`peat-protocol/src/network/iroh_transport.rs:11-19`). **Wire-visible NodeId break at rc.13→rc.14** — pre-rc.14 static-peer TOML must be regenerated.
- **peat_mesh ↔ peat-lite identity bridging (`btle_to_peat_node_id`):** not directly visible in this repo's source (lives in peat-btle/peat-mesh). Marked **Shipped (via re-export/gbrain)** — could not confirm the exact symbol name in this tree; flag for direct peat-btle audit.

---

## 4 · Hierarchy vocabulary & roles

**Two enums named with hierarchy levels; one already migrated to ADR-066 vocab, the other not. The `peat-schema` proto has also migrated off the `Squad*` names (see below). The rename is mid-flight.**

- **`HierarchyLevel`** (`peat-protocol/src/security/authorization.rs:331-343`): `Node, Cell, Cohort, Federation, Coalition` — **already the ADR-066 abstract vocabulary**, with doc-comments explicitly noting "was previously squad/platoon/company." So the security-layer enum is ahead of the ADR's formal status.
- **ADR-066 (abstract hierarchy vocabulary) Status: Proposed.** Map: `Platform→Platform, Squad→Cell, Platoon→Cohort, Company→Federation, Battalion→Coalition`. The ADR is Proposed but `authorization.rs` already uses the new names, and at HEAD `8a94796` `peat-schema/proto/hierarchy.proto` has also been renamed — it now defines `CellSummary`/`CohortSummary`/`FederationSummary`/`CoalitionSummary` with **zero `Squad`/`squad_id`** (`hierarchy.proto:24,71,72,122,172`). The earlier "schema still ships `SquadSummary`/`squad_id`" claim is **stale**. (peat-btle `src/lib.rs` still uses legacy `Platform`/`Squad` — keep that distinct.)
- **ADR-068 (Node base-unit vocabulary) Status: Proposed** — proposes renaming `Platform`→`Node` at the base unit. Open epics **#968** (converge base unit on Node) and **#904** (workspace-wide military→abstract rename) and **#970** (ADR-068 follow-ups) track this. So vocabulary is actively churning: a reader will see `Platform`, `Squad`/`Cell`, `cohort_id`, `platform_id` all coexisting.
- `CellState` already carries `cohort_id` (`peat-protocol/src/hierarchy/maintenance.rs:374` uses `cell.cohort_id`).

**RBAC roles — DOCUMENTATION/CODE MISMATCH (high-risk finding):**
- The **actual** `Role` enum (`peat-protocol/src/security/authorization.rs:50-64`) has **five variants: `Leader, Member, Observer, Commander, Admin`.**
- **README §Layer-3 (README.md:262) claims the five RBAC levels are "Observer, Member, Operator, Leader, and Supervisor."** **`Operator` and `Supervisor` are NOT in the `Role` enum.** This is a factual error in the README — the count (five) is right but two names are wrong.
- Separately, `CellRole` (`peat-protocol/src/models/role.rs:14-29`) is a *different* concept — capability assignment, seven variants `Leader, Sensor, Compute, Relay, Strike, Support, Follower` — not the RBAC privilege model. Curriculum must not conflate `Role` (RBAC) with `CellRole` (capability).
- A third axis: `AuthorityLevel` (`peat-schema/proto/node.proto:61-67`): `UNSPECIFIED, OBSERVER, ADVISOR, SUPERVISOR, COMMANDER` — the human-machine-teaming authority ladder. This is likely where README's "Supervisor"/"Observer" leaked in; the README conflated `AuthorityLevel` with `Role`.

---

## 5 · Crypto primitives & FIPS posture

**peat-protocol's own crypto manifest is FIPS-clean.** The previously-dead `chacha20poly1305` + `x25519-dalek` manifest entries were **removed 2026-05-18** (rc.12 bump) — comment at `peat-protocol/Cargo.toml` says they were "dead manifest declarations (grep `src/` returned zero uses) AND contradicted the FIPS rule." Symmetric AEAD + DH now live in peat-mesh per ADR-060 §5.

| Function | Algorithm in use | FIPS |
|---|---|---|
| Device identity / signatures | Ed25519 (`ed25519-dalek = "2"`, `peat-protocol/Cargo.toml`) | ✅ FIPS 186-5 |
| Symmetric AEAD | AES-256-GCM (peat-mesh, README:251) | ✅ SP 800-38D |
| Key agreement | **ECDH P-256** — swapped from X25519 in peat-mesh rc.12 per ADR-060 §5 (`peat-protocol/src/security/encryption.rs:46`, `security/mod.rs:83-85`: "X25519 → ECDH-P256; public key is 33 bytes compressed SEC1") | ✅ SP 800-56A |
| KDF | HKDF-SHA-256 (`hkdf = "0.12"`; iroh seed derivation `iroh_transport.rs:14`) | ✅ |
| MAC / formation handshake | HMAC-SHA-256 (`hmac = "0.12"`; `FORMATION_HANDSHAKE_ALPN = b"peat/formation-auth/1"`, `network/formation_handshake.rs:49`) | ✅ FIPS 198-1 |
| Hash | SHA-256 (`sha2 = "0.10"`) | ✅ |
| Password hashing | Argon2 (`argon2 = "0.5"`, Phase-3 user auth) | n/a (not a FIPS KDF; flag if used in FIPS-mode path) |

**FIPS violations are confined to PROPOSED ADRs, not shipped code:**
- **ADR-052 (peat-lora, Proposed)** references **ChaCha20-Poly1305** — a FIPS violation per ADR-060 / `peat/CLAUDE.md`, but it is design-only (no crate).
- `peat/CLAUDE.md` lists existing ChaCha20 references in ADR-006/044/048/049, `README.md`, `docs/spec/005-security.md`, `docs/whitepaper/10b-spec-appendix.md` as **predating the FIPS rule, queued for amendment** (ADR-060 §References). gbrain research note `quic-iridium-sbd-feasibility` also cites a "30-byte ChaCha20-Poly1305 overhead" for peat-btle framing — **flag**: if peat-btle's wire crypto is ChaCha20, that is a shipped FIPS conflict; could not confirm in this tree (peat-btle external). Mark **needs direct peat-btle audit**.
- **`ring` vs `aws-lc-rs`:** README:256 correctly states the default `ring` backend is NOT FIPS-validated; iroh must run under `aws-lc-rs`. peat-protocol's dev-dep iroh is pinned `default-features = false, features = ["tls-aws-lc-rs"]` (`peat-protocol/Cargo.toml [dev-dependencies]`) — FIPS-posture parity.

- **ADR-060 (encryption tiers) Status: Proposed** — yet its §5 FIPS-primitive decisions are already implemented (X25519→ECDH-P256 landed rc.12). Another case of code ahead of ADR formal status.

---

## 6 · MLS / group key agreement — DOCUMENTED-AS-SHIPPED BUT NOT IMPLEMENTED (high-risk)

README §Layer-4 (README.md:266) states cells use **MLS (RFC 9420)** with ciphersuite `MLS_128_DHKEMP256_AES128GCM_SHA256_P256`. **No MLS implementation exists:** grep for `openmls|mls-rs|rfc.?9420|DHKEMP256` across `peat-protocol/src` + `peat-ffi/src` returned empty, and `Cargo.lock` has no `openmls`/`mls-rs` entry. MLS is **Proposed** (ADR-044, which `peat/CLAUDE.md` flags as a pre-FIPS ADR carrying ChaCha20 references). Curriculum/README presenting MLS as a shipped security layer is **misleading** and must be relabeled **Proposed**.

---

## 7 · Leader election & formation

- **Leader election is deterministic / selection-criteria-based, NOT consensus.** `peat-protocol/src/hierarchy/maintenance.rs` selects via priority-ordered scoring (capacity, same-zone preference, smaller-cell load balance — `find_merge_candidate`, lines ~336-388). No Raft/Paxos. **Shipped.** The actual cell-leader election logic also lives in peat-mesh's formation path (via re-export); the maintenance module here handles cell merge/split/rebalance and re-election triggers ("Leaders will be re-elected", lines 227/252/312).
- **Formation handshake = HMAC challenge-response.** `FORMATION_HANDSHAKE_ALPN = b"peat/formation-auth/1"` (`network/formation_handshake.rs:49`); 30 s timeout; pre-shared formation key proven via HMAC-SHA-256, key never crosses the wire (README:262, `subtle = "2.6"` constant-time compare in manifest). **Shipped.**
- Routing: cell leaders route upward to zone level; non-leaders cannot cross-cell or reach zone (`peat-mesh/src/routing/router.rs:246, 362-433`). **Shipped.**

---

## 8 · Public API surface a consumer integrates against

From `peat-protocol/src/lib.rs:78-106`, the public modules are: `cell, command, composition, cot, credentials, discovery, distribution, error, event, ffi, geohash, hierarchy, mesh_integration, models, network, policy, qos, security, storage, sync, testing, traits, transport`. It **re-exports `peat_mesh` and `peat_schema`** (`lib.rs:105-106`) so a consumer depends on peat-protocol alone.

Typical entry points (README:86-111): `peat_protocol::network::IrohTransport::from_seed_at_addr`, `AutomergeBackend::with_transport(store, transport)`, `storage::capabilities::SyncCapable`. FFI consumers use `peat-ffi` (UniFFI `0.31` + JNI `0.21`); JNI symbols were renamed `Java_com_defenseunicorns_atak_peat_*` → `Java_com_defenseunicorns_peat_*` (peat-ffi 0.2.0 breaking ABI; stripping the ATAK consumer name tracked by open issue **#846**).

Three integration depths (README:124-126): Shallow REST/HTTP (~500-1000 LOC, ~500ms+), Medium library (~2000-5000 LOC, <50ms), Deep transport (~5000+ LOC, <5ms). These LOC/latency figures are **author estimates, not measured** — flag as **unverified**.

---

## 9 · Quantitative claims — verified vs not

| Claim | README loc | Verdict |
|---|---|---|
| "93-99% bandwidth reduction via hierarchical aggregation" | README:231,299 | **Partly traceable / rounded.** Whitepaper validates **79% replication-op reduction at 24 nodes**, **60-95% CRDT differential-sync reduction**, **"95%+" hierarchical target**, and 86%/96% *connection-count* reduction (`docs/Peat-Lab-Experiments-Whitepaper.md:21,23,293,318-320`). "93-99%" is a marketing rounding not stated verbatim in the whitepaper. **Flag the discrepancy.** |
| "O(n log n) vs O(n²) flat mesh" | README:234 | **Structurally supported** by whitepaper connection table (full mesh O(n²) 9,120 conns vs hierarchical ~384 at scale; `Whitepaper:291-293`). Design/analytical claim, not a runtime benchmark. |
| "Priority 1 latency <5 s end-to-end" | README:232 | **Target, not validated.** QoS enforcement (TTL/bandwidth) is **in-progress** per roadmap (gbrain conversation note). Mark **unverified / in-flight**. |
| "Simulation validated 1,000+ nodes; 24-node lab" | README:233,340 | **Traceable to whitepaper:** single-machine validated to **1000 nodes** with a **1023 hard limit** (Linux bridge), lab experiments E11-E13 at 2-1000 nodes (`Whitepaper:13,163,213`). README's "24-node lab" aligns with the platoon-scale (24-node) figures. **Verified against whitepaper** (itself a single-machine simulation, not a field deployment). Note open epic **#724/#725/#726/#727** still targets 900/1.2K/10K validation — so "1,000+" is the *current* ceiling, not exceeded. |
| SBD ~1,960 B MO / ~1,890 B MT, 5-20 s, $0.04-0.13/msg | (constrained track / ADR-051) | **Unverifiable from this repo** — peat-sbd is proposal-only; 1,960 B is an Iridium SBD hardware limit cited in gbrain `quic-iridium-sbd-feasibility`, not a Peat measurement. |
| LoRa 7-87 km, 1.5-9.1 kB/s; BLE 100-400 m ~2 Mbps | (ADR-052 / ADR-039) | **Hardware-spec assumptions**, not Peat measurements. Mark as external hardware facts. |
| Automerge ~10 MB vs peat-lite 256 KB | — | peat-lite 256 KB RAM target is real (ADR-035, gbrain firmware/readme memory budget). Automerge "~10 MB" footprint is **unverified** in this repo. |

---

## 10 · ADR status table (requested set)

| ADR | Title | Status |
|---|---|---|
| 011 | CRDT+Networking (Ditto vs Automerge/Iroh) | **Proposed** |
| 032 | Pluggable transport abstraction | **Proposed** |
| 035 | Peat-Lite embedded nodes | **Proposed** |
| 039 | Peat-BTLE mesh transport | **Proposed** |
| 041 | Multi-transport embedded integration | **Accepted** |
| 046 | Targeted message delivery | **Proposed** |
| 051 | Peat-SBD satellite transport | **Proposed** |
| 052 | Peat-LoRa long-range radio | **Proposed** (ChaCha20 ref → FIPS conflict) |
| 059 | Cross-transport document bridging | **Proposed** |
| 060 | Encryption tiers (FIPS posture) | **Proposed** (but §5 already implemented in code) |
| 063 | Persistent multiplexed sync streams | **Proposed** (rc.25 floor cites ADR-063 / peat-mesh#175) |
| 066 | Abstract hierarchy vocabulary | **Proposed** (but `HierarchyLevel` enum already migrated) |

**Note:** Of the requested set, **only ADR-041 is Accepted.** The transport/CRDT/hierarchy/FIPS foundations are formally "Proposed" even where code already implements them — a recurring code-ahead-of-ADR pattern. Open issue **#695** ("Triage 22 Proposed ADRs before public release") and **#768/#769** (promote ADR-046/025 to Accepted) track the backlog.

---

## 11 · Open epics / issues (from `gh issue list -L 60`, HEAD `35d0f11`)

- **#853** Implement ADR-046 Targeted Message Delivery (epic) + phases **#854-#859**; duplicate tracker **#546**; spec issues **#770/#771/#772**.
- **#780** Epic: targeted binary distribution (capability-gated delivery) + **#774-#778**.
- **#904** epic: rename military-hierarchy terms → abstract vocabulary (workspace-wide); **#968** converge base unit on Node (ADR-068); **#970** ADR-068 follow-ups.
- **#935** ADR: persistent multiplexed sync streams (ADR-063 / peat-mesh#175); **#932** migrate formation_handshake to `&dyn QuicMeshConnection` (ADR-062 follow-up).
- **#724** EPIC Peat-Sim scaling validation 900/1.2K/10K (**#725/#726/#727**).
- **#833** disable n0 relay default; **#846** strip ATAK consumer-name refs; **#850/#829/#873** sync-reliability bugs; **#828** ADR-059 Amendment 4 cycle-break.
- **#592** Epic membership certificates/enrollment (**#588/#589/#590/#591**); **#547** ADR-045 Zarf/UDS; **#950** ADR-064 ARM64 CI; **#695** triage 22 Proposed ADRs.
- **#941** Authorization model deferred pending Layer-1 device identity (relevant to §4 RBAC).

(No epics literally numbered "#857 sync GC" / "#904 vocabulary rename" exactly as the prompt anticipated: #857 is ADR-046 Phase-4 *selectors*; #904 IS the vocabulary-rename epic. Logged.)

---

## 12 · Top gaps (contribution map seeds)

1. **README RBAC role names wrong** (Operator/Supervisor vs actual Commander/Admin) — doc fix, ~trivial. README.md:262.
2. **MLS presented as shipped but unimplemented** — relabel Proposed (ADR-044); or implement under a FIPS suite. Large.
3. **`command_log` CRDT does not exist** — any teaching material must mark Speculative; peat-lite has only Lww/G/PN/OrSet.
4. **peat-sbd / peat-lora are ADR-only** (051/052) — no crates; 052 carries a ChaCha20 FIPS conflict to fix before any impl.
5. **Hierarchy vocabulary mid-rename** — `HierarchyLevel` uses new terms and `peat-schema`'s hierarchy proto is already renamed (`CellSummary`/`CohortSummary`/`FederationSummary`/`CoalitionSummary`, no `Squad`/`squad_id`); only `peat-btle/src/lib.rs` is still fully legacy `Platform/Squad/Platoon/Company`; epics #904/#968 open.
6. **QoS enforcement (TTL/bandwidth, "<5s P1") in-flight**, not validated — ADR-019 framework present (`qos/` modules) but enforcement incomplete.
7. **"93-99%" bandwidth claim** not stated verbatim in the whitepaper (whitepaper: 79%/60-95%/95%+) — reconcile the number.
8. **peat-btle wire crypto** possibly ChaCha20 (gbrain note) — needs direct peat-btle audit against FIPS rule; if true, a shipped violation.
9. **ADRs formally Proposed while code ships** (011/035/039/060/063/066) — triage backlog #695.

---

## 13 · Assumptions / decisions logged during this audit

- peat-mesh/peat-btle/peat-lite source not in this tree → relied on re-export shims, their tests, manifest comments, and gbrain-indexed copies. Items so derived are tagged **(via re-export/gbrain)**.
- Did not `cargo build`/`cargo test` (read-only audit; would not change verdicts on type/status questions). Build-command verification deferred to the curriculum-run phase (module 08).
- "DeviceId" `code_def` in gbrain returned 0 (peat-mesh not indexed under the queried scope); confirmed instead via the re-export test assertions in `device_id.rs`.
- The prompt's anticipated issue numbers (#857 sync GC) did not match current reality; recorded actual mapping in §11 rather than asking.

---

## 2026-06-19 incremental delta (`35d0f11` → `68e9c3c`, rc.25 → rc.26)

Three commits:

- **`68e9c3c`** — workspace bumped `0.9.0-rc.25` → `0.9.0-rc.26`; peat-mesh dependency floor raised to
  `>=0.9.0-rc.43, <0.9.1` (`Cargo.toml:287`). peat-ffi unchanged at `0.2.7`.
- **`70f77a4`** (#993) — `peat-protocol/src/storage/{file_distribution,model_distribution}.rs` are now
  3-line re-exports of `peat_mesh::storage::…` (relocation per peat#992; peat-mesh is the canonical
  iroh consumer). The implementation no longer lives in the protocol/spec layer.
- **`72fc043`** (#991) — **ADR-071 (Proposed)** added (`docs/adr/071-subscription-based-convergence.md`,
  Status: Proposed, dated 2026-06-19) + the additive Phase-1 convergence **seam** (then relocated to
  peat-mesh by #993): `collection` on the distribution doc, `NeedEvaluator` + `CollectionSubscriptionNeed`,
  `should_deliver`/`can_skip_permanently`, opt-in via `with_need_evaluator` (default None — existing
  behavior unchanged). Unit + single-process receive-path tests only.

ADR-071 model: distribution moves from **sender-enumerated** (`resolve_targets` → `target_nodes`) to
**interest-driven convergence** — the writer publishes availability under a collection/topic and each
receiver locally evaluates a pluggable need predicate (subscription now; version-gap + capability are
Phases 2–3) and pulls if needed, locating a holder via provider gossip. Directed send (node-list) is
retained. **Proposed**; only the Phase-1 subscription seam exists in code, and it is inert by default.

---

### 2026-06-22 delta — `68e9c3c → 8a94796` (workspace 0.9.0-rc.26 → rc.27)

- **`relay-n0-hosted` facade fix (peat#995).** After the ADR-062 relocation moved `IrohTransport`
  endpoint construction into `peat-mesh`, `peat-protocol`'s `relay-n0-hosted` feature was an orphaned
  no-op (`= []`). rc.27 wires it to forward — `peat-protocol/Cargo.toml:123` →
  `["peat-mesh/relay-n0-hosted"]` — and adds the same passthrough on `peat-ffi`
  (`peat-ffi/Cargo.toml:109`). Still OFF by default. **Shipped.**
- **`peat-ffi` non-blocking `connect_peer_nowait` (peat#995).** Fire-and-forget variant of
  `connect_peer` — spawns dial + formation handshake + sync trigger on the runtime and returns
  immediately; shares `connect_peer_inner` with the blocking variant (`peat-ffi/src/lib.rs:813,1222`).
  Background failures surfaced via `tracing`. `peat-ffi` 0.2.7 → 0.2.8; Maven AAR cut 0.1.3
  (`peat-ffi/android/build.gradle.kts:33`). **Shipped.**
- **ADR-072 (Proposed): synced-folder lifecycle & file-handling policy** (`docs/adr/072-...md`).
  Publisher-declared lifecycle/handling policy on the ADR-071 distribution document (deletion,
  idempotent re-drop, version ordering). v1 unidirectional (a root is outbox OR inbox; no conflict
  management). **Status: Proposed — no code.**
- **ADR-059 (Proposed): "Data as a Capability"** (`docs/adr/059-data-as-a-capability.md`, peat#997).
  Extends the capability-advertisement pattern (ADR-018) to arbitrary data sources. **NUMBER
  COLLISION:** a second file `docs/adr/059-cross-transport-document-bridging.md` already owns ADR-059
  (the one the curriculum cites for the peat-btle cycle-break "Amendment 4"). The curriculum's
  existing ADR-059 references point at the cross-transport-bridging ADR and remain correct; the new
  data-as-a-capability ADR is logged but not woven into prose to avoid conflating the two. **Status:
  Proposed — no code.**
- No `peat-protocol/src`, `peat-mesh`, or `peat-schema` source change this delta — protocol/mesh
  diagrams' facts unaffected.

---

### 2026-06-29 delta — `8a94796 → 871776d` (workspace 0.9.0-rc.27 → rc.28)

All change is in `peat-ffi` (crate `0.2.8 → 0.2.9`; Maven AAR `0.1.3 → 0.1.4`; peat#1000). `peat-schema`
pin bumped in lockstep (`=0.9.0-rc.28`, `peat-protocol/Cargo.toml`). **No `peat-protocol/src`,
`peat-schema`, or `peat-mesh` source change** in this repo this delta — the new surface is additive on
the FFI binding only.

- **Persistent peer roster (peat#1000) [Shipped].** New `RosterStore` (JSON on disk, `peat-ffi/src/roster.rs:35,57`)
  tracks known group members across restarts. Six new UniFFI exports — `roster_remember`, `roster_upsert`,
  `roster_remove`, `roster_get`, `roster_list`, `roster_list_by_group` (`peat-ffi/src/lib.rs:2533-2569`) —
  plus a new `RosterEntry` UniFFI Record. Stored as **plain JSON, non-secret reachability hints only — no
  FIPS concern** (explicit in CHANGELOG).
- **Per-peer reconnect supervisor (peat#1000) [Shipped].** Dial state machine `Idle → Connecting →
  Connected → Backoff` with exponential backoff (`BACKOFF_BASE_MS=2_000`, `BACKOFF_MAX_MS=300_000` i.e.
  5 min, `peat-ffi/src/supervisor.rs:23,25`) plus deterministic per-peer jitter; concurrent in-flight
  dials bounded at `MAX_CONCURRENT_RECONNECT_DIALS = 8` via `tokio::sync::Semaphore`. Three new UniFFI
  exports — `reconnect_known_peers` (honours backoff), `wake_reconnect` (clears backoffs; call on
  foreground/network-up), `on_peer_observed` (`peat-ffi/src/lib.rs:2477,2490,2511`). **Cross-transport
  dedup:** the connected set is the union of iroh peers and any roster member with a live link on any
  transport (BLE, etc.), so a peer reachable over BLE is not re-dialed over iroh/relay.
- **Origin-tagged `DocumentChange` (peat#1000) [Shipped].** New required field `origin: ChangeOrigin` on
  the UniFFI `DocumentChange` Record; `ChangeOrigin` is a new UniFFI enum — `Local` / `Remote { peer_id }`
  (`peat-ffi/src/lib.rs:608`), converted from `peat_mesh::ChangeOrigin` at the FFI boundary
  (`lib.rs:618-622`). Lets consumers notify only on remote changes. **Coordinated binding regen** — paired
  with peat-flutter#13 (already in the audited peat-flutter HEAD).
- **Four Dart C-ABI shims (peat#1000) [Shipped].** Hand-rolled `#[no_mangle] extern "C"` reformatting flat
  `FFIBuffer` arrays for `roster_remember`, `reconnect_known_peers`, `wake_reconnect`, `on_peer_observed`
  (`peat-ffi/src/dart_ffi.rs`), alongside the existing shims.
- **FIPS posture unchanged.** The new code adds no crypto; roster persistence is plain JSON. No
  ChaCha20/X25519 reintroduced anywhere in source.

### 2026-07-06 delta — `871776d → 2778bd9` (workspace 0.9.0-rc.28 → **rc.29**; peat-ffi 0.2.9 → **0.2.10**)

Range = 11 commits (#1006/#1007 Android mDNS, #1016 mesh-pin drop, #1017 blob FFI, #1018 supply-chain,
#1019 release, #1020 tak-bridge move, #1021 ADR-073, + CI). **No enum/role/wire change** — CellRole still
= 7 (`peat-protocol/src/models/role.rs:14-30`); `cot/event.rs` change is `BytesText::unescape()` →
`xml10_content()` (behaviour-equivalent, forced by quick-xml 0.37→0.41, RUSTSEC-2026-0194/0195);
`sync/automerge.rs` +141 is the Android dial handler (additive). No crypto added.

- **UniFFI blob surface + poll-based `blob_fetch_start` (peat#1017) [Shipped].** New `#[uniffi::export]`
  methods on `PeatNode` (`peat-ffi/src/lib.rs:3358` on, `#[cfg(feature = "sync")]`): `enable_blob_transfer`
  (`:3369`, now takes `Option<String>` not `Option<SocketAddr>`), `blob_add_peer` (`:3410`),
  `blob_add_peer_id` (`:3442`, NEW — wraps peat-mesh `add_peer_from_hex_id`), `blob_put` (`:3468`),
  `blob_exists_locally` (`:3499`), `blob_endpoint_id` (`:3514`), `blob_bound_addr` (`:3522`), and
  **`blob_fetch_start(hash_hex, size_bytes, peer_id_hex?) -> Arc<BlobFetchHandle>`** (`:3541`). It is a
  *synchronous* fn that spawns a tokio task (`:3572`) and returns immediately; caller polls
  `BlobFetchHandle.status() -> BlobFetchStatus {Pending,Started,Downloading,Completed,Failed}` (`:3613,3690`)
  and cancels via `dispose()` (`:3700`, also on Drop). `Some(peer_id)` → `fetch_blob_from_peer` (direct);
  `None` → `fetch_blob` (mesh-sync). Dart C-ABI shims mirror all of it (`peat-ffi/src/dart_ffi.rs`).
  **NEEDS_RUNTIME** (handle/poll semantics confirmed; live transfer not benchmarked here).
- **Android mDNS: `bindAddress` + formation identity through JNI (peat#1006) [Shipped].**
  `normalize_bind_address` (`peat-ffi/src/lib.rs:9021`); `createNodeJni`/`createNodeWithConfigJni` gained a
  trailing nullable `bindAddress` (`:9061`, `:9175`) — **breaking JNI arity**. `create_node` derives the iroh
  identity via `IrohTransport::from_formation_with_discovery_at_addr` when a base64 shared key is present
  (`:1859`; legacy `from_seed_...` fallback `:1869`). `base64 = "0.22"` dep added (`peat-ffi/Cargo.toml:152`).
- **Dial mDNS-discovered peers on Android (peat#1007) [Shipped].** A peat-controlled `_peat._udp` browse
  consumer in `IrohPeerDiscovery` (`peat-protocol/src/sync/automerge.rs:2682` on): converts
  `DiscoveryEvent::PeerFound/PeerUpdated` → dialable `NetworkPeerInfo`, dials via `connect_peer`,
  authenticates via `respond_to_formation_auth`, with a 10 s reconnect-watchdog re-dial and `get_connection`
  dedup. Non-hex node_id skipped.
- **Dropped `[patch.crates-io]` git pin on peat-mesh; floor raised (peat#1016) [Shipped/manifest].** The
  workspace `Cargo.toml` git patch on `peat-mesh` was removed and Cargo.lock flipped to the registry crate.
  PR title says "bump floor to rc.44", but the **final HEAD floor is `>=0.9.0-rc.45, <0.9.1`**
  (`Cargo.toml:291`; re-bumped by the blob work #1017 to get `fetch_blob_from_peer`). Ground truth: floor
  **rc.45**, registry-sourced, no git pin. peat-schema pin bumped in lockstep to `=0.9.0-rc.29`
  (`peat-protocol/Cargo.toml:26`; `peat-ffi/Cargo.toml:140`).
- **ADR-073 (peer-ejection Rayfish review) [Proposed, no code].** New `docs/adr/073-peer-ejection-rayfish-review.md`,
  **Status: Proposed** (`:3`), dated 2026-07-07. A design review comparing "Rayfish" (an external open-source
  P2P mesh VPN on the same Iroh/QUIC substrate, used only as a comparison reference) against ADR-056
  (Compromised Node Detection/Isolation/Ejection); Decision = **no change to ADR-056**, with three follow-ups
  before Phase 1. No code. ADR total is now **79** (`ls docs/adr/*.md` = 79; 75 numbered + 4 reference).
  Accepted count **unchanged at 11** (002, 009, 015, 016, 023, 024, 030, 041, 047, 057, 070 — must match both
  `**Status:** Accepted` and `**Status**: Accepted` formats; a single-format grep under-counts to 9).
- **`examples/peat-tak-bridge` removed, moved to standalone `peat-tak` repo (peat#1020) [Shipped/move].**
  Deleted from the workspace (`Cargo.toml:17` comment); `peat-tak` is now consumed as a published crate
  (`=0.0.2`) by peat-sapient-bridge. **Stale refs persist** in `docs/whitepaper/05-...:156`,
  `docs/adr/070-...:65,484`, `docs/adr/058-...` (several), `DEPENDENCY-LICENSES.md:258` — those are peat-repo
  docs, not curriculum; curriculum links fixed in Modules 2 & 6.
- **FIPS posture unchanged.** No crypto added; supply-chain #1018 trusts `cmov`/RustCrypto-utils publishing.

*Could-not-confirm:* Maven AAR version for rc.29 — no AAR version bump found in this diff range (stale
references only). Logged, not asserted.

### 2026-07-13 delta — `2778bd9 → cb6a818` (workspace 0.9.0-rc.29 → rc.30; peat-ffi 0.2.10 → 0.2.11)
- **ADR-074 "peat-schema as the single source of truth" [Proposed; first tranche Shipped].**
  `docs/adr/074-schema-single-source-of-truth.md:3` `**Status**: Proposed`. The ADR wants every mesh
  message type to originate in peat-schema, with sub-crates deriving via generated `From<proto>`. A first
  migration step already shipped in code (peat#1022, `f767475`): the UniFFI records `NodeInfo`/`TrackInfo`/
  `CellInfo` were shrunk to proto-backed fields, `NodeStatus` folded into a proto-aligned `HealthStatus`,
  and the hand-rolled `MarkerInfo`/`CommandInfo` records + `get_markers()`/`get_commands()` accessors were
  **removed** from `peat-ffi/src/lib.rs`. Label precisely: governing ADR Proposed, first code step Shipped.
- **New proto types `Kinematics` + `PositionError` [Shipped].** `peat-schema/proto/common.proto:37,45`.
  `Kinematics{velocity, heading, acceleration, vertical_speed}`; `PositionError{circular_error(CEP),
  linear_error(LEP), vertical_error}`. Wired onto `Track` (fields 12/13) and `NodeState` (8/9, LWW merge in
  `peat-protocol/src/models/node.rs`); NaN/range validation in `peat-schema/src/validation/track.rs:122,158`.
  `Track.velocity`/`cep_m`/`vertical_error_m` now `[deprecated]` (but still populated by consumers — dual-write).
- **Proto schema version → 0.5.0 [Shipped].** Every `.proto` header now reads `Version: 0.5.0` (pre-1.0),
  a wire-schema version distinct from the peat-schema *crate* version (`0.9.0-rc.30`, `version.workspace = true`).
  CAP→Peat proto-comment rename (a9c2ac0) across 14 files. **Observed (not curriculum, not fixed here):** two
  residual all-caps `PEAT` in `security.proto:1` and `model.proto:5` (upstream source, house-rule drift there).
- **crossbeam-epoch 0.9.18 → 0.9.20 [Shipped]** (RUSTSEC-2026-0204, transitive; `Cargo.lock:1078`). No API impact.
- **ADR total 79 → 80** (`ls docs/adr/*.md` = 80; 76 numbered incl ADR-074 + 4 reference). **Accepted unchanged at 11**
  (002,009,015,016,023,024,030,041,047,057,070 — both `**Status:**`/`**Status**:` formats). CellRole still 7
  (`models/role.rs:13-29`). **FIPS posture unchanged** — no crypto touched anywhere in the diff.
- peat-ffi now exact-pins `peat-schema =0.9.0-rc.30` (fa5643c, `peat-ffi/Cargo.toml:147`), realizing ADR-074 Rule 5.

---

## Delta — 2026-07-20 (full sweep; `cb6a818` → `a1ce620`, rc.30 → rc.31; peat-ffi 0.2.11 → 0.2.12)

- **TAK transport REMOVED from `peat-transport` [Shipped].** peat#1015 (`492fb54`) deleted
  `peat-transport/src/tak/**` (2 insertions, 3861 deletions across 17 files: `tak/bridge/*`,
  `cot_translator.rs`, `server.rs`, `queue.rs`, `reconnect.rs`, `mesh.rs`, etc.). `peat-transport`
  now exposes only `error`, `http`, `lib` (`peat-transport/src/lib.rs`) — a read-only HTTP/REST query
  surface (`GET /api/v1/{nodes,cells,beacons}`). The TAK/CoT TCP bridge migrated to the standalone
  `peat-tak` repo. **CoT *translation* stays put** — `peat-protocol/src/cot/{mod,event}.rs` still
  present (`peat-protocol/src/lib.rs:81 pub mod cot;`, ADR-020/028).
- **peat-ffi 0.2.12: default-on `dart-ffi` feature gates the FFIBuffer adapter [Shipped].** peat#1031
  (`1a8cef7`). `peat-ffi/Cargo.toml`: `default = ["sync", "dart-ffi"]`; `peat-ffi/src/lib.rs:199`
  `#[cfg(feature = "dart-ffi")] pub mod dart_ffi;`. Direct consumers keep symbols by default; a native
  wrapper owning the adapter builds with `default-features = false` (see peat-flutter#29). UniFFI + JNI
  surfaces unchanged. peat#1030.
- **Codex migration.** `peat/CLAUDE.md` → `AGENTS.md` (`7630a9c`); `SKILL.md` → `.agents/skills/peat-ecosystem/SKILL.md`.
  FIPS hard-rule intact in `AGENTS.md:25-42` (AES-256-GCM; Ed25519/ECDSA-P256/384; ECDH-P256/384; X25519
  marginal/needs-review; HMAC-SHA-256; SHA-256/384/512; ChaCha20 = violation tracked ADR-060). Curriculum
  house-rule references to `peat/CLAUDE.md` should now read `peat/AGENTS.md`.
- **Test re-enables / lint** (no curriculum-facing surface): three-node mesh convergence re-enabled
  (`36ce5d1`, #829/#1027), transitive gossip lifecycle race fix (`a1ce620`, #1035), Dart shim Arc-handle
  lifecycle test (`152bfad`, #1028), Rust 1.97 Clippy map-iteration lint (`ce267c0`).
- **Stable anchors re-derived (full sweep):** `ls docs/adr/*.md` = **80** (76 numbered incl ADR-074; 4 reference);
  **11 Accepted** unchanged; ADR-074 still **Proposed** (`074-...:3`). `CellRole` still **7** (`peat-protocol/src/models/role.rs:14-29`).
  peat-mesh floor unchanged `>=0.9.0-rc.45` (`Cargo.toml:291`). **FIPS source posture unchanged** — no crypto in the diff.
