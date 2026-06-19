# Module 9 — The Protocol Specifications

**Goal:** know the **normative** layer — the IETF RFC-style specs that define PEAT precisely enough
for an independent implementation. Source: [`peat/docs/spec/`](../peat/docs/spec/) (five specs +
README, ~3,750 lines; RFC 2119 MUST/SHOULD/MAY language; targeting an IETF Informational RFC).

> **When do you read these?** When you need the *exact* answer — a wire byte layout, an enum value,
> a timeout, a conflict-resolution rule. The earlier modules teach the concepts; the specs are the
> contract. Recommended order (per the spec README): **003 schema → 001 transport → 002 sync →
> 004 coordination → 005 security.**
>
> ⚠️ These specs are dated **2025-01-07**; some details have since evolved (noted inline). Where a
> spec disagrees with a newer ADR or the manifests, the newer source wins — but the specs remain the
> best single description of the protocol's shape.

---

## 9.1 `001-transport` — wire formats & connection lifecycle

Defines the `Transport` trait, the connection lifecycle, and four transports. Two identities you'll
see everywhere: **`PeerId` = first 32 bytes of SHA-256(Ed25519 public key)**, hex-serialized; wire
byte order is **big-endian**, timestamps are **Unix epoch milliseconds (u64)**.

- **QUIC/Iroh (primary):** TLS 1.3 built in. Handshake message carries Version, Type
  (`0x01` Handshake / `0x02` Ack / `0x03` ChallengeResponse), Device ID (32B), Formation ID (16B
  UUID), Nonce (32B), Public Key (32B). **Stream-ID ranges:** `0` control, `1–15` reserved,
  `16–255` CRDT sync, `256–511` app data, `512+` user. Keepalive every ~30s; idle > 120s may close.
- **UDP bypass** (latency-critical, skips CRDT): magic `0x48 0x56` ("HV"), flags for Signed/
  Encrypted, priority (0 Low → 3 Critical), TTL, sender id, optional 64B Ed25519 signature.
  **Unauthenticated by default — MUST use Signed mode on untrusted networks.** Multicast groups
  `239.255.72.86–88`.
- **Peat-Lite** (constrained devices): magic `0xCAFE`, 4B truncated device id, msg types Register/
  Ack/Heartbeat/Data, CRC-16, seq wraparound, 100ms ack timeout, 3 retries.
- **BLE mesh:** GATT service `0x1826`, data char `0x2A6E`, control char `0x2A6F`; advertises
  Formation ID + Mesh ID; 3-byte fragment header for MTU.

Reconnect backoff: 100ms → max 30s, ±10% jitter. (Spec port examples 4433/4434/4435 differ from the
operator guide's 4040 — defaults vary by binary; Module 8 §8.4.)

## 9.2 `002-sync` — CRDT semantics & reconciliation

The state layer is **exclusively Automerge** (a normative choice), reconciled with **Negentropy**.

- **CRDT types:** Map = LWW per key; List = RGA (insertion order); Text = Peritext; Counter =
  PN-Counter; primitives = registers. **Operations:** Put / Delete / Insert / Remove / Increment /
  SetRoot.
- **Identity:** Document ID = UUIDv4; **Actor ID = DeviceId[0:8] ‖ session counter (128-bit)**.
- **Sync flow:** `SyncRequest {have: heads, want: bloom_filter}` → `SyncResponse {changes, heads,
  synced}`, repeat until `synced=true`. Each Change has a SHA-256 hash, parent `deps`, actor, per-
  actor `seq`, timestamp, and compressed operations.
- **Conflict resolution (deterministic):** maps → compare **Lamport timestamps**, tie-break by
  lexicographic actor id; lists → RGA order by (Lamport ts, actor id); counters → sum(inc) −
  sum(dec). **Deletes are tombstones**, GC'd after all peers ack + a retention period (default 7
  days).
- **Negentropy:** range-fingerprint set reconciliation (`XOR(SHA256(item)[0:16])`), Init → Response →
  Finalize — O(log n) rounds, the basis of the §3.4 wire types in Module 3.
