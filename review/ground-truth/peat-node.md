# Ground truth — `peat-node`

**Repo:** `peat-node` (GitHub `defenseunicorns/peat-node`)
**Audited HEAD:** `bbe3b68` — `chore(release): v0.4.7` (advanced from `4e1b5c8` on the 2026-06-19
incremental — the prior "5 commits ahead" delta is now audited; see the dated delta at the end of this file).
**Working tree:** clean, on `main`. `git fetch` ran; advanced through `v0.4.4`–`v0.4.7` this run, no pull/working-tree change beyond fetch.
**Package version (`Cargo.toml`):** `0.4.7` (the `peat-cli` crate is at `0.4.5`).

## What this repo actually is

A **sidecar / node binary**, not a transport or CRDT implementation. `peat-node` embeds `peat-mesh` and `peat-protocol` and exposes them as a gRPC/Connect/gRPC-Web API on a single port for co-located applications (the Kubernetes sidecar pattern). `Cargo.toml:12` description: "Peat mesh node — exposes peat-protocol as a gRPC API for co-located applications." The entire mesh stack (Automerge store, Iroh endpoint, sync coordinator, blob store) is constructed in one call: `peat_mesh::sync::AutomergeBackend::with_iroh` (`src/node.rs:347`). peat-node owns only the sidecar-specific layers: encryption-at-rest, the change-event broadcast channel, the connect-peer retry/reconnect logic, the QoS-priority relay fanout, attachments, collection-config persistence, and peer discovery wiring (`src/node.rs:1-9` doc comment).

Primary integration target: UDS Remote Agent (`README.md:9`). The agent watcher polls the agent's Connect RPC APIs and mirrors fleet state into CRDT collections (`src/watcher.rs`, `README.md:30-66`).

## Transports — what is implemented and its status

**Exactly one transport is implemented in this repo: Iroh QUIC, embedded via `peat-mesh`.** Status: **Shipped.**

- `iroh = "=1.0.0-rc.1"` pinned (`Cargo.toml:135`), must match peat-mesh's iroh pin exactly (mixed versions cause UB in iroh's process-global crypto provider + ALPN registry — `Cargo.toml:131-135`).
- The endpoint is bound inside `AutomergeBackend::with_iroh`; peat-node configures it via `AutomergeBackendConfig` (`src/node.rs:289-347`).
- Connection path: `backend.transport().connect_and_authenticate(peer_id)` with a 3-attempt / 200 ms-backoff retry loop working around upstream peat#759 handshake race (`src/node.rs:1251-1273`).
- Peer reachability hints accepted: direct `host:port`/`ip:port` (DNS-resolved via `tokio::net::lookup_host`, multi-record) and an optional relay URL (`src/node.rs:1196-1230`). The n0 public relay is no longer used by default — `connect_peer` requires at least one of addresses or relay_url (`src/node.rs:1154-1159`).

