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
feature — **trails the umbrella (`rc.27`) by 3 rc's**. **No dep on peat-mesh / peat-btle / peat-lite /
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
peat-schema pin trails the umbrella by 3 rc's. `docs/c2-collaboration.md` is accurate/current.

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
- No wire-level transport security (flagged above).
- Tests present and substantial but not run live in this read-only audit.
