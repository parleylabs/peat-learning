# Ground truth — `peat-btle`

**Audited commit:** `3d70f48` (HEAD = origin/main, branch `main`, up to date with origin — `git fetch` showed 0 commits ahead/behind). Working tree had one unrelated modified file (`peat-btle-implementation-kickstart.md`); no pull/modification performed.
**Crate version:** `0.4.0` (`Cargo.toml:3`).
**Repo role:** BLE mesh transport for the Peat Protocol. Multi-platform Rust crate (`crate-type = ["rlib", "cdylib"]`, `Cargo.toml:14`) plus an Android AAR via UniFFI and an Apple FFI package under `ios/`.

Code is the source of truth throughout. Every claim below is cited to `path:line`. Labels: **Shipped** (in code, tested), **In flight** (open issue/PR), **Proposed** (ADR/aspiration, no code), **Speculative** (doc invented, not anywhere).

---

## 0 · Headline corrections (a skeptical reader will catch these)

1. **Crypto is NOT ChaCha20-Poly1305 anymore — it is AES-256-GCM + ECDH-P256.** The crate migrated 2026-05-18 to FIPS-approved primitives, but **`README.md` still documents ChaCha20-Poly1305 and X25519 as the live algorithms** (`README.md:196, 218, 242, 282`). This is the single most embarrassing live-doc/code drift. The actual code uses `aes-gcm` (`Aes256Gcm`) and `p256` (`Cargo.toml:108,112`; `src/security/mesh_key.rs:23-25,146,176`; `src/security/peer_key.rs:33`). All `chacha`/`x25519` strings remaining in `src/` are historical comments only (`grep` confirmed: `src/security/mesh_key.rs:21`, `peer_key.rs:24`, `document.rs:106`).
2. **The FIPS migration is itself only partly done — issue #75 is OPEN.** "FIPS compliance: migrate e2e encryption (src/security/*) to **aws-lc-rs**" (open, 2026-06-16). The current `aes-gcm`/`p256` crates are pure-Rust RustCrypto implementations — *FIPS-approved algorithms* but **not running inside a CMVP-validated cryptographic module**. For a defense-prime audience this distinction matters: algorithm choice is fixed (in flight → shipped), module validation is **In flight** (#75).
3. **The crypto swap is undocumented in CHANGELOG.md.** It appears only in `Cargo.toml` comments (`Cargo.toml:101-112`) and inline source comments. `CHANGELOG.md` `[Unreleased]` and `[0.4.0]` say nothing about the AES/P-256 swap (grep for `aes`/`fips`/`chacha` in CHANGELOG returns nothing).
4. **GATT service UUID is inconsistent across three places.** `README.md:187` advertises 16-bit `0xF47A`; the code constant is `PEAT_SERVICE_UUID_16BIT = 0xA1B2` derived from `PEAT_SERVICE_UUID = a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d` (`src/lib.rs:337,343`); and `src/gatt/mod.rs:23` doc-comments a *third* value `f47ac10b-58cc-4372-a567-0e02b2c3d479`. **Code of record = `a1b2c3d4…` / `0xA1B2`** (`src/gatt/service.rs:213` uses `PEAT_SERVICE_UUID`, and the test at `src/lib.rs:577-582` pins it).
5. **`NodeId` is a 32-bit `u32`, not a SHA-256 hash.** The prompt hypothesized "NodeId = SHA-256 of Ed25519 public key." Reality: `NodeId` wraps a `u32` (`src/lib.rs:367-371`). When derived from identity it is the **first 4 bytes of the BLAKE3 hash** of the Ed25519 public key, interpreted little-endian (`src/security/identity.rs:139-148`). 4 bytes = 32 bits → birthday-collision risk at ~77k nodes; relevant to the "1,000+ node" claim. It can also be derived non-cryptographically from a BLE MAC (`src/lib.rs:408-447`) or set to any literal.
6. **The hierarchy enum uses legacy military vocabulary, not ADR-066.** `HierarchyLevel` = `Platform / Squad / Platoon / Company` (`src/lib.rs:495-507`), **not** the ADR-066 Platform/Cell/Cohort/Federation/Coalition vocabulary. Material citing ADR-066 names against this crate is wrong for peat-btle.

---

## 1 · Transports actually implemented

`peat-btle` implements exactly **one** transport: **Bluetooth Low Energy**. There is no SBD, no LoRa, no QUIC here.