- Tuning: change bundling (100ms / 1KB), zstd level 3, subscription model with DQL filters + geo
  bbox.

## 9.3 `003-schema` — the data model (Protobuf)

Defines every tactical entity as Protobuf, organized into versioned packages
(`peat.{common,beacon,mission,capability,security,ai,cot}.v1`). Field-number ranges are reserved
(1–99 core, 100–199 standard ext, 200–299 org, 1000+ app). Highlights:

- **Beacon** (a track/position report): `track_id`, `device_id`, `callsign`, `position` (WGS84 + CEP
  accuracies), `affiliation` (MEMBER/EXTERNAL/NEUTRAL/PENDING), `dimension` (GROUND/AIR/SURFACE/
  SUBSURFACE/SPACE), `platform_type`, `status`, `power_level`, `ttl_seconds`, `confidence`.
- **Mission / Objective:** types (OBSERVATION/ACTION/PATROL/…), priority (ROUTINE→FLASH), area of
  operations, assigned cells, status lifecycle.
- **Capability advertisement:** sensors (EO/IR/RADAR/LIDAR/ACOUSTIC/RF/CBRN/GPS/IMU), actuators,
  comms (link types MESH/SATCOM/HF/VHF/UHF/LTE/WIFI/BLE), compute, power.
- **Schema evolution rules (MUST):** new fields use new numbers; existing field semantics/types never
  change; required fields aren't removed; deprecate across 2 major versions.
- **CoT/TAK mapping:** `track_id`→`uid`, `position`→`point`, `affiliation`→type prefix (`a-f`/`a-h`/
  `a-n`/`a-u`), `dimension`→atom (G/A/S/U), with a `<__peat>` detail extension. This is the normative
  basis of the CoT bridge (Modules 2 §2.8, 5, 8 §8.7).

## 9.4 `004-coordination` — cells, election, hierarchy

The normative version of Module 2·5. Defines the cell state machine (Forming → Active ↔ Degraded →
Dissolved), the formation handshake (`FormationRequest → Challenge → Response → Accept`), and leader
election. Confirms the **technical score weights** (compute 0.30, comms 0.25, sensors 0.20, power
0.15, reliability 0.10) and adds the **hybrid human-machine** dimension: an authority score
(`rank·0.6 + authority·0.3 + (1−cognitive_load)·0.1`) combined via a `LeadershipPolicy`
(RankDominant / TechnicalDominant / Hybrid / Contextual). Tie-break: authority rank → membership
duration → device id. Election timeout 5s (emergency 2s).

It also formalizes **emergent-capability patterns** (WideAreaObservation ≥3 overlapping sensors,
SenseAndAct, CoordinatedResponse, ExtendedRange, MultiSpectralFusion, EdgeIntelligence) and the
**three flows** (upward reports / downward commands / horizontal handoffs), plus partition handling
(both sides elect; the original-leader side keeps the Cell ID; merge-negotiate on heal).

> **Vocabulary caution:** spec 004 (2025-01-07) uses the older level names (Node/Team/Group/Formation/
> Cluster/Root with example sizes). The **current** enum is Platform/Cell/Cohort/Federation/Coalition
> (ADR-066, Module 2·5 §2·5.1). Same hierarchy, renamed.

## 9.5 `005-security` — auth, authz, encryption, audit

The normative security model (Module 2 §2.7 + 2·5.4 in full):

- **Identity:** Ed25519 keypair; **Device ID = SHA-256(public key)**; platform-specific key storage
  (Android Keystore, iOS Secure Enclave, TPM, ESP32 eFuse).
- **Authentication:** challenge-response (32B nonce, sign with Ed25519), reject challenges > 30s old.
  **Version-negotiated signing (ADR-065):** v1 adds `protocol_version` into the signed bytes;
  `negotiated = min(peer_version, current)` so a v1 node talking to a v0 node falls back cleanly.
  Formation-key auth = `HMAC-SHA-256(formation_key, nonce)`, constant-time compare.
