# Ground-truth audit — `peat-flutter`

**Repo:** `github.com/defenseunicorns/peat-flutter` · **Audited HEAD:** `369f05d` (2026-06-24,
"feat: reconnect-supervisor bindings + facade (putRaw, peerTransportStates, origin decode,
enableN0Relay fix) #13") · **Version:** `0.0.1` (`pubspec.yaml:3`; pre-1.0, API still evolving).
First tracked 2026-06-25.

## What it is
The **Flutter/Dart binding layer** for Peat: a Flutter FFI plugin wrapping the published `peat-ffi`
Rust crate (UniFFI). A Flutter app calls `PeatFlutterNode.initialize()` then
`PeatFlutterNode.create(NodeConfig(...))` to run a node that discovers peers (mDNS/BLE/iroh relay)
and syncs Automerge/CRDT docs device-to-device with no central server. ~10.5k LOC Dart + a 13-line
Rust cdylib (`rust/src/lib.rs`) that only `uniffi_reexport_scaffolding!()`s peat-ffi's symbols into
the bundled native lib. Pattern mirrors `defenseunicorns/peat-swift`.

## Structure
- `lib/src/peat_node.dart` (443 ln) — **the public hand-written API**: class `PeatFlutterNode`
  (lifecycle, discovery, `putCell`/`cells`, `putCommand`/`commands`, `connectPeer`/`connectPeerNowait`,
  reconnect supervisor `rememberPeer`/`reconnectKnownPeers`/`wakeReconnect`/`onPeerObserved`,
  `startSync`/`syncStats`, raw+proto doc put/get, `subscribeChanges` 50 ms poll Stream,
  BLE frame bus `startOutboundFrames`/`ingestInboundFrame`, CRDT counter+KV).
- `lib/src/generated/peat_ffi.dart` (9,966 ln) — UniFFI surface, **maintained BY HAND** (the
  `uniffi-bindgen-dart` generator is not public; reconciled via `just check-bindings`). Carries
  `bindingsContractVersion = 30` + per-function FFI checksums verified at load.
- `rust/` cdylib wrapper; `example/` = "peat-water" P2P counter demo (3,647 ln); `test/` = **one**
  file (`change_origin_test.dart`, JSON round-trips only — binary FFI decode path explicitly NOT
  covered). `lib/src/proto/` is **empty** (`.gitkeep`).

## Relationship to other Peat repos
Binds **`peat-ffi =0.2.9` from crates.io** via **UniFFI 0.28** (`rust/Cargo.toml:28`, confirmed in
`rust/Cargo.lock`) — not a local `peat` checkout, and **not** peat-node gRPC. Transitively (peat-ffi
`bluetooth`,`lite-bridge`,`sync` features) the bundled `libpeat_ffi` pulls in **peat-btle 0.4.0** and
**peat-mesh** (iroh/Automerge). It is a *distribution vehicle* for the Rust stack, not new protocol.

## Capabilities
- **Shipped** — node lifecycle, mDNS/iroh discovery, CRDT doc sync, proto-typed + raw doc put/get
  (`lib/src/peat_node.dart:84-298`); contract-version + checksum integrity guard at lib load
  (`peat_ffi.dart:3706-3725`); CRDT counter+KV over Automerge-over-BLE (`peat_node.dart:403-426`);
  reconnect supervisor + `connectPeerNowait` (`peat_node.dart:181-236`, HEAD `369f05d`); iroh n0
  public-relay toggle `TransportConfigFFI.enableN0Relay` (default `false`, `peat_ffi.dart:1473`).
- **In-flight** — unbounded `commands` Automerge doc growth, acknowledged unresolved
  (`docs/crdt-counter-over-ble.md:118-147`); test coverage thin (native FFI decode, two-device
  relay acceptance, Android-NDK build all explicitly not yet run — `ONBOARDING.md`).
- **Proposed** — generalize CRDT-sync into peat-mesh with per-peer capability negotiation
  (`docs/crdt-counter-over-ble.md:105-107`).

## Crypto / FIPS posture — ⚠ FLAG
peat-flutter writes **no crypto of its own**. But the shipped `libpeat_ffi` it bundles pulls (via the
pinned `bluetooth` feature → **peat-btle 0.4.0**, `rust/Cargo.lock`) **`chacha20poly1305` +
`x25519-dalek`** — **non-FIPS** primitives, at the BLE link layer, in every Flutter consumer's
artifact. `aes-gcm`/`p256`/`ring`/`rustls`/`hmac`/`sha2` (FIPS-approvable set) coexist for the
QUIC/iroh path. **This contradicts the "code already migrated off ChaCha20/X25519" narrative, which
held for peat-mesh (rc.12) — the BLE transport in peat-btle 0.4.0 evidently did NOT migrate.**
→ **VERIFIED against `peat-btle` source:** the source DID migrate to AES-256-GCM/ECDH-P256 (commit
`c8b013e`), but the **crates.io-published `0.4.0`** this lockfile pins was never re-published, so the
shipped artifact is still non-FIPS. There is **no `peat-btle#75` and no `aws-lc-rs` migration** (the
`aws-lc-rs` in this lock is the TLS stack, not BLE crypto); `peat-current-state` §6 must be corrected
to the published-vs-source split. The demo hardcodes an all-zeros `sharedKey`
(`example/lib/main.dart:433`) — demo-only.

## README-vs-code drift
Empty `lib/src/proto/` vs README:36-39 "committed proto stubs"; pin `=0.2.9` vs CHANGELOG "0.2.8"
(header still says 0.2.5); `ffigen` regen instructions (README:113-122) contradict the
hand-maintained-bindings note (README:40-43); pubspec "wrapping peat-schema/protocol/mesh/btle"
implies multi-crate binding but it binds only peat-ffi.

