# Ground truth — `peat-lite`

**Repo:** `peat-lite` (subdirectory `./peat-lite` of the PEAT umbrella)
**Audited at HEAD:** `7a8a8fb` — *refactor: rename hierarchy tiers to abstract vocabulary (ADR-066) (#28)*
**Prior commit:** `dee6616` — chore: bump version to 0.2.5 (#27)
**Branch:** `main`. `git fetch` ran clean; working tree NOT modified. `git rev-list --left-right --count origin/main...HEAD` = `0 0` — local main is exactly even with origin/main (origin not ahead).
**Crate version:** `0.2.5` (Cargo.toml:12), edition 2021, MSRV 1.70.

> Audit method: code read line-by-line; `gh issue list -L 40` returned empty for this remote (no issues surfaced via the configured `gh` auth/remote — logged as an assumption, not a finding). No `CHANGELOG.md` or `ROADMAP.md` exists in the repo (confirmed by `ls`). Status of every capability is anchored to a `path:line`.

---

## 1. What this repo actually is

`peat-lite` is the **lightweight CRDT + wire-protocol building block** for resource-constrained PEAT nodes. It is `no_std`-capable (default feature `std`; disable for embedded) with a single runtime dependency: `heapless` 0.8 for fixed-capacity collections (Cargo.toml:38). It is **not** a transport, a mesh, or a networking stack — it is a codec + bounded data-structure library that other crates (peat-mesh, peat-btle, firmware) consume.

The repo is a Cargo workspace (Cargo.toml:1-8) with members `.` (core) and `android-ffi`. **Workspace-excluded** (own toolchain / Cargo.toml): `firmware/` (ESP32), `android/` (Gradle), `fuzz/` (cargo-fuzz).

Four build targets:
- **Core Rust library** (`src/`, `crate-type = ["rlib"]`, Cargo.toml:28).
- **Android FFI** (`android-ffi/`, UniFFI-bound, `android-ffi/src/lib.rs`).
- **ESP32 firmware** (`firmware/`, separate `cargo +esp` toolchain).
- **Fuzz harness** (`fuzz/`, 5 targets, requires nightly).

---

## 2. Transports — SHIPPED vs PROPOSED

**Headline: `peat-lite` implements NO network transport itself.** It defines a *wire format* and *constants* that transports use. The actual sending/receiving lives in firmware (UDP/WiFi) and in sibling repos (peat-mesh, peat-btle).

| Transport | Status in *this repo* | Evidence |
|---|---|---|
| **UDP datagram framing** (the Peat-Lite binary protocol, ADR-035) | **Shipped** as a codec | `src/protocol/header.rs`, `src/protocol/constants.rs:13` `DEFAULT_PORT = 5555`, `MULTICAST_ADDR = 239.255.72.76` (constants.rs:16), `MAX_PACKET_SIZE = 512` "fits in a single UDP datagram" (constants.rs:22). The *codec* is shipped; peat-lite does not open sockets. |
| **WiFi/UDP on ESP32** (actual socket I/O) | **Shipped in firmware** (separate crate, gated) | `firmware/src/wifi_main.rs`; `firmware/Cargo.toml:84-91` `wifi` feature pulls `esp-radio`, `smoltcp` (socket-udp), `blocking-network-stack` (a `git` dependency, Cargo.toml:50). Not part of the published library; requires ESP32 toolchain. |
| **BLE** | **Not in this repo** — lives in `peat-btle` | peat-lite only references BLE as a *NodeId derivation hint* in a doc comment (`src/node_id.rs:11` "Derived from BLE MAC address"). SKILL.md routes BLE to `peat-btle/SKILL.md`. |
| **peat-sbd (Iridium SBD, ADR-051)** | **Not present** — ADR proposal only | No crate, no module, no symbol. The Document envelope doc-comment mentions "future LoRa transports" (document.rs:5) but ships nothing. |
| **peat-lora (ADR-052)** | **Not present** — ADR proposal only | Same. `document.rs:59-61` references "LoRa-class transports carrying envelopes" as a *design intent* for the transport-agnostic envelope; no LoRa code exists here. |

The `protocol::document` module is explicitly engineered to be **transport-agnostic** so a future LoRa/SBD transport can carry it without codec changes (document.rs:1-15, 66-71) — but that is forward-looking design, not a shipped transport.

---

## 3. CRDT types — the reality

There are **two distinct CRDT surfaces** in this repo, which the curriculum must not conflate:

### 3a. Core library CRDTs (`src/`, no_std, published crate)
| Type | Status | Evidence |
|---|---|---|
| `LwwRegister<T>` | **Shipped** | `src/lww.rs:33`. LWW with deterministic tiebreak: higher timestamp wins; ties broken by higher `node_id.as_u32()` (`src/lww.rs:96-99`). |
| `GCounter<const MAX_NODES = 32>` | **Shipped** | `src/counter.rs:38`. Grow-only; merge = per-node max (counter.rs:92-104); `increment` saturating (counter.rs:73). |
| `CannedMessageStore<const MAX_ENTRIES = 256>` | **Shipped** (LWW-per-(source,type)) | `src/canned.rs:750`. LWW semantics with oldest-eviction when full (canned.rs:773-789). |
| `CannedMessageAckEvent` | **Shipped** (hybrid CRDT) | `src/canned.rs:432`. OR-set ACK map merge for same event; LWW (higher timestamp wins) across different events (canned.rs:532-555). ACK map bounded at `MAX_CANNED_ACKS = 64` (canned.rs:16). |
| `Position` | Shipped helper (not a CRDT) | `src/lww.rs:125`. Fixed-point lat/lon microdegrees + alt cm; encodes to 12 bytes. Intended payload for `LwwRegister`. |

**`PnCounter` and `OrSet` are NOT implemented in the core library `src/`.** They appear **only as wire-type identifiers** in the `CrdtType` enum (`src/protocol/crdt_type.rs:6-11`: `LwwRegister=0x01, GCounter=0x02, PnCounter=0x03, OrSet=0x04`). The core crate re-exports `GCounter` and `LwwRegister` at the root (`src/lib.rs:74-75`) but exports **no** `PnCounter` or `OrSet` struct. `default_ttl_for_crdt` matches all four enum variants (ttl.rs:48-55) but that is a TTL lookup table over the wire-type byte, not a CRDT implementation.

> The §4 contract's check "peat-lite has only `LwwRegister`, `GCounter`, `PnCounter`, `OrSet`" needs nuance: as **implemented core CRDTs**, the library has only `LwwRegister` and `GCounter` (+ the canned-message hybrid CRDTs). `PnCounter`/`OrSet` exist as **wire-protocol type codes** (CrdtType enum) and `PnCounter` is implemented **only in the firmware crate** (see 3b). `OrSet` has **no struct implementation anywhere** in this repo — it is a reserved wire byte only.

The contract's other check — "confirm `command_log` CRDT does NOT exist" — **confirmed**: no `command_log`, `CommandLog`, or command-log type appears anywhere in the repo.

### 3b. Firmware CRDTs (`firmware/src/crdt/`, workspace-excluded, NOT published)
| Type | Status | Evidence |
|---|---|---|
| `GCounter` | Shipped in firmware (separate impl) | `firmware/src/crdt/g_counter.rs` |
| `LwwRegister` | Shipped in firmware (separate impl) | `firmware/src/crdt/lww_register.rs` |
| `PnCounter` | **Shipped in firmware only** | `firmware/src/crdt/pn_counter.rs:14` — two internal G-Counters (increments − decrements), implements a local `LiteCrdt` trait. This is a *separate* implementation from the published `peat-lite` library and is **not** what a Rust consumer of the `peat-lite` crate gets. |

This firmware/library duplication is a notable structural fact: the firmware does **not** reuse the published library's CRDT structs for PnCounter; it has its own `LiteCrdt` trait + types. `OrSet` is not implemented in firmware either.

---

## 4. Identity / addressing — `NodeId`

**`NodeId` is a 32-bit integer (`u32`), NOT a SHA-256 of an Ed25519 public key.** `src/node_id.rs:15` — `pub struct NodeId(u32)`, `#[repr(transparent)]`, 4 bytes.

- Derivation per the doc comment: "Derived from BLE MAC address or assigned during provisioning" (node_id.rs:9-12). There is **no key-derivation code** in this repo — `NodeId::new(u32)` and `from_le_bytes`/`from_be_bytes` are the only constructors (node_id.rs:20-34).
- "Collision probability is acceptable for tactical mesh sizes (<1000 nodes)" (node_id.rs:11) — a stated design assumption, not a guarantee.
- `NodeId::NULL = 0` is the sentinel (node_id.rs:61); `is_null()` checks for 0.

> **Correction for the curriculum:** The `NodeId = SHA-256(Ed25519 pubkey)` model and `btle_to_peat_node_id` bridging are **peat-mesh / peat-btle** concepts. In `peat-lite`, `NodeId` is a bare 32-bit value with no cryptographic derivation. Any material that says peat-lite's `NodeId` is a hashed public key is **wrong** at this HEAD. The 32-bit↔larger-id bridging happens in the host-side crates, not here.

There is **no hierarchy enum** in `peat-lite`. The ADR-066 hierarchy vocabulary (Platform/Cell/Cohort/Federation/Coalition) referenced by commit `7a8a8fb` lives in the umbrella/peat repos; the rename commit touched naming but `peat-lite` defines no tier enum. (Confirmed: no `Hierarchy`, `Tier`, `Cell`, `Cohort`, `Federation`, `Coalition` enum in `src/`.)

---

## 5. Crypto primitives — FIPS posture

**`peat-lite` uses only FIPS-approved primitives. There is NO ChaCha20-Poly1305 or AES anywhere in this repo.** This is a clean repo against the ADR-060 FIPS-only rule.

| Primitive | Where | FIPS status |
|---|---|---|
| Ed25519 (signature **carrier**, not signer) | `src/wire.rs:17` `SIGNATURE_SIZE = 64`; `src/canned.rs:306-361` `encode_signed`/`decode_signed`/`decode_auto`. The core library **only carries** a 64-byte signature; it does **not** sign or verify — "Caller is responsible for computing the signature" (canned.rs:316-317) and "This does NOT verify the signature" (canned.rs:343). | Ed25519 is FIPS-approved (FIPS 186-5). |
| Ed25519 verification (actual) | `firmware/src/ota.rs:183-194` uses `ed25519_dalek::{Signature, Verifier, VerifyingKey}` to verify the OTA offer signature over the SHA-256 digest. Compile-time / optional via `ota` feature (`firmware/Cargo.toml:107-112`). | FIPS-approved. |
| SHA-256 | `firmware/src/ota.rs` (`sha2::Sha256`, streaming hash of firmware image, ota.rs:529-589); also the `sha256: [u8;32]` field in OtaOffer (ota.rs:75). `sha2` 0.10 in `firmware/Cargo.toml:62`. | FIPS 180-4 approved. |

> **No FIPS violation in `peat-lite`.** The ChaCha20-Poly1305 concern from the §4 contract belongs to **peat-btle / ADR-052**, NOT this repo. If the curriculum flags a FIPS gap for peat-lite, that is a misattribution. The only crypto deps here (`ed25519-dalek`, `sha2`) are FIPS-approved and live exclusively in the firmware crate's `ota` feature.

The `DOC_FLAG_ENCRYPTED` bit (document.rs:94) is **reserved and rejected by the encoder** today (document.rs:228-230) — there is no per-document encryption layer implemented. The Document envelope is plaintext-on-the-wire; encryption is a future layer, explicitly not wired (document.rs:84-93).

---

## 6. Wire protocol — shipped surface

The canonical Peat-Lite binary protocol (ADR-035) is **shipped** in `src/protocol/`:

- **16-byte fixed header** (`src/protocol/header.rs:5-10`, `HEADER_SIZE = 16` constants.rs:19): `MAGIC "PEAT"` (4B, constants.rs:4) + version (1B, `PROTOCOL_VERSION = 1` constants.rs:7) + type (1B) + flags (2B LE) + node_id (4B LE) + seq_num (4B LE). Decode validates magic, version, message type (header.rs:45-67).
- **Message types** (`src/protocol/message_type.rs:13-49`, `#[non_exhaustive]`): `Announce=0x01, Heartbeat=0x02, Data=0x03, Query=0x04, Ack=0x05, Leave=0x06, Document=0x07, OtaOffer=0x10 … OtaAbort=0x16`.
  - **README discrepancy (minor):** README.md:59 lists "Message types: Announce, Heartbeat, Data, Query, Ack, Leave, OTA (0x10-0x16)" and **omits `Document` (0x07)**, which IS implemented and re-exported. The README's "OTA (0x10-0x16)" also collapses 7 distinct OTA variants. Flag as outdated README.
- **CrdtType byte** (crdt_type.rs): LwwRegister/GCounter/PnCounter/OrSet (see §3a caveat).
- **`Document` universal envelope** (`src/protocol/document.rs`) — **shipped, heavily tested** (collection + doc_id + timestamp_ms + opaque body, length-prefixed; tombstone flag; NUL-byte rejection; UTF-8 validation; reserved-flag rejection). Implements the "universal" half of ADR-059 Amendment 4 (document.rs:9-15). Max field sizes: collection 255B, doc_id 65535B, body 65535B (document.rs:112-119).
- **TTL suffix** (`src/protocol/ttl.rs`): when `FLAG_HAS_TTL` (0x0001) set, last 4 bytes of payload = u32 LE TTL seconds. Defaults: LWW 300s, GCounter 3600s, PnCounter 3600s, OrSet never-expires (constants.rs:39-48, ttl.rs:48-55).
- **NodeCapabilities** (`src/protocol/capabilities.rs`): 16-bit flag set for Full/Lite negotiation. `lite()` = PRIMITIVE_CRDT|SENSOR_INPUT (caps.rs:43-45); `full()` = storage|relay|doc-crdt|prim-crdt|blob|history|agg (caps.rs:48-58). `can_sync_with` requires both sides have PRIMITIVE_CRDT (caps.rs:91-93).
- **OTA wire constants** (`src/protocol/ota.rs`): chunk data 448B (= 496 payload − 6 framing, ota.rs:7), offer v1 76B / v2 140B (signed), result + abort codes. The OTA *logic* (state machine, flash, rollback) is in `firmware/src/ota.rs`, not the library.
- **CannedMessage codec** (`src/canned.rs`, `src/wire.rs`): marker `0xAF` (wire.rs:14), unsigned event 22B, signed event 86B (= 22 + 64 sig). ACK event 24B base + 12B per ACK, max 24 + 64×12 = 792B (wire.rs:32, canned.rs:561).

---

## 7. Public API a consumer integrates against

Re-exported at the crate root (`src/lib.rs:71-88`):
- **Types:** `NodeId`, `CannedMessage`, `CannedMessageEvent`, `CannedMessageAckEvent`, `CannedMessageStore`, `GCounter`, `LwwRegister`.
- **Protocol:** `MessageType`, `CrdtType`, `Header`, `NodeCapabilities`, `MessageError`, `WireError`.
- **Functions:** `encode_header`, `decode_header`, `append_ttl`, `strip_ttl`, `default_ttl_for_crdt`.
- **Constants:** all of `protocol::constants::*` (MAGIC, DEFAULT_PORT, HEADER_SIZE, MAX_PACKET_SIZE, TTL defaults, flags) plus wire sizes (CANNED_MESSAGE_*_SIZE, SIGNATURE_SIZE, CANNED_ACK_EVENT_MAX_SIZE).
- **Document envelope:** accessed via `peat_lite::protocol::document::{encode, decode, DocumentRef, …}` (re-exported through `protocol` mod, mod.rs:20-23) — NOT at the bare crate root.

**Android consumers** integrate against the UniFFI surface in `android-ffi/src/lib.rs` (free functions like `encode_canned_message_event`, `decode_canned_message_ack_event`, `canned_message_ack_event_merge`). The FFI surface is **canned-message-only** — it does NOT expose GCounter, LwwRegister, the header codec, or the Document envelope to Kotlin.

Features (Cargo.toml:30-33): `default = ["std"]`, `std`, and no_std via `--no-default-features`. The lib.rs doc comment (lib.rs:30) mentions an `android` feature, but **no `android` feature is declared in the core Cargo.toml** — the Android cdylib is a *separate workspace member* (`android-ffi`), not a feature of the core crate. Flag as stale doc comment.

---

## 8. Capabilities: shipped / proposed / speculative

**Shipped (in code, tested):**
- 16-byte header codec; CrdtType byte; MessageType enum incl. `Document`.
- `LwwRegister`, `GCounter` core CRDTs; `CannedMessageStore` (LWW); `CannedMessageAckEvent` (OR-set + LWW hybrid).
- Universal `Document` envelope (encode/decode, tombstone, UTF-8/NUL/flag validation) — extensively unit + fuzz tested.
- TTL suffix codec; NodeCapabilities Full/Lite negotiation.
- Canned-message wire format (unsigned 22B / signed 86B / ACK), signature *carriage* (not verification).
- OTA wire constants (library) + full OTA state machine with SHA-256 streaming verify, Ed25519 signature verify, A/B partition rollback (firmware, `ota` feature).
- Android UniFFI bindings for canned messages.
- 5 cargo-fuzz targets (`fuzz/fuzz_targets/`).
- Firmware `PnCounter` (firmware crate only).

**Proposed / not implemented:**
- Per-document encryption (`DOC_FLAG_ENCRYPTED` reserved + encoder-rejected, document.rs:84-94).
- `OrSet` CRDT — wire byte `0x04` reserved, no struct anywhere.
- `PnCounter` in the *published library* (only the wire byte + firmware impl exist).
- SBD (ADR-051) and LoRa (ADR-052) transports — ADR proposals; the Document envelope is *designed for* them but ships no transport.

**Speculative (teaching-only, not in repo):** anti-entropy / digest / version-vector / IBLT sync schemes — **none exist in peat-lite**. This crate has no sync engine, no anti-entropy, no leader election. Merge is pairwise CRDT merge on whatever the caller hands it. Any sync/anti-entropy claim attributed to peat-lite is speculative for this repo (that machinery, if anywhere, is peat-mesh).

---

## 9. Quantitative claims — verifiable vs not

**Verifiable from code at this HEAD:**
- Wire sizes: header 16B (constants.rs:19); unsigned canned msg 22B, signed 86B, sig 64B (wire.rs:22-27, asserted canned.rs:1342-1348); ACK event 24B base + 12B/ACK, max 792B (wire.rs:32, canned.rs:561); Document overhead 14B fixed + fields (document.rs:178, `30 + N + M + K` framed incl. header, document.rs:64).
- `MAX_PACKET_SIZE = 512`, `MAX_PAYLOAD_SIZE = 496` (= 512 − 16) (constants.rs:22-25); OTA chunk data 448B (ota.rs:7).
- Field caps: collection 255B, doc_id 65535B, body 65535B (document.rs:112-119).
- TTL defaults: LWW 300s, GCounter/PnCounter 3600s (constants.rs:39-45).
- `DEFAULT_PORT = 5555`, multicast `239.255.72.76` (constants.rs:13-16).
- Memory figures (README.md:17-25): NodeId 4B (✓ `u32`), CannedMessage 1B (✓ `#[repr(u8)]`), GCounter "~4 bytes per node" / default 32 nodes ≈ 260B (counter.rs:17-18), CannedMessageStore 256 entries ≈ 6KB (canned.rs:748-749). These are stated estimates consistent with the type definitions; not independently benchmarked.
- Test counts: **108** core `#[test]` (src + tests), **7** android-ffi, **47** firmware. 5 fuzz targets. (Counted via grep; not executed in this audit — `cargo test` not run.)

**NOT verifiable in this repo (must be sourced elsewhere or flagged):**
- "256KB of RAM" budget (README.md:7,13; lib.rs:21) — a *design target / marketing figure*, not enforced or measured by any code here. No allocator cap, no static-RAM assertion.
- BLE range/throughput (100-400m, ~2 Mbps), LoRa range/rate (7-87km, 1.5-9.1 kB/s), SBD message sizes (~1,960B MO / ~1,890B MT), $/msg, latency (<5s P1), "93-99% bandwidth reduction", "Automerge ~10 MB vs peat-lite 256 KB", "O(n log n)", "1,000+ node validation" — **none of these appear in peat-lite code.** They are ecosystem/curriculum claims that this repo neither makes nor substantiates. peat-lite contains zero Automerge code and zero benchmark harness for these numbers.
- The `<1000 nodes` collision-acceptability claim (node_id.rs:11) is a stated design assumption, not a tested bound.

---

## 10. Assumptions / decisions logged during audit

1. **`gh issue list -L 40` returned empty** for the configured remote. Decision: proceeded without issue cross-referencing rather than blocking; treat epic numbers (#853/#857/#904) as umbrella-repo issues not resolvable from this repo's `gh` context. No issue-status claims made.
2. **`cargo test` not executed** (would require building the full toolchain; audit is read-based per the "do not modify working tree" instruction beyond `git fetch`). Test *counts* are from static grep of `#[test]`; pass/fail not asserted here.
3. **Firmware vs library CRDT duplication** treated as a real structural fact worth surfacing, not a bug to reconcile.
4. Treated the lib.rs `android` feature mention and README's missing `Document` message type as **doc-staleness findings** (code is source of truth), not protocol changes.

---

## 11. Top gaps for the curriculum to get right

1. peat-lite's `NodeId` is a **bare `u32`**, not a hashed Ed25519 key. The crypto-identity model is peat-mesh's, not peat-lite's.
2. peat-lite **has no transport, no sync engine, no anti-entropy, no leader election**. It is a codec + bounded CRDTs. SBD/LoRa are ADR proposals; the Document envelope is merely *designed to be carriable* by them.
3. Implemented core CRDTs are **only `LwwRegister` + `GCounter`** (plus canned-message hybrid CRDTs). `PnCounter` is firmware-only; `OrSet` is a reserved wire byte with no implementation anywhere.
4. **No FIPS violation here** — only Ed25519 + SHA-256 (firmware `ota` feature). The ChaCha20-Poly1305 FIPS concern is peat-btle/ADR-052, not peat-lite. Do not misattribute it.
5. The `Document` (0x07) message type and the whole universal-envelope codec are **shipped and well-tested** but **missing from the README**'s message-type list. The README is the stale artifact, not the code.
