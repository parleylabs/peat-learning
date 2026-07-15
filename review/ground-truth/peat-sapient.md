# Ground-truth audit — `peat-sapient`

**Repo:** `github.com/defenseunicorns/peat-sapient` · **Audited HEAD:** `8ec24d4` (2026-06-11,
"fix(detection): handle deprecated SAPIENT v7 feet coordinate systems; clarify MGRS status #27") ·
**Version:** `0.1.0` (`Cargo.toml:3`). SAPIENT proto pinned to upstream `dstl/SAPIENT-Proto-Files`
commit `244a741…` (`proto/VERSION:1`). First tracked 2026-06-25.

## What it is
A **Rust** library implementing the **SAPIENT (BSI Flex 335 v2.0)** sensor-interoperability protocol
and **bridging it into Peat**. Two layers: **Layer 1** (always compiled) is a standalone SAPIENT lib
— prost-generated protobuf types, a length-prefixed TCP codec, connection management; **Layer 2**
(default `peat` feature) adds bidirectional transforms between SAPIENT messages and `peat-schema`
document types plus a `SapientBridge` runtime. Run as an HLDMM TCP server: connected SAPIENT sensors
(DLMMs) register/report/detect, the bridge converts those to peat-schema events over an mpsc channel;
outbound peat commands become SAPIENT `Task`s with DIL replay. **It is the sensor-standard analog of
the TAK/CoT bridge in `peat-transport`** (`docs/c2-collaboration.md:78` models it after peat-mavlink).

## Structure
- **SAPIENT layer (always):** `src/proto.rs` (generated), `src/codec.rs` (4-byte LE length prefix,
  16 MiB frame cap), `src/connection.rs` (HLDMM listen / DLMM connect-with-retry), `src/error.rs`,
  vendored `proto/` (BSI Flex 335 v2.0).
- **Transforms (`peat` feature):** `src/transform/{registration,status,detection,alert,task}.rs`.
- **Bridge runtime:** `src/bridge.rs` (`SapientBridge`, `route_message`), `src/registry.rs`
  (`NodeRegistry`), `src/rate_limit.rs` (per-node token bucket), `src/task_queue.rs` (DIL per-node
  FIFO + TTL), `src/watchdog.rs` (heartbeat timeout). Unit tests in every src file + `tests/`
  (codec/connection/proto_roundtrip + an Apex integration harness that auto-skips without `apex.py`).

## SAPIENT specifics
- **BSI Flex 335 v2.0 only.** The recent "SAPIENT v7" commit handles **deprecated feet-based
  coordinate enums** older sensors emit (decoded as raw enum ints before typed `try_from`,
  `src/transform/detection.rs:234-257`) — it does NOT target a v7 wire protocol.
- **Inbound → SapientUpdate:** `Registration`→`Registered`, `StatusReport`→`StatusUpdated`,
  `DetectionReport`→`Detected`, `Alert`→`Alerted`, `TaskAck`→`TaskAcknowledged`; all else →`Ignored`
  (never panics) (`src/bridge.rs:378-461`). **Outbound:** auto `RegistrationAck` + `Task`.
- **Coordinates** (`detection.rs`): LatLng deg/rad, deprecated feet, **UTM→WGS84 inverse**
  (Snyder/Transverse-Mercator, `utm_to_latlon:139-195`), RangeBearing variants, CEP via 0.5887.
  **MGRS explicitly NOT supported** (not a v2.0 enum variant; ADR-070 anticipated it).

## Relationship to other Peat repos
Depends on **`peat-schema` only**, pinned **`0.9.0-rc.24`** (`Cargo.toml:38`) behind the `peat`
feature — **trails the umbrella (`rc.28`) by 4 rc's**. **No dep on peat-mesh / peat-btle / peat-lite /
peat-gateway / peat-node / peat-transport.** A pure schema-level translator; mesh propagation is left
to a consuming Peat node. SAPIENT data lands in **peat-schema docs**: detections→`track::v1::Track`,
registration→`capability::v1::CapabilityAdvertisement`, status→`node::v1::NodeState`,
alerts→crate-local `SapientAlertEvent` (deliberately NOT `peat_schema::AlertProduct`); outbound
`command::v1::HierarchicalCommand`→SAPIENT `Task`.