## Curriculum mapping
**New sub-section under `04-peat-btle-and-lite`** — suggested **"04c — Client bindings
(peat-flutter / peat-swift)"** (the BLE transport + CRDT-over-BLE demo are its core). Cross-link
`02-peat-protocol` (consumes the peat-ffi UniFFI boundary) and `06-data-flows` (the subscribeChanges
poll + BLE frame bus + CRDT merge are concrete flows). The non-FIPS BLE flag feeds
`07-repo-links-and-gaps` + `09-protocol-specs`. The peat-water example is an `08-running-and-operating`
walkthrough candidate. **Note: `peat-swift` exists too** — a sibling client-binding repo to consider.

## Unverifiable / flagged
- Native runtime semantics rest on the **hand-maintained** `peat_ffi.dart` + contract/checksum guard,
  not on peat-ffi source (not in this repo).
- No green-CI evidence beyond a self-hosted arm64 native-build job; iOS-sim/Android-NDK flagged unverified.
- The single test covers only JSON codecs; cross-peer sync untested in-repo.
- BLE non-FIPS flag is **confirmed** from `Cargo.lock`; the QUIC-path FIPS runtime config is
  **unverifiable from this repo** (needs peat-mesh/peat-ffi source).

---

### 2026-06-29 delta — `369f05d → 4a6554f` (still `0.0.1`)

Single commit, peat-flutter#17 ("reconnect-supervisor bindings + relay URL fix"). Touches
`lib/src/peat_node.dart` (+8) and `rust/Cargo.lock` (transitive minor bumps only). **No pubspec version
change.**

- **Reconnect-supervisor facade [Shipped].** The Dart facade now exposes `reconnectKnownPeers()`,
  `wakeReconnect()`, `onPeerObserved(nodeId)` (`lib/src/peat_node.dart:228,234` + generated
  `lib/src/generated/peat_ffi.dart:9392,9400,9408`), thin wrappers over the peat-ffi `0.2.9` reconnect
  surface (peat#1000). The generated bindings were regenerated against the new UniFFI contract.
- **Relay URL fix [Shipped].** `endpointAddr` getter now returns the iroh relay `https://` URL (or `'—'`
  when not yet established) instead of the old endpoint-address form, with an `isEmpty` guard
  (`lib/src/peat_node.dart:89-98`).
- **FIPS flag UNCHANGED.** `rust/Cargo.lock` still pins **`peat-btle 0.4.0`** (the crates.io artifact
  bundling **`chacha20poly1305 0.10.1` + `x25519-dalek 2.0.1`**), `peat-ffi 0.2.9`, `peat-mesh
  0.9.0-rc.43` — none of these lines moved in `369f05d..4a6554f`. The published-vs-source non-FIPS split
  (C1) **persists**: peat-btle *source* is FIPS-clean, the *published* 0.4.0 that peat-flutter builds is
  not. The only lockfile churn was unrelated transitive minors (libc, etc.).

### 2026-07-13 delta — `291d7e4 → 129c74c` (v0.0.1; re-pins published peat-ffi 0.2.9 → 0.2.10)
- **Blob client surface [Shipped]** (peat-flutter#20, `982bea6`). `rust/Cargo.toml:28` bumps
  `peat-ffi =0.2.10` (published crate, checksum `df75222c…`). `PeatFlutterNode.blobDownload(hashHex, sizeBytes,
  {peerIdHex})` returns a poll-based `Stream<BlobFetchStatus>` (`lib/src/peat_node.dart:451`); omit the peer →
  mesh-sync candidate selection, pass it → direct P2P pull, no fallback. Supporting facades: `enableBlobTransfer`,
  `blobAddPeer(Id)`, `blobPut`, `blobExistsLocally`, `blobEndpointId`, `blobBoundAddr`. Hand-maintained (not
  generated) UniFFI bindings; `BlobFetchStatus` mirrors peat-mesh `BlobProgress`. Verified against published 0.2.10
  release (`cc93e5e` contains `blob_fetch_start`).
- **MarkerInfo OR-Set facade + deleteDocument + delete-event visibility [Shipped facade; hard-delete propagation In-flight]**
  (peat-flutter#21/#2/#3, `129c74c` — surfaced already-present bindings, 0 lines of generated code changed).
  `putMarker`/`markers`/`deleteDocument`; `MarkerInfo` is a CoT/2525-style map-marker record; delete uses an
  **OR-Set soft-delete tombstone** (`deleted:true`), because peat-mesh does not fan out `ChangeEvent::Removed`
  under default SoftDelete (`peat-mesh/src/node/mod.rs:119,206`) — that mesh-wide hard-delete is In-flight.
- **C1 published-vs-source split still holds, unchanged.** Lockfile still pins **published peat-btle 0.4.0**
  (checksum `a57dd351…`) which bundles non-FIPS `chacha20poly1305 0.10.1` + `x25519-dalek 2.0.1`, while peat-btle
  *source* (same 0.4.0) is FIPS-clean. Other pins now: peat-mesh rc.45, peat-protocol rc.29.
- **Forward-incompat flag (new todo).** The marker FFI (`put_marker`/`get_markers`/`MarkerInfo`) present in the
  pinned published 0.2.10 was **removed** from peat-ffi source 0.2.11 by peat#1022/ADR-074. When peat-flutter next
  re-pins 0.2.11, its hand-maintained marker bindings will fail UniFFI checksum unless refactored to the proto schema.