| Transport | Status | Evidence |
|---|---|---|
| **BLE (GATT + advertising/scanning)** | **Shipped** | `BluetoothLETransport` implements `MeshTransport` (ADR-032 abstraction) at `src/transport.rs:159-324`; `send_to` chunks by MTU and writes the sync-data characteristic (`src/transport.rs:298-319`). |
| Linux / BlueZ | **Shipped** | `src/platform/linux/` via `bluer` (`Cargo.toml:128`). |
| macOS / iOS (CoreBluetooth) | **Shipped (macOS) / partial (iOS)** | `src/platform/apple/` via `objc2-core-bluetooth` (`Cargo.toml:138`). README marks iOS "Beta — service discovery and characteristic callbacks incomplete" (`README.md:43`); a `TODO` confirms central-side gaps (`src/platform/apple/central.rs:209`). |
| Android (UniFFI AAR) | **Shipped** | `src/platform/android/`; `android` feature pulls `uniffi`+`translator-codec`+`peat-lite-frame` (`Cargo.toml:57`). |
| ESP32 (NimBLE) | **Shipped** | `src/platform/esp32.rs`; `esp32` feature (`Cargo.toml:72-82`); `examples/nano33-ble-sense/`. |
| Windows (WinRT) | **In flight / not validated** | `src/platform/windows/` exists (`adapter.rs`, `gatt_server.rs`); README "Planned — adapter exists, not yet tested" (`README.md:44`); a comment notes the GATT-server write path is not real (`src/platform/windows/gatt_server.rs:227`). |
| **peat-sbd (Iridium SBD)** | **Proposed — not in this repo** | Zero references anywhere in `peat-btle`. (grep for `peat-sbd`/`iridium` = none.) An ADR-051 proposal elsewhere; nothing here. |
| **peat-lora** | **Proposed — not in this repo** | Mentioned once, aspirationally, in a Cargo.toml comment ("…allows peat-btle, *future* peat-lora…", `Cargo.toml:171`). No code, no dep, no module. |

**Cross-transport bridging (ADR-059)** is partially shipped *as a codec*: the `translator-codec` feature provides `BleTranslator` typed structs, postcard scaffolding, 0xB6 frame marker, and JSON projections (`src/translator.rs`, gated at `src/lib.rs:194-195`). The `Translator` *trait impl* was deliberately **removed** from this crate in 0.4.0 (the `peat-mesh ⇄ peat-btle` back-edge break, ADR-059 Amendment 4 Slice 4.b) and now lives in peat-mesh's tree (`CHANGELOG.md:10-46`; `src/lib.rs:182-195`). peat-btle has **zero peat-mesh dependency** as of 0.4.0.

---

## 2 · CRDT types

All in `src/sync/crdt.rs`. These are hand-rolled, byte-packed CRDTs for `no_std`/BLE — not Automerge.

| Type | Semantics | Evidence |
|---|---|---|
| `LwwRegister<T>` | Last-writer-wins; tie broken by higher `NodeId` | `src/sync/crdt.rs:65-142` |
| `GCounter` | Grow-only counter, per-node `BTreeMap`, merge = component-wise max | `src/sync/crdt.rs:149-262` |
| `EmergencyEvent` | OR-set-style ACK map (monotonic false→true) keyed by `(source_node, timestamp)`; different events resolve by higher timestamp | `src/sync/crdt.rs:676-894` |
| `Peripheral` | Composite record (callsign/health/event/location) with `timestamp` | `src/sync/crdt.rs:901-1113` |
| `Position`, `HealthStatus` | LWW-wrapped value records (validated decoders) | `src/sync/crdt.rs:266-516` |
| `ChatCRDT` / `ChatMessage` | Add-only set, bounded to 32 msgs; **`legacy-chat` feature, deprecated, slated for removal** | `src/sync/crdt.rs:1120-1560`; `Cargo.toml:49-50` |
| `VectorClock` | Causality vector for delta-sync negotiation | `src/sync/delta.rs:264-298` |

**Confirmed absent:** no `PnCounter`, no standalone `OrSet` type (OR-set semantics are inlined in `EmergencyEvent` only; `src/translator.rs:148,163` references "OR-set" purely as a comment describing ACK-map semantics), and **no `command_log` CRDT** (grep: none). This matches the prompt's expectation that `command_log` does not exist. Note: peat-btle's primitive set (`LwwRegister`, `GCounter`, `EmergencyEvent`, `Peripheral`) is **not** identical to peat-lite's documented set — they are sibling crates with overlapping but distinct primitives.