- **Authorization:** roles (Observer→Member→Operator→Leader→Supervisor) × permissions, plus **access
  levels** Public(0)/Internal(1)/Restricted(2)/Sensitive(3)/Critical(4).
- **Encryption (FIPS 140-3, ADR-060 §5, amended 2026-05-18):** **AES-256-GCM** symmetric, **ECDH
  P-256/P-384** key exchange (X25519 marginal, pending FIPS review), Ed25519 signatures, SHA-256/384,
  HKDF-SHA-256, HMAC-SHA-256; TLS/QUIC under a FIPS-mode provider (`aws-lc-rs`).
  > This **supersedes** the ChaCha20-Poly1305 named in the older operator guide (Module 8 §8.5).
- **Key management:** formation-key rotation on member departure (5-min grace); group-key rotation
  toward **MLS tree ratcheting** for forward secrecy (leader-distribution fallback); retain last few
  epochs for late messages.
- **Audit:** hash-chained, signed `AuditLogEntry` ledger (tamper-evident), tiered retention
  (auth 90d, violations 1y, key mgmt 2y).

---

## 9.6 The eight facts to carry from the specs

1. **`PeerId`/`DeviceId` = SHA-256(Ed25519 pubkey)** — one derivation, used everywhere.
2. **Automerge + Negentropy** is the *only* sync stack; conflict resolution is deterministic (LWW +
   Lamport + actor-id tie-break).
3. **QUIC/Iroh primary**, with stream-id ranges partitioning control / CRDT / app traffic.
4. **UDP bypass is unauthenticated by default** — Signed mode is mandatory on untrusted links.
5. **Formation key = HMAC-SHA-256 proof-of-knowledge**, separate from the Ed25519 device challenge.
6. **Leader election is hybrid** (technical + human-authority), deterministic, no consensus round.
7. **Emergent capabilities are formal patterns**, not ad-hoc — they drive capability-based tasking.
8. **Crypto is FIPS 140-3** (AES-256-GCM / P-256-384 / Ed25519) post-ADR-060 — newer than several
   prose docs that still say ChaCha20-Poly1305 or X25519.

## 9.7 Spec ↔ ADR traceability (where to go deeper)

| Spec | Backing ADRs |
|------|--------------|
| 001 transport | ADR-010, ADR-030, ADR-032, ADR-042/043 |
| 002 sync | ADR-005, ADR-007, ADR-011, ADR-016 |
| 003 schema | ADR-012, ADR-020, ADR-028 |
| 004 coordination | ADR-004, ADR-014, ADR-024, ADR-027, ADR-066 |
| 005 security | ADR-006, ADR-044, ADR-060 (FIPS), ADR-065 (version negotiation) |

> There is also an **IRTF DINRG submission set** under `peat/spec/` (`draft-peat-protocol-00.md`),
> a more polished single-document version targeting the research-group track. The `docs/spec/` files
> are the detailed per-layer working specs.

## Try it

1. Read `docs/spec/README.md`, then skim each spec in the recommended order, matching sections to the
   modules they formalize (002→Modules 2/3 sync, 004→Module 2·5, 005→Module 2 §2.7).
2. Pick one wire format (e.g. the UDP bypass header in 001 §5) and lay out the bytes on paper.
3. Find one place a spec is out of date vs. a newer ADR (hint: the cipher in 005 pre-2026 vs. ADR-060)
   and confirm which the code follows via `peat-mesh`'s crypto deps (Module 3 §3.6).

## Checkpoint

- How are `PeerId` and `DeviceId` derived, and why does that matter for trust?
- Which CRDT backs maps vs. lists vs. counters, and how are map conflicts resolved?
- Why is the UDP bypass channel's default (unauthenticated) a footgun, and what's the fix?
- What does ADR-065 version negotiation protect against?
- Name the FIPS cipher suite and one older doc that contradicts it.

---

Back to [Module 0 — Start Here](00-START-HERE.md) · or the interactive
[`peat-learning-hub.html`](peat-learning-hub.html).
