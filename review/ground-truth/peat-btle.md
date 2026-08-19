# Ground truth — `peat-btle`

**Audited commit:** `654db7b` (HEAD = origin/main; advanced from `bcfa954` in the 2026-07-27 incremental — 6 commits, headlined by the AWS-LC crypto migration #81).
**Crate version:** `0.4.0` (`Cargo.toml:3`).
**Repo role:** BLE mesh transport for the Peat Protocol. Multi-platform Rust crate (`crate-type = ["rlib", "cdylib"]`, `Cargo.toml:14`) plus an Android AAR via UniFFI and an Apple FFI package under `ios/`.

Code is the source of truth throughout. Every claim below is cited to `path:line`. Labels: **Shipped** (in code, tested), **In flight** (open issue/PR), **Proposed** (ADR/aspiration, no code), **Speculative** (doc invented, not anywhere).

---

## 0 · Headline corrections (a skeptical reader will catch these)

1. **Crypto is NOT ChaCha20-Poly1305 anymore — it is AES-256-GCM + ECDH-P256, now through `aws-lc-rs`.** The crate migrated 2026-05-18 to FIPS-approved primitives (RustCrypto `aes-gcm`/`p256`), then on **2026-07-23 (#81, tracked by #75; commit `35cf716`)** routed **all** AEAD/ECDH/HKDF/HMAC/hash through **`aws-lc-rs 1.17`** and removed the direct `aes-gcm`/`p256`/`hkdf`/`sha2`/`blake3` deps. Code uses `aws_lc_rs::aead::AES_256_GCM` + `agreement::ECDH_P256` + `hkdf`/`hmac`/`digest::SHA256` (`Cargo.toml:111-112`; `src/security/mesh_key.rs:21-23,121`; `src/security/peer_key.rs:28-29,55`; `src/security/identity.rs:49`). `README.md` **still documents ChaCha20-Poly1305 and X25519** — a live doc-bug drift, not source behaviour.
2. **The FIPS migration shipped in source but was never re-published to crates.io — the source is now two migrations ahead of the published crate.** Source at HEAD `654db7b` is FIPS-approved through AWS-LC (`Cargo.toml:111-112`), still version `0.4.0 / [Unreleased]`. The **crates.io-published `peat-btle 0.4.0`** (checksum `a57dd351…`) **still depends on `chacha20poly1305` + `x25519-dalek`**: same version string; neither the 2026-05-18 RustCrypto swap nor the 2026-07-23 AWS-LC migration was re-published. peat-flutter's lockfile pins exactly that published artifact (`peat-flutter/rust/Cargo.lock:3498-3531,631,6402`). **Provider posture:** default build (`default = ["std","aws-lc-non-fips"]`) links the regular AWS-LC provider (`aws-lc-sys`) — approved algorithms, not the validated module — while the mutually exclusive **`fips` feature** links `aws-lc-rs-fips` = `aws-lc-fips-sys`, the **CMVP-validated AWS-LC FIPS module** (`Cargo.toml:17,25,26,111,112`). Issue **#75 now exists** and is cited by peat ADR-060 §5.
3. **The crypto migration IS now documented in CHANGELOG.md.** `[Unreleased]` records the BREAKING change: AES-256-GCM/ECDH-P256/HKDF-SHA256/HMAC-SHA256/SHA-256 via `aws-lc-rs`; removed RustCrypto AEAD/ECDH/KDF and BLAKE3; NodeId/mesh-id/genesis/beacon derivation moved to SHA-256/HMAC/HKDF; explicit crypto version bytes (mesh docs v2, E2EE v2, encrypted beacons v3); ECDH-P256 keys as 65-byte uncompressed SEC1; earlier wire formats rejected (mixed-version meshes unsupported); Apple XCFramework now targets iOS 13+.
4. **GATT service UUID is inconsistent across three places.** `README.md:187` advertises 16-bit `0xF47A`; the code constant is `PEAT_SERVICE_UUID_16BIT = 0xA1B2` derived from `PEAT_SERVICE_UUID = a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d` (`src/lib.rs:337,343`); and `src/gatt/mod.rs:23` doc-comments a *third* value `f47ac10b-58cc-4372-a567-0e02b2c3d479`. **Code of record = `a1b2c3d4…` / `0xA1B2`** (`src/gatt/service.rs:213` uses `PEAT_SERVICE_UUID`, and the test at `src/lib.rs:577-582` pins it).
5. **`NodeId` is a 32-bit `u32`, not a full-width hash — and as of 2026-07-23 the hash is SHA-256, not BLAKE3.** `NodeId` wraps a `u32` (`src/lib.rs:367-371`). When derived from identity it is the **first 4 bytes of SHA-256(Ed25519 public key)**, interpreted little-endian (`src/security/identity.rs:314-315`; the AWS-LC migration #81 replaced the previous BLAKE3 derivation). 4 bytes = 32 bits → birthday-collision risk at ~77k nodes; relevant to the "1,000+ node" claim. It can also be derived non-cryptographically from a BLE MAC (`src/lib.rs:408-447`) or set to any literal.
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
| Provider | **`aws-lc-rs 1.17`** (default `aws-lc-sys`; `fips` feature → `aws-lc-fips-sys`) | `Cargo.toml:17,25,26,111,112` | FIPS-approved algorithms; the `fips` feature links the **CMVP-validated AWS-LC FIPS module** |
| Mesh-wide AEAD | **AES-256-GCM** (NIST SP 800-38D) | `src/security/mesh_key.rs:21-23,121` | FIPS-approved algorithm |
| Per-peer key exchange | **ECDH on NIST P-256** (SP 800-56A) | `src/security/peer_key.rs:28-29,55` | FIPS-approved |
| Per-peer AEAD | **AES-256-GCM** | `src/security/peer_session.rs:31-33` | FIPS-approved |
| Key derivation | **HKDF-SHA256** | `src/security/mesh_key.rs:121`; `src/security/genesis.rs:159-179` | FIPS-approved |
| Signing / identity | **Ed25519** | `src/security/identity.rs:157` (`ed25519-dalek`, `Cargo.toml:116`) | Ed25519 is in FIPS 186-5; CMVP coverage uneven — note for compliance |
| NodeId / mesh-id hash | **SHA-256 / HMAC-SHA256** | `src/security/identity.rs:49,314-315`; `src/security/genesis.rs:141-147` | **FIPS-approved hash** (moved off BLAKE3 in the 2026-07-23 AWS-LC migration). NodeId = first 4 bytes of SHA-256(pubkey) — addressing-only, not a security boundary. |
| Mesh-wide overhead | 30 bytes (2 marker + 12 nonce + 16 tag) | `README.md:220` | — |
| Per-peer E2EE overhead | README says 46 bytes (`README.md:298-304`); `src/security/mod.rs:71` doc says 44 bytes | conflicting docs — verify against `PeerEncryptedMessage::encode` | unverified-number |

**FIPS verdict vs ADR-060:** The crate's *source* satisfies ADR-060 §5 and has gone further — as of 2026-07-23 (#81/#75) all crypto routes through `aws-lc-rs`, and the `fips` feature links the CMVP-validated AWS-LC FIPS module, so a validated FIPS 140 boundary is now reachable in the crate itself (opt-in). BLAKE3 is gone (NodeId now SHA-256). The old "ChaCha20 is a live ADR-060 violation" framing remains **true two ways**: (1) `README.md` still advertises the non-FIPS primitives, and (2) the **crates.io-published `0.4.0` (checksum `a57dd351…`) still ships `chacha20poly1305`/`x25519-dalek`** — the source is now two migrations ahead but neither was re-published, so consumers pinning `0.4.0` get the non-FIPS crypto (`peat-flutter/rust/Cargo.lock:3498-3531,631,6402`). Remaining real FIPS gaps: the un-republished crate, and that the *default* build uses the non-FIPS AWS-LC provider (a `--features fips` build is required to claim the validated module).

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
| Coded PHY (long range) | **Shipped (config + capability), hardware-dependent** | `BlePhy::LeCodedS8`, `coded-phy` feature (`Cargo.toml:85`); `TransportCapabilities` 400 m / 125 kB/s (`transport.rs:76-107`). Range is a spec assumption, not a measured Peat number. |
| ADR-059 translator codec (0xB6 frames) | **Shipped (codec only; trait impl moved to peat-mesh)** | `src/translator.rs`; `CHANGELOG.md:10-46` |
| `peat-lite-frame` Document carrier | **Shipped (optional feature)** | `Cargo.toml:47,174` depends on `peat-lite 0.2.5` |
| Windows transport | **In flight** | code present, untested (`README.md:44`) |
| iOS CoreBluetooth full support | **In flight** | "Beta", incomplete callbacks (`README.md:43`; `apple/central.rs:209`) |
| CMVP/FIPS-validated crypto module | **Available opt-in (Shipped, #75/#81)** | the `fips` feature links `aws-lc-fips-sys` (the CMVP-validated AWS-LC FIPS module); the default build uses the non-FIPS AWS-LC provider (`aws-lc-sys`) — `Cargo.toml:17,25,26,111,112` |
| Re-publish FIPS source to crates.io | **Still an open gap** | source `654db7b` routes crypto through `aws-lc-rs` but published `0.4.0` (`a57dd351…`) still ships `chacha20poly1305`/`x25519-dalek` (`peat-flutter/rust/Cargo.lock:3498-3531`) — two source migrations un-republished |
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
- "300 m+ / 400 m" range — BLE-spec/Coded-PHY assumption, hardware-dependent, no Peat field data here (range-test scaffolding exists: `docs/RANGE_TEST_NOTES.md`, `examples/range_test_node*.rs`, but no published numbers in repo).
- "Automerge ~10 MB vs peat-btle <256 KB RAM" (`README.md:137`) — architectural assertion, no benchmark in this repo.
- "<5% radio duty cycle" / "20%+ for commercial SDKs" comparison table (`README.md:28-31`) — uncited.
- Per-peer E2EE overhead: README 46 B vs `src/security/mod.rs:71` doc 44 B — internal contradiction; verify against `PeerEncryptedMessage::encode`.

---

## 9 · ADRs present in this repo (local, not umbrella numbering)

`docs/adr/` contains **repo-local** ADRs, NOT the peat umbrella ADR-039/051/052/059/060/066 set:
`001-hive-lite-primitives.md`, `03-peat-mesh-app-architecture-v2.md`, `04-mobile-desktop-architecture.md`, `ADR-001-trust-architecture.md`, `ADR-002-mesh-provisioning.md`, `ADR-003-extensible-document-registry.md`. The umbrella ADR numbers (032, 059, 060) are referenced only in code comments/doc-strings, not as files here. No `ROADMAP.md`. CHANGELOG now carries a populated `[Unreleased]` section (the 2026-07-23 AWS-LC crypto migration + Android/Apple adapter link-state fixes) above `[0.4.0] - 2026-05-06`.

---

## 10 · Assumptions logged (decisions made without asking)

- **Code-of-record for the service UUID = `a1b2c3d4…`/`0xA1B2`** (`src/lib.rs:337,343`), because that is the constant the GATT service and its test actually use; the README `0xF47A` and the `gatt/mod.rs` `f47ac10b…` are stale doc artifacts.
- **Crypto = AES-256-GCM/P-256/Ed25519/BLAKE3/HKDF-SHA256** treated as ground truth over README's ChaCha20 text, because the manifest deps and all live call sites use the former; README is stale.
- **iOS/Windows treated as In-flight** rather than Shipped based on README status labels corroborated by in-code TODOs.
- Battery/range/RAM figures treated as **doc estimates** (unverifiable) absent any in-repo measurement harness output.
- Did not run `cargo build/test` (read-only audit sufficient to establish reality; 723 test fns observed but not executed).

## Delta — 2026-07-27 (incremental; `bcfa954` → `654db7b`, 0.4.0 source, first delta since first audit)

Six commits, headlined by the **AWS-LC crypto migration (#81, tracked by #75; `35cf716`, 2026-07-23)** —
see §0 headline corrections and §5 above, all rewritten this run. Summary:
- **All crypto now routes through `aws-lc-rs 1.17`** (AES-256-GCM, ECDH-P256, HKDF/HMAC-SHA-256, SHA-256);
  direct `aes-gcm`/`p256`/`hkdf`/`sha2`/`blake3` deps removed (`Cargo.toml:111-112`). Default feature
  `aws-lc-non-fips` → `aws-lc-sys`; mutually exclusive **`fips` feature → `aws-lc-fips-sys` = the
  CMVP-validated AWS-LC FIPS module** (`Cargo.toml:17,25,26`).
- **BLAKE3 removed.** NodeId = first 4 bytes of SHA-256(pubkey) (`identity.rs:314-315`); mesh-id =
  HMAC-SHA-256 of mesh name; secrets via HKDF-SHA-256 (`genesis.rs:141-179`).
- **Clean wire + identity cutover (not backward compatible):** mesh docs `MESH_ENCRYPTION_VERSION=2`
  (`mesh_key.rs:27`), per-peer `E2EE_PROTOCOL_VERSION=2` (`peer_key.rs:36`), `ENCRYPTED_BEACON_VERSION=0x03`
  (`encrypted_beacon.rs:69`); `KeyExchangeMessage` 37 → **71 bytes** (`ENCODED_LEN = 1+4+1+65`,
  `peer_key.rs:236,251`), ECDH keys as 65-byte uncompressed SEC1 (`ECDH_PUBLIC_KEY_SIZE=65`, `peer_key.rs:42`).
- **Also this run:** production BLE link-state reporting + generic Android adapter link state (#84/#86),
  state redelivery after reconnect (#83), Apple XCFramework now targets iOS 13+ (AWS-LC requirement).
- **Published-vs-source gap persists and widens:** crates.io `0.4.0` (`a57dd351…`) still ships
  ChaCha20/X25519; source is now two migrations ahead, never re-published
  (`peat-flutter/rust/Cargo.lock:3498-3531`).
- **NEEDS_RUNTIME:** BLE-rate/range constants unchanged (declared, not measured); no `cargo build/test` run.

## Delta — 2026-08-03 (incremental; `654db7b` → `7ae0ecc`, 0.4.0 `[Unreleased]`)

One commit (#89), **Android/Kotlin only** — no Rust source, no crypto primitive, no published artifact
change; crate still `0.4.0` with the new API under `[Unreleased]`.
- **Public `sendChat` API [Shipped + tested]** — `fun sendChat(chat: PeatChat): Boolean`
  (`android/.../PeatBtle.kt:2399`) and a `fun sendChat(sender, message)` convenience overload (`:2437`,
  fills `timestamp`/`originNode`). Returns `false` if the mesh isn't running; otherwise encodes the chat,
  seeds the relay-dedup set (so the originator's own message doesn't bounce back), writes to every connected
  peripheral, and notifies connected centrals. Unit-tested in `PeatBtleSendChatTest.kt` (both overloads,
  mesh-not-running guards, wire-framing `0xAD` marker, convenience field-fill); multi-hop relay deferred to
  on-device validation.
- **Crypto boundary [flag, not a regression]:** `sendChat` uses the **unencrypted direct-GATT write path**
  (raw `0xAD` chat document), explicitly contrasted in code with `broadcastBytes` (mesh-encrypted). So chat
  sent this way crosses BLE **without application-layer mesh encryption** — documented intent, a plaintext
  local-chat path. Distinct from the AES-256-GCM mesh crypto.
- **Published-vs-source non-FIPS split UNCHANGED** — this commit alters neither the crates.io artifact nor
  the source crypto (Cargo.toml still declares `aws-lc-rs` non-fips default + optional `aws-lc-rs-fips`).
  FIPS source posture unchanged.

## Delta — 2026-08-17 (incremental; `2946c62` → `8d9d247`, 0.4.0 `[Unreleased]`)

One fix (PR #91, commit `cfe464d`): **stamp `peripheral_id` on the anonymous-tracks receive path**.
Two files touched — `src/peat_mesh.rs` (+34/-10) and a new `tests/anonymous_tracks_peripheral_id.rs`.
No UniFFI surface change, no crypto change.

- **The bug it fixes.** A `tracks` document keys on the sender's `peripheral_id` — its doc id is
  `ble-<HEX8>` (`src/translator.rs:302`, prefix `:79`) — and `tracks` is the one collection whose decoder
  requires `peripheral_id` on `DecodeInboundCtx` (`src/peat_mesh.rs:709-712`). The anonymous receive
  bridge `on_ble_data_received_anonymous` (`:2879`) previously passed `None` unconditionally, so every
  inbound `tracks` frame **Err'd** while other collections kept decoding — meaning a BLE-only peer could
  transmit position (PLI) that no peer could receive, silently breaking the BLE→QUIC position bridge.
- **The fix [Shipped].** It resolves the sender through `PeerManager`'s existing
  `identifier → NodeId` index: `self.peer_manager.get_node_id(identifier)` (`:2950`), passed into
  `try_handle_translator_marker` (`:2952-2956`) which stamps it onto the context
  (`peripheral_id: source_node.map(|n| n.as_u32())`, `:778`) before `decode_inbound_sync`; the resolved
  sender is surfaced on `DataReceivedResult` instead of the old `NodeId(0)` placeholder (`:2964-2967`).
- **Fail-loud contract preserved.** An *unknown* identifier still passes `None`, and the translator
  **Errs** rather than defaulting to `ble-00000000` — collapsing unknown senders onto one id would merge
  every such peer into a single phantom track (rationale `:2944-2948`). Pinned by
  `anonymous_tracks_frame_from_an_unknown_peer_does_not_fabricate_an_id` (`tests/…:162`), plus a
  non-`tracks` control asserting the anonymous path was not narrowed.
- **FIPS / published-vs-source split unchanged.** Source still routes all crypto through `aws-lc-rs 1.17`
  (`Cargo.toml:111-112`; AES-256-GCM/ECDH-P256/HKDF-SHA256/HMAC-SHA256/SHA-256, no RustCrypto/BLAKE3);
  the `fips` feature links CMVP-validated `aws-lc-fips-sys`. The crates.io-published `peat-btle 0.4.0`
  (the build peat-flutter pulls) still ships non-FIPS ChaCha20-Poly1305 + X25519 — source is two
  migrations ahead but neither was re-published.
- **Version:** unchanged `0.4.0`; CHANGELOG top is still `[Unreleased]` (the fix landed as a commit and
  was not separately itemized under Unreleased).