---

## 3 · Identity & addressing

- `NodeId` — 32-bit `u32` (`src/lib.rs:367-371`). Derivations: literal `new(u32)`; hex `parse` (`:385`); from BLE MAC last-4-bytes (`:408`); from MAC string (`:435`); **cryptographic** = first 4 bytes of BLAKE3(ed25519_pubkey) LE (`src/security/identity.rs:139-148`).
- `DeviceIdentity` — Ed25519 signing keypair (`ed25519-dalek` 2.1, `Cargo.toml:115`); `generate()`, `sign()/verify()`, `public_key()` (`src/security/identity.rs:85-201`).
- `IdentityAttestation` — `node_id(4) ‖ public_key(32) ‖ timestamp_ms(8) ‖ signature(64)`, Ed25519-signed (`src/security/identity.rs:182-234`; README table `README.md:361-366`).
- `IdentityRegistry` — TOFU (trust-on-first-use) registry (`src/security/registry.rs`; re-exported `src/lib.rs:294-297`).
- `MeshGenesis` / `MeshCredentials` / `MembershipPolicy{Open,Controlled,Strict}` — mesh bootstrap with 256-bit CSPRNG seed, HKDF-derived encryption secret + beacon key (`src/security/genesis.rs`; README `:368-407`).
- `MembershipToken` — tactical-trust token (`src/security/membership_token.rs`).

---

## 4 · Hierarchy enum

`HierarchyLevel { Platform=0, Squad=1, Platoon=2, Company=3 }` (`src/lib.rs:495-525`). **Legacy military vocabulary.** This is the only hierarchy type in the crate and it does **not** track ADR-066 (Platform/Cell/Cohort/Federation/Coalition). Curriculum must flag this when mapping peat-btle to the umbrella vocabulary.

---

## 5 · Crypto primitives actually used (FIPS posture)

| Use | Primitive | Evidence | FIPS posture |
|---|---|---|---|
| Mesh-wide AEAD | **AES-256-GCM** (NIST SP 800-38D) | `src/security/mesh_key.rs:23-25,146,176` | FIPS-approved algorithm |
| Per-peer key exchange | **ECDH on NIST P-256** (SP 800-56A) | `src/security/peer_key.rs:33,89-143` | FIPS-approved |
| Per-peer AEAD | **AES-256-GCM** | `src/security/peer_session.rs:31-33` | FIPS-approved |
| Key derivation | **HKDF-SHA256** | `src/security/mesh_key.rs:124-128` | FIPS-approved |
| Signing / identity | **Ed25519** | `src/security/identity.rs:49,157` | Ed25519 is in FIPS 186-5; CMVP coverage uneven — note for compliance |
| NodeId hash | **BLAKE3** | `src/security/identity.rs:141` | **NOT a FIPS-approved hash.** Used only for NodeId derivation (non-secret addressing), not for any security boundary — but worth flagging. |
| Mesh-wide overhead | 30 bytes (2 marker + 12 nonce + 16 tag) | `README.md:220` | — |
| Per-peer E2EE overhead | README says 46 bytes (`README.md:298-304`); `src/security/mod.rs:71` doc says 44 bytes | conflicting docs — verify against `PeerEncryptedMessage::encode` | unverified-number |