## Capabilities
- **Shipped** — SAPIENT TCP codec (`codec.rs:21-64`); DLMM connect-w/-backoff + HLDMM accept
  (`connection.rs:57-82`); all five transforms (registration `34-97`, status `56-105`, detection
  `374-455`, alert `66-117`, task `47-84`); `SapientBridge::start()` listener + routing + auto
  RegistrationAck (`bridge.rs:137-330`); **DIL outbound task queue** (per-node FIFO/TTL/replay,
  `task_queue.rs:35-125`); `TaskAck`→`TaskAcknowledged` (`bridge.rs:420-433`); per-node detection
  rate limiter (`rate_limit.rs:56-83`); heartbeat watchdog→`NodeDisconnected` (`watchdog.rs:20-55`);
  `NodeRegistry` (`registry.rs:31-86`).
- **In-flight** — propagating `TaskAcknowledged` upward as peat `CommandAcknowledgment`, blocked on a
  peat-schema `CommandCoordinator` extension (`docs/c2-collaboration.md:281-285`).
- **Proposed** — Direction B (a Peat node acting as a DLMM registering to an external HLDMM), blocked
  on an authority ADR; first-class peat-schema `Alert` type to replace the local struct.
- **Speculative** — MGRS coordinate support (`detection.rs:22-23`).

## Crypto / FIPS posture — ⚠ FLAG
**No cryptography in this crate** — grep for FIPS *and* banned primitives returned zero crypto hits.
The SAPIENT wire transport is **plain TCP with no TLS** (`src/connection.rs`, `src/codec.rs`). Nothing
non-FIPS to flag, but for a defense deployment note that **confidentiality/integrity must be provided
by the surrounding environment** — the crate ships no transport encryption.

## README-vs-code drift
`src/transform/mod.rs:3-5` stale "All modules are stubs until Phase 3" (all are implemented); README
quick-start uses `state.health_status` but the field is `health` (`status.rs:78`) — would not compile;
peat-schema pin trails the umbrella by 4 rc's. `docs/c2-collaboration.md` is accurate/current.

## Curriculum mapping
**Section / case-study in `07-repo-links-and-gaps`**, paired with peat-transport's TAK/CoT bridge as
"the two external-standard bridges." Cross-reference `09-protocol-specs` (SAPIENT/BSI Flex 335 v2.0
wire format) and `06-data-flows` (sensor→Track→mesh up; HierarchicalCommand→Task down). Touches
`02b-formation-and-leadership` lightly (bridge node sits at the Cell tier). **Do not create a
standalone module yet** — promote only if SAPIENT becomes a first-class delivered capability (e.g.
once `TaskAck→CommandAcknowledgment` lands and a Peat node ships it wired in).

## Unverifiable / flagged
- Apex `message_io.py` framing citation (`codec.rs:4`) — Apex not vendored; integration tests
  auto-skip without `apex.py`.
- `proto/VERSION` upstream hash `244a741…` — correspondence to "BSI Flex 335 v2.0" asserted, not
  verifiable offline.
- ADR-070 / ADR-009 references point to the `peat` umbrella, not present here.
- No wire-level transport security (flagged above). **SUPERSEDED 2026-07-06 — see delta below: opt-in TLS now ships.**
- Tests present and substantial but not run live in this read-only audit.

---

## 2026-07-06 delta — `8ec24d4 → 5118afa` (v0.1.0; major restructure)

Range = 13 commits. This supersedes the "single lib, depends on peat-schema ONLY, plain TCP no TLS" facts above.

**Now a 3-crate workspace** (root `Cargo.toml:3`, resolver 2):