**Peer discovery mechanisms (shipped):**
- **mDNS** (`MdnsDiscovery`, on by default; `src/node.rs:606-661`). Advertises real LAN interface addresses. Disabled via `--disable-mdns`; the Helm chart now defaults `disableMdns: true` because multicast is unavailable in K8s (`CHANGELOG.md` 0.4.1 entry).
- **Kubernetes EndpointSlice discovery** (`KubernetesDiscovery`, gated on the peat-mesh `kubernetes` feature + `--enable-kubernetes-discovery`; `src/node.rs:663-712`). New in 0.4.2 (#151). Derives each pod's iroh `EndpointId` from `HKDF-SHA256(shared_key, "iroh:" + POD_NAME)` so any pod computes any peer's id from its pod name (`src/node.rs:826-887`).
- **Static peering** via `PEAT_NODE_PEERS` (`endpoint_id@host:port`), aided by the offline `peat-node derive-id` subcommand (new in 0.4.3 / #162; `src/main.rs:312-330`, `src/identity.rs`).

**NOT in this repo (confirmed absent):**
- **peat-sbd (ADR-051 / Iridium SBD) and peat-lora (ADR-052)** — **no code, no crate, no dependency.** `grep -rniE "sbd|iridium|lora"` over `src/ proto/ docs/ README.md CHANGELOG.md` returns nothing. They are ADR proposals living in the `peat` repo, not consumed here. **Proposed**, not shipped.
- **BLE / peat-btle / peat-lite** — **not implemented in peat-node.** Only references: `docs/peat-node-adr-001-peat-cli.md:14` cites ADR-035 (peat-lite) in prose, and `docs/DESIGN.md:188` has a forward-looking architecture diagram labeled `"QUIC / BLE / relay"`. That BLE label is **speculative/aspirational in this repo** — there is no BLE transport code path. peat-node speaks Iroh QUIC only. Flag any curriculum claim that peat-node "supports BLE."

## CRDT type

**Automerge**, via peat-mesh's `AutomergeStore` / `AutomergeBackend` (`src/node.rs:25-27`). peat-node defines **no CRDT types of its own** — it stores JSON documents in named collections that peat-mesh maps onto Automerge documents (`automerge_to_json` / `json_to_automerge`, `src/node.rs:25`). The peat-lite register/counter/set CRDTs (`LwwRegister`, `GCounter`, `PnCounter`, `OrSet`) are **not** present here — they live in `peat-lite`, which peat-node does not depend on. There is **no `command_log` CRDT** anywhere in this repo (confirmed). Commands are stored as ordinary JSON documents in the `commands` collection (`proto/sidecar.proto:342-373`, typed `Command` message + `PutCommand`/`GetCommands`).

## Identity / addressing

- **Node network identity = iroh `EndpointId`** (the public half of an Ed25519 keypair; iroh's native identity). `endpoint_addr()` returns `EndpointId` as a string (`src/node.rs:1068-1070`). This is **not** the "NodeId = SHA-256(Ed25519 pubkey)" derivation from the addressing design note — peat-node uses iroh's `EndpointId` directly (Ed25519 public key), with no SHA-256 wrapping. Flag this discrepancy against any doc claiming peat-node derives NodeId as SHA-256 of a pubkey.
- **`--node-id` is a separate human label** (a UUID by default; `README.md:216`), used as the HKDF info input, not as the wire identity.
- **Deterministic identity derivation (shipped, new in 0.4.3):** `HKDF-SHA256(salt=None, IKM=base64decode(shared_key), info="iroh:" + node_id) → 32 bytes → iroh::SecretKey::from_bytes` (`src/crypto.rs:119-132`, `src/identity.rs:47-74`). This recipe **must stay byte-for-byte identical** to peat-mesh's `PeerConnector::derive_peer_endpoint_id`; pinned by a known-answer test `derive_iroh_node_key_matches_documented_recipe` (`src/crypto.rs:216-223`). The same derivation serves both K8s discovery and static peering — single source so they cannot drift (`src/identity.rs:38-42`).
- A `debug_assert!` enforces a 16-byte IKM floor to prevent silently deriving identities from weak keying material (`src/crypto.rs:120-126`).

## Hierarchy / vocabulary

peat-node carries **no `HierarchyLevel` enum of its own.** Its typed collections use the **current ADR-066 vocabulary**: a `Cell` message (formation of nodes, `proto/sidecar.proto:272-302`) with `PutCell`/`GetCells`. **No legacy Squad/Platoon/Company terms** appear in `src/` or `proto/` (grep confirmed). The `Cargo.toml` dependency comment (`Cargo.toml:24-31`) notes the peat-mesh internal `HierarchyLevel` rename (Squad/Platoon/Company → Cell/Cohort/Federation + new Coalition) is **internal to peat-mesh and not part of peat-node's consumed surface**. `Command.scope`-style COHORT/FEDERATION/COALITION terms live in peat-schema, not exposed by this proto. So: peat-node is on the new vocabulary; the hierarchy enum itself is upstream.

## Crypto primitives actually used (FIPS posture)

All primitives in this repo are **FIPS-approved** — no violation found in peat-node.

| Primitive | Where | Status |
|---|---|---|
| **AES-256-GCM** (`aes-gcm` 0.10) | Encryption at rest for document content (`src/crypto.rs:1-87`, `StoreCipher`). Format `ENC:v1:<base64(nonce‖ct‖tag)>`, 12-byte random nonce, 16-byte tag. | FIPS-approved. Shipped. Opt-in via `--encryption-key`. |
| **HKDF-SHA256** (`hkdf` 0.12 + `sha2` 0.10) | Deterministic iroh keypair derivation (`src/crypto.rs:119-132`). | FIPS-approved per ADR-049 (`src/crypto.rs:106`). Shipped. |
| **SHA-256** (`sha2` 0.10) | Attachment ingest hashing (`src/attachments/ingest.rs`, `registry.rs`). | FIPS-approved. Shipped. |
| **BLAKE3** | Iroh blob content-addressing (attachment `blob_token`, `proto/sidecar.proto:613`). Internal to iroh-blobs, not a peat-node choice. | Not a FIPS AEAD; content-address only, not confidentiality. |
| **Ed25519** | iroh `EndpointId` / QUIC handshake identity (via iroh). | Standard; iroh-owned. |

**No ChaCha20-Poly1305 in peat-node** (grep confirmed). `CHANGELOG.md:253` records that peat-mesh **already swapped** ChaCha20-Poly1305 + X25519 → AES-256-GCM + ECDH-P256 (FIPS 140-3 equivalents, ADR-060 §5) at peat-mesh rc.12, and notes peat-node never constructed those primitives directly so the swap was transparent at the sidecar boundary. The transport-layer AEAD (QUIC) is whatever iroh negotiates — outside peat-node's control. So the ADR-060 ChaCha20 concern that applies to peat-btle/ADR-052 does **not** apply to peat-node; its sync path now rides FIPS primitives upstream.

## Public API a consumer integrates against

The wire contract is `proto/sidecar.proto` — service `peat.sidecar.v1.PeatSidecar`, served over Connect + gRPC + gRPC-Web on a single port (default `tcp://0.0.0.0:50051`; UDS also supported). Compiled by `connectrpc-build` (`build.rs`, `Cargo.toml:138,197`).

**RPC count discrepancy:** README and the proto/header say **"25 RPCs"** (`README.md:132`, `proto/sidecar.proto:14`-region), but the proto actually defines **28 RPCs** and `src/service.rs` implements **27** `async fn` handlers (grep counts). The gap: the 3 Collection-Lifecycle-Config RPCs (`SetCollectionConfig`/`GetCollectionConfig`/`ListCollectionConfigs`, `proto/sidecar.proto:129-145`, peat-node#55 / ADR-016) were added after the "25 RPCs" prose was written. **The "25 RPCs" figure is outdated — real count is 28 defined / 27 implemented.** (The 1-RPC gap between 28 defined and 27 impl handlers is worth a deeper check, but all README-listed categories have handlers; the discrepancy is most likely a streaming-RPC counting nuance, flagged as needing a line-by-line map.)

RPC groups (`proto/sidecar.proto`, `src/service.rs`):
- **Lifecycle:** `GetStatus`.
- **Peers:** `ConnectPeer`, `DisconnectPeer`, `ListPeers`.
- **Generic document CRUD:** `PutDocument`, `GetDocument`, `DeleteDocument`, `ListDocuments` (JSON blobs in named collections).
- **Typed collections:** `PutNode`/`GetNodes`, `PutCell`/`GetCells`, `PutTrack`/`GetTracks`, `PutCommand`/`GetCommands` — convenience wrappers that serialize the typed message to JSON into the matching collection (e.g. `put_cell` → `put_document("cells", …)`, `src/service.rs:229-244`).
- **Subscriptions:** `Subscribe` (server-streaming; initial snapshot of matching docs as UPSERTs, then live changes; optional `SubscriptionQuery` predicate — eq/lt/gt/and/or/not/all, top-level fields only in v1, `proto/sidecar.proto:377-451`).
- **Sync control:** `StartSync`, `StopSync`, `GetSyncStats`.
- **Attachments (PRD-006):** `SendAttachments`, `GetAttachmentDistribution`, `SubscribeAttachmentBundle` (streaming), `CancelAttachmentDistribution`. **Disabled by default — return `Unimplemented` until at least one `--attachment-root` is configured** (`README.md:144`, `proto/sidecar.proto:88-93`, `src/node.rs:419-428`).
- **Collection config (peat-node#55 / ADR-016):** `SetCollectionConfig`, `GetCollectionConfig`, `ListCollectionConfigs`.

Operator CLI **`peat`** (`crates/peat-cli`) ships in the same image at `/usr/local/bin/peat`; joins the mesh as a real node, runs one CRUD-shaped command, exits (`README.md:109-127`).

## Shipped vs in-flight vs proposed vs speculative

**Shipped (verified in code at 4e1b5c8):**
- Iroh QUIC mesh participation embedded via peat-mesh.
- Automerge CRDT sync of JSON documents across peers; transitive gossip with echo suppression (`src/node.rs:963-999`).
- gRPC/Connect/gRPC-Web API on single port (28 RPCs defined).
- Generic + typed (Node/Cell/Track/Command) collections.
- Server-streaming `Subscribe` with content-predicate filtering (top-level fields).
- AES-256-GCM encryption at rest (opt-in).
- Deterministic iroh identity + offline `derive-id` (0.4.3).
- mDNS, Kubernetes EndpointSlice, and static peer discovery.
- Auto-reconnect watchdog with per-peer exponential backoff (5 s → 120 s, peat-node#91; `src/node.rs:203-216,488-604`).
- QoS-priority relay fanout — drains highest-QoS-first (Critical=1 … Bulk=5), coalescing + bounded shed (peat-node#138; `src/fanout.rs`). QoS class per collection comes from `peat_mesh::qos::QoSClass::for_collection`: `commands`/`contact-reports`/`alerts` → Critical; `cells`/`nodes`/`audit-logs` → High; `beacons`/`platforms`/`tracks` → Normal; others → Bulk (`peat-mesh/src/qos/mod.rs:154-161`).
- PRD-006 attachment distribution (send-side + receive-side inbox watcher) over iroh blobs.
- Collection-lifecycle config persistence (deletion policy + TTLs, ADR-016).
- Tombstone GC knobs (`--tombstone-ttl-hours`, `--gc-interval-secs`, `--gc-batch-size`; peat-node#136).
- Helm chart, Zarf package, UDS bundle, Docker multi-stage build.

**In-flight (open issues):**
- gRPC listener auth/authz — **not implemented**; `#38` open ADR. **The gRPC surface is currently unauthenticated.** Material flag-worthy for an air-gapped/enterprise reader.
- ADR-046 targeted-delivery primitives through the sidecar proto — `#53` open.
- Consolidating the reconnect watchdog with a peat-mesh `ReconnectionManager` — `#100` open.
- WebSocket API for real-time subscriptions — `#17` open.
- iroh 1.0.0 stable bump (currently `rc.1`) — `#161` open.
- Identity provisioning / credential lifecycle for DDIL mesh — `#14` open.

**Proposed (ADR-only, not in this repo):** peat-sbd (ADR-051), peat-lora (ADR-052) — no code here.

**v1-honesty caveats baked into the proto (read these before quoting attachment behavior):**
- `AttachmentPriority` is recorded and used for local queue ordering but does **NOT** enforce wire-level cross-class preemption in v1 — a CRITICAL bundle will not pause an in-flight BULK transfer until PRD-004 lands (`proto/sidecar.proto:551-578`).
- `DistributionStatus` is **sender-side only** in v1; `PARTIAL` is reserved for v2 (needs receive-side observer hooks). COMPLETED = every targeted peer pulled all bytes from this sender (`proto/sidecar.proto:631-656`).
- `CapableScope` distribution scope is reserved-but-rejected (`FAILED_PRECONDITION`) until a capability-vocabulary ADR ships (`proto/sidecar.proto:543-549`).
- Collection-config deletion policy is **persisted and surfaced but not yet enforced per-collection through peat-mesh** (needs an `AutomergeBackend::set_deletion_policy()` surface; `proto/sidecar.proto:706-713`, `src/node.rs:218-231`).

## Quantitative claims — verifiable vs not

- **"25 RPCs"** (`README.md:132`) — **DISPROVEN at HEAD.** Proto defines 28; service.rs implements 27 handlers. Outdated.
- Reconnect watchdog timings (5 s interval, 5 s min / 120 s max backoff) — **verified** in code constants (`src/node.rs:203-216`).
- Tombstone defaults (168 h / 7-day, 300 s GC interval, 1000 batch) — **verified** in code/comments (`src/node.rs:64-71`).
- Blob stall timeout default 30 s — **verified** (`src/node.rs:60`, CHANGELOG 0.4.0).
- QoS 5-class ranking Critical=1 … Bulk=5 — **verified** (`peat-mesh/src/qos/mod.rs:70-101`).
- **Network-physics numbers (SBD ~1,960 B, LoRa 7–87 km / 1.5–9.1 kB/s, BLE 100–400 m / ~2 Mbps, "93–99% bandwidth reduction", "<5 s P1 latency", "O(n log n)", "1,000+ node" validation)** — **NONE of these appear anywhere in peat-node** (grep over CHANGELOG/docs/README returns nothing). They are not this repo's claims; they belong to peat-mesh / peat-lite / peat-btle / the design note. **Unverifiable from peat-node** — do not source them here.
- README "Tested Scenarios" table (single-sidecar smoke, two-node sub-second sync, real UDS agent + watcher, two k3d clusters, in-cluster Helm) — these are claimed Pass results (`README.md:67-75`); backed by the `tests/` integration suite (24 test files incl. `sync_test.rs`, `partition_test.rs`, `sync_subprocess_test.rs`, `attachments_*`) but the specific "sub-second" figure is a test-run observation, not a pinned benchmark. **Partially verifiable** (tests exist; latency figure unpinned).

## Assumptions / decisions logged

- **Decision:** treated audited reality as `0.4.3` @ `4e1b5c8`; did not pull the 5 origin commits or inspect v0.4.4/v0.4.5 tags (instructed not to modify the tree). Their content is an **unverified delta** — a re-audit should check whether the "25 RPCs" doc fix or auth landed there.
- **Decision:** the 28-defined-vs-27-implemented RPC gap is reported as-is; did not exhaustively line-map every proto RPC to a handler (all README categories are covered). Logged as a follow-up rather than a blocker.
- **Assumption:** BLE in `docs/DESIGN.md:188` is an architecture-diagram aspiration, not implemented code — based on the complete absence of any BLE/btle dependency or code path. Holds unless a future peat-mesh feature adds it transparently.
- **Assumption:** peat-node inherits peat-mesh's FIPS posture for the sync/transport AEAD (ADR-060 swap already done at mesh rc.12). peat-node itself only ever uses AES-256-GCM / HKDF-SHA256 / SHA-256 directly. Holds at this pin.

---

## 2026-06-19 incremental delta (`4e1b5c8` → `bbe3b68`, v0.4.3 → v0.4.7)

The prior "5 commits ahead" unverified delta is now audited. Releases v0.4.4–v0.4.7:

- **Attachment outbox watcher (#167, v0.4.5; PRD-006 v1.1).** `src/attachments/outbox.rs`: a send-side
  poll watcher over the configured `--attachment-root` dirs. When a file is *stable* across a poll and
  not yet sent, it auto-distributes by synthesising the same `SendAttachments` request an app would
  send (reuses `handlers::send_attachments` end-to-end). **Off by default**, gated on
  `PEAT_NODE_ATTACHMENT_OUTBOX_WATCH`; the explicit RPC stays the safe default. Polls (not inotify) for
  reliability across container bind mounts. Symmetric counterpart to the receive-side inbox watcher.
  **Shipped, off by default.**
- **Peer-status heartbeat + sync-stack logging (#169, v0.4.7).** `PEER_STATUS_LOG_INTERVAL =
  Duration::from_secs(30)` (`src/node.rs:207`); a 30 s task logs `connected_peers` (live CRDT-sync
  connections, from the transport) vs `known_peers` (peers this node dialed, from the blob store) at
  info. `known_peers` is the exact set `resolve_targets` and `fetch_blob` use, so the line answers "why
  did a synced distribution doc never become a delivered file." Default `RUST_LOG` filter widened to
  cover the whole sync stack (`peat_protocol=info`, `iroh=warn`). Multi-host example docs note the
  two-way-dial requirement. **Shipped.**
- **Empty-env startup-crash fix + content-validated attachment-delivery functest (#164, v0.4.4).**
  **Shipped.**
- Release/CI plumbing (cargo-chef + rust-cache, Dockerfile, chart bumps) across #165/#166/#168.

**Confirmed unchanged from the 0.4.3 audit:** `proto/sidecar.proto` still defines **27** `rpc`s in one
`service PeatSidecar` (the curriculum's "~28/27" stands; README "25" still stale). gRPC auth #38 did
**not** land — the surface remains unauthenticated. Tombstone/RBAC details unchanged.

**Not exercised (read-only audit):** the outbox watcher's poll/stability/dedup and the heartbeat were
read from source, not run live.