**FIPS verdict vs ADR-060:** The crate has *already moved off* ChaCha20-Poly1305/X25519 to AES-256-GCM/P-256 to satisfy ADR-060 §5 (per `Cargo.toml:101-112`). So the old "ChaCha20 is a live ADR-060 violation" framing is **outdated for the code** — but **still true for `README.md`**, which advertises the non-FIPS primitives. Remaining real FIPS gap: no CMVP-validated module yet (issue #75, In flight); BLAKE3 for NodeId (cosmetic, non-security).

---

## 6 · Public API a consumer integrates against

Re-exports at `src/lib.rs:236-332`. Three integration surfaces:

1. **Transport** — `BluetoothLETransport` + `MeshTransport` trait (`start/stop/connect/disconnect/send_to/...`), `BleConfig`, `PowerProfile`, platform adapters (`BluerAdapter`, `AndroidAdapter`, `CoreBluetoothAdapter`, `WinRtBleAdapter`, `MockBleAdapter`) gated by feature (`src/lib.rs:251-268`, `transport.rs:211-253`).
2. **`PeatMesh` / `PeatMeshConfig`** — the high-level mesh object (std-only, `src/lib.rs:284-285`). Builder config: `new(node_id, callsign, mesh_id)`, `with_encryption`, `with_strict_encryption`, `with_relay`, `with_max_relay_hops`, `with_relay_fanout`, `with_peripheral_type`, `with_sync_interval`, `with_max_peers` (`src/peat_mesh.rs:162-256`). Runtime API includes identity/attestation (`create_attestation`, `verify_peer_identity`), mesh-wide encryption, per-peer E2EE (`enable_peer_e2ee`, `initiate_peer_e2ee`, `send_peer_e2ee`, `handle_*`), multi-hop relay (`wrap_for_relay`, `process_relay_envelope`, `get_relay_targets`, seen-cache), delta-sync (`build_delta_document_for_peer`, `build_full_delta_document`), app-document registry (`store_app_document`, `get_app_document`), persistence (`to_persisted_state`, `from_persisted`), and the ADR-059 decoded-document JSON callback (`set_decoded_document_json_callback`) (`src/peat_mesh.rs:402-2122`).
3. **UniFFI surface** — `src/uniffi_bindings.rs` (2685 lines), `uniffi` feature, generates Kotlin + Swift; `DecodedDocumentJsonCallback` callback interface (`src/lib.rs:219-226`).

Document/wire markers (`src/document.rs:51-175`): `EXTENDED 0xAB`, `EMERGENCY 0xAC`, `CHAT 0xAD`, `ENCRYPTED 0xAE`, `PEER_E2EE 0xAF`, `KEY_EXCHANGE 0xB0`, `RELAY_ENVELOPE 0xB1`, `DELTA_DOCUMENT 0xB2`, `TRANSLATOR_FRAME 0xB6`, reserved `0xB7–0xBF`.

---

## 7 · Advertised capabilities — shipped vs proposed vs speculative

| Capability | Status | Evidence / note |
|---|---|---|
| BLE transport (`MeshTransport`) | **Shipped** | `src/transport.rs` |
| Multi-hop relay (TTL, seen-cache, fanout) | **Shipped** | `src/relay.rs:54-60` (`DEFAULT_MAX_HOPS=7`, marker `0xB1`, seen-TTL 300_000 ms); `src/peat_mesh.rs:1137-1317` |
| Mesh-wide encryption (AES-256-GCM) | **Shipped** | `src/security/mesh_key.rs` |
| Per-peer E2EE (ECDH-P256 + AES-256-GCM, replay counter) | **Shipped** | `src/security/peer_session.rs`, `peer_key.rs` |
| Ed25519 device identity + TOFU registry + attestation | **Shipped** | `src/security/identity.rs`, `registry.rs` |
| Delta/CRDT sync over GATT, MTU chunking + reassembly | **Shipped** | `src/sync/`, `transport.rs:298-319` |
| Power profiles | **Shipped (3 of 4)** | `PowerProfile{Aggressive,Balanced,LowPower}` (`src/power/profile.rs:77-88`). **`UltraLow` (0.5%/2min/~36h) listed in `README.md:177` and absent from the enum — doc-only / Speculative.** |
| Coded PHY (long range) | **Shipped (config + capability), hardware-dependent** | `BlePhy::LeCodedS8`, `coded-phy` feature (`Cargo.toml:85`); `TransportCapabilities` 400 m / 125 kB/s (`transport.rs:76-107`). Range is a spec assumption, not a measured PEAT number. |
| ADR-059 translator codec (0xB6 frames) | **Shipped (codec only; trait impl moved to peat-mesh)** | `src/translator.rs`; `CHANGELOG.md:10-46` |
| `peat-lite-frame` Document carrier | **Shipped (optional feature)** | `Cargo.toml:47,174` depends on `peat-lite 0.2.5` |
| Windows transport | **In flight** | code present, untested (`README.md:44`) |
| iOS CoreBluetooth full support | **In flight** | "Beta", incomplete callbacks (`README.md:43`; `apple/central.rs:209`) |
| CMVP/FIPS-validated crypto module (aws-lc-rs) | **In flight** | issue #75 (open) |
| Public chat-send (originate, not just relay) | **In flight** | issue #26 (open) |
| Reconnect re-delivery of pending CRDT state | **In flight** | issue #73 (open) |
| `peer_link_info` on production adapters | **In flight** | issue #45 (open, ADR-032 Amendment A) |
| peat-sbd / peat-lora transports | **Proposed (elsewhere) / not in repo** | see §1 |
| Leader election / consensus | **Not in this crate** | none in `peat-btle`; lives in peat-mesh. Relay peer selection uses gossip strategies, not election (`src/peat_mesh.rs:312`, `src/gossip.rs`). |

---

## 8 · Quantitative claims — verifiable vs not

**Verifiable in code:**
- BLE base capabilities: 250 kB/s, 30 ms latency, 100 m range, 512-byte max message (`src/transport.rs:61-72`); Coded S=8: 125 kB/s, 400 m (`:76-88`); Le2M: 500 kB/s, 50 m (`:94-99`). These are **declared capability constants**, not measured.
- Relay defaults: 7 max hops, 300 s seen-cache TTL (`src/relay.rs:57-60`).
- Power-profile duty cycle is **computed** from scan/adv/conn timing (`src/power/profile.rs:39-55`), not hardcoded; the enum doc-comments label Aggressive≈20%, Balanced≈10%, LowPower≈2% (`:77-88`).
- Crypto overheads: mesh-wide 30 B (`README.md:220`) consistent with 2+12+16.
- 723 test functions across `src/` + `tests/` (8 integration test files incl. `encryption_benchmark.rs`, `mesh_sync.rs`, `peat_lite_uniffi_e2e.rs`).

**Unverifiable / doc-only (flag for chase):**
- Watch-battery hours (~6/12/20/36 h) — `README.md:172-179` estimates "for typical smartwatch (300 mAh)"; no in-repo measurement; UltraLow 36 h has no code path at all.
- "18–24 hour battery life" and "3–4 hour" competitor figure (`README.md:31`) — marketing, uncited.
- "300 m+ / 400 m" range — BLE-spec/Coded-PHY assumption, hardware-dependent, no PEAT field data here (range-test scaffolding exists: `docs/RANGE_TEST_NOTES.md`, `examples/range_test_node*.rs`, but no published numbers in repo).
- "Automerge ~10 MB vs peat-btle <256 KB RAM" (`README.md:137`) — architectural assertion, no benchmark in this repo.
- "<5% radio duty cycle" / "20%+ for commercial SDKs" comparison table (`README.md:28-31`) — uncited.
- Per-peer E2EE overhead: README 46 B vs `src/security/mod.rs:71` doc 44 B — internal contradiction; verify against `PeerEncryptedMessage::encode`.

---

## 9 · ADRs present in this repo (local, not umbrella numbering)

`docs/adr/` contains **repo-local** ADRs, NOT the peat umbrella ADR-039/051/052/059/060/066 set:
`001-hive-lite-primitives.md`, `03-peat-mesh-app-architecture-v2.md`, `04-mobile-desktop-architecture.md`, `ADR-001-trust-architecture.md`, `ADR-002-mesh-provisioning.md`, `ADR-003-extensible-document-registry.md`. The umbrella ADR numbers (032, 059, 060) are referenced only in code comments/doc-strings, not as files here. No `ROADMAP.md`. CHANGELOG is current through `[0.4.0] - 2026-05-06` with an empty `[Unreleased]`.

---

## 10 · Assumptions logged (decisions made without asking)

- **Code-of-record for the service UUID = `a1b2c3d4…`/`0xA1B2`** (`src/lib.rs:337,343`), because that is the constant the GATT service and its test actually use; the README `0xF47A` and the `gatt/mod.rs` `f47ac10b…` are stale doc artifacts.
- **Crypto = AES-256-GCM/P-256/Ed25519/BLAKE3/HKDF-SHA256** treated as ground truth over README's ChaCha20 text, because the manifest deps and all live call sites use the former; README is stale.
- **iOS/Windows treated as In-flight** rather than Shipped based on README status labels corroborated by in-code TODOs.
- Battery/range/RAM figures treated as **doc estimates** (unverifiable) absent any in-repo measurement harness output.
- Did not run `cargo build/test` (read-only audit sufficient to establish reality; 723 test fns observed but not executed).