| Crate | Role | Depends on |
|---|---|---|
| `peat-sapient` | Core BSI Flex 335 v2.0 lib: proto codec, TCP/TLS connection mgmt, transforms, `SapientBridge` C2 (task/ack/watchdog), rate limiting | `peat-schema` **optional** behind `peat` feature (`Cargo.toml:20,44`); NOT peat-mesh. TLS deps optional. |
| `peat-mesh-sapient` | One-way `impl Translator`/`Transport` for SAPIENT (ADR-059 Amend.4) + bridge→mesh subscriber glue | `peat-mesh =0.9.0-rc.41` (`:23`), `peat-sapient` (path), `peat-schema =0.9.0-rc.24` (`:25`) |
| `peat-sapient-bridge` | Deployable **binary**: bidirectional SAPIENT↔mesh↔TAK bridge service | `peat-mesh =rc.41`, `peat-mesh-sapient`, `peat-sapient`, **`peat-tak =0.0.2`** (`:22`); dev-dep `peat-protocol =rc.28` |

- **TLS is now Shipped, but OPTIONAL / default-off (peat-sapient#39).** `peat-sapient-bridge/Cargo.toml:15`
  `default = ["tls"]` so the shipped binary is built with TLS, but at runtime
  `config.sapient.tls`/`config.tak.tls` are `unwrap_or(false)` (`peat-sapient-bridge/src/main.rs:132,164`)
  — **plain TCP is still fully supported** (`connection::connect`/`accept`; TLS variants
  `connect_tls`/`accept_tls` are parallel, `src/connection.rs:270,312`). TLS covers (a) SAPIENT DLMM↔HLDMM
  (mTLS both directions, `connection.rs:147,201`) and (b) TAK-server mTLS via `peat_tak::TakIdentity`
  (`main.rs:165-197`). Mesh side is separate (iroh QUIC). Library = `tokio-rustls 0.26` + `rustls-pemfile 2.2`,
  provider `aws_lc_rs`.
- **FIPS posture — PARTIAL (source-level flag; all crates 0.1.0, unpublished).** `fips_crypto_provider()`
  (`connection.rs:337-348`) restricts *cipher suites* to AES-GCM only (TLS1.3 AES-256/128-GCM; TLS1.2
  ECDHE-{ECDSA,RSA}-AES-GCM) and thus excludes ChaCha20. **But** (1) it spreads
  `aws_lc_rs::default_provider()`'s `kx_groups` unchanged, so **X25519 (non-FIPS) is still offered**
  alongside P-256/P-384 — cipher-suite filtering does not constrain the ECDHE group; and (2) it uses the
  standard `default_provider()`, **not** the `aws-lc-rs` *fips* module, so even the AES-GCM suites run
  through the non-validated build. The `// FIPS-approved cipher suites only` comment overstates. **NEEDS_RUNTIME**
  to prove X25519 is actually negotiated (it is offered by construction).
- **SAPIENT → mesh → TAK/CoT via Translator (peat-sapient#28) [Shipped].** `impl Translator for SapientTranslator`
  (`peat-mesh-sapient/src/translator.rs:91`): DetectionReport→`"tracks"`, Registration/StatusReport→`"platforms"`
  (`transport.rs:246-250`). Subscriber glue `run_bridge_subscriber` (`subscriber.rs:41`) publishes
  Registered/StatusUpdated→platforms, Detected→tracks (origin `"sapient"`). C2 (TaskAck/Alert/AlertAck) not
  yet published (no v1 mesh doc). SAPIENT↔CoT is achieved at the mesh layer — the bridge binary registers
  both `SapientTranslator` and `peat_tak::CotTranslator` on one `TransportManager` and fans out the shared
  `tracks` collection (`main.rs:129,162,208-224`) — not a direct SAPIENT→CoT converter.
- **New capabilities [Shipped]:** AlertAck/`AlertAckStatus` (`bridge.rs:60,292,591`), `command_id` correlation
  in TaskAck (`bridge.rs:120-127,255-259,449-456`), watchdog (`watchdog.rs:20`, emits `NodeDisconnected` after
  2× silence), per-node DetectionReport rate limit (`rate_limit.rs:15,56`), DLMM reconnect-after-HLDMM-drop
  (`peat-mesh-sapient/src/transport.rs:459` `run_dlmm_connect_loop` + TLS variant, #40), criterion benchmarks.
  **DLMM** = Detection-Level Multi-sensor Management Module (sensor/platform); **HLDMM** = High-Level Decision
  Making Module (manager); Peat can act as either (default HLDMM, `main.rs:239`).
- **peat-schema pin unchanged at `0.9.0-rc.24`** (`peat-sapient/Cargo.toml:44`, `peat-mesh-sapient/Cargo.toml:25`) —
  **trails the umbrella (rc.29) by 5**. Pins are now internally inconsistent (peat-mesh rc.41, peat-protocol
  dev-dep rc.28, peat-schema rc.24). Open todo to re-check on each sync (a bump may rename transform fields).
- **Curriculum mapping this run:** Module 7 §7.1 (row rewritten to 3-crate workspace + opt-in TLS) and §7.8
  (FIPS-posture note added — flips the prior "plain TCP, no TLS" flag with the precise caveats above). A full
  Module 7 SAPIENT case-study section + a Module 9 BSI-Flex wire-format section remain open_todos for the full sweep.

### 2026-07-13 delta — `5118afa → 93d51ac` (v0.1.0; +12 commits)
- **peat-schema lag closed [Shipped].** `peat-sapient/Cargo.toml:46` + `peat-mesh-sapient/Cargo.toml:25`
  bumped `peat-schema 0.9.0-rc.24 → 0.9.0-rc.30` (caret) — now matches the umbrella; the prior 5-RC lag is gone.
  `peat-mesh` (rc.41) and `peat-protocol` (rc.28, dev-dep) unchanged. Temporary git-dep patches introduced then
  removed within this window — at HEAD all external deps are plain crates.io pins (no `[patch]`, no `git =`).
- **position_error migration is DUAL-WRITE, not a removal [Shipped]** (`src/transform/detection.rs:503` etc.).
  Track now carries `position_error: Some(PositionError{circular_error←CEP, vertical_error←z_error,
  linear_error=0.0 always})` — but the deprecated `cep_m`/`vertical_error_m` are **still populated** under
  `#[allow(deprecated)]` (tests assert `pe.circular_error == pos.cep_m`). `Track.kinematics` present but `None`
  everywhere. Confirms the rc.30 peat-schema surface (new `common::v1::PositionError`, `kinematics` field).
- **`--tak-protocol` flag [Shipped]** (`peat-sapient-bridge/src/config.rs:100`, consumed `main.rs:191`): selects
  the CoT **wire encoding** — `raw-xml`/`xml-tcp`/`protobuf-v1` (→ `peat_tak::TakProtocolVersion`), unknown values
  hard-error. Orthogonal to `--tak-tls` (transport). Inbound framing auto-detected per-message (`0x3C` XML vs `0xBF` protobuf).
- **peat-tak 0.0.2 → 0.0.3 [Shipped]** (`peat-sapient-bridge/Cargo.toml:22`, feature `mesh-translator`).
- **BSI Flex 335 v2.0 compliance CI [Shipped]** (peat-sapient#41, `.github/workflows/ci.yml:71-121`). Runs the
  DSTL `dstl/BSI-Flex-335-v2-Test-Harness` (.NET, pinned ref `6a2999d`) as an HLDMM validator against a Rust
  `sapient-compliance-client` on port 15066. This is **standards-interop compliance, NOT cryptographic FIPS
  validation** — do not conflate. Vendored proto pin unchanged (`proto/VERSION` = `244a741…`, a separate DSTL repo).
- **TLS/FIPS posture UNCHANGED — `connection.rs` has an empty diff this cycle.** `fips_crypto_provider()`
  (`connection.rs:337-349`) restricts cipher suites to AES-GCM but spreads `aws_lc_rs::default_provider()` so
  **X25519 (non-FIPS) is still offered at key exchange**, and it is the standard provider, not the aws-lc-rs *fips*
  module. Report as "AES-GCM suites but X25519 kx still offered," never "FIPS-clean." One change to flag: the
  `peat-sapient-bridge` crate now enables `tls` **by default at compile time** (`Cargo.toml:15` `default=["tls"]`)
  — but runtime TLS is still opt-in (`config.*.tls` default false, `main.rs:132,164`) and needs cert paths.
  Operator guide added (`docs/operator-guide.md`). TAK Server 5.7 mTLS interop is **manually** validated (In-flight),
  distinct from the automated BSI job. All three crates still 0.1.0, path-linked, unpublished.
