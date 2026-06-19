# PEAT Curriculum Review — Change Log

Per-document change log for the curriculum rewrite. Each entry records what was corrected in
that document and why, assembled from the 14 per-document rewrite passes. All rewrites were done
in place against the audited HEADs recorded in `learning/review/ground-truth.md` §1 and the merged
`learning/review/claim-ledger.md`. Originals are preserved in `learning/review/originals/`.

The headline corrections, applied consistently across every document:

- **Hierarchy leaf tier:** the shipped enum leaf is **`Node`**, not `Platform`. ADR-066 is
  **Proposed** (not "current/authoritative"); the rename is mid-flight (#904/#968/#970, ADR-068).
  peat-btle is still fully legacy `Platform/Squad/Platoon/Company`; peat-schema proto still ships
  `SquadSummary`.
- **Identity:** there is no 256-bit NodeId and no "one derivation everywhere." `DeviceId` =
  SHA-256(Ed25519 pubkey) **truncated to 16 bytes**; four real schemes exist (mesh `DeviceId` 16 B,
  peat-node raw iroh EndpointId, peat-btle BLAKE3[..4] u32, peat-lite bare u32).
- **Crypto / FIPS:** shipped code is FIPS-clean — **AES-256-GCM + ECDH-P256 + Ed25519 + HKDF/HMAC-
  SHA-256 over `aws-lc-rs`** (rc.12 swap, 2026-05-18). P-256 only (no P-384). ChaCha20 survives only
  in **proposed ADR-052** and stale READMEs and is **flagged, never propagated**. The
  **FIPS-approved-algorithm vs CMVP-validated-module** distinction (peat-btle#75) is stated. **MLS is
  Proposed (ADR-044), not shipped.**
- **Roles:** removed the invented `CapabilityType::Weapon`; corrected RBAC to the real
  `Leader/Member/Observer/Commander/Admin` (README's `Operator/Supervisor` called out as wrong).
- **Status labels:** every document gained a visible **Shipped / In-flight / Proposed / Speculative**
  legend and per-capability labels; every quantitative figure is now cited or flagged.
- **Vendor names removed:** "ATAK/WebTAK/TAK Server" → "CoT consumer"; protocol names (CoT, TAK,
  MIL-STD-2525) retained. "Autonomy under human authority" framing preserved on all tasking content.

---

## Top claims that remain UNVERIFIABLE (chase before publish)

Pulled from every rewrite pass. A human should resolve these against live source / a re-audit:

- **peat-node origin delta:** peat-node origin is **5 commits ahead** of the audited HEAD `4e1b5c8`
  (v0.4.4/v0.4.5 tagged, NOT audited). The 28/27 RPC count, gRPC auth (#38), tombstone/RBAC details
  could have shifted there. Affects modules 01, 02b, 03, 04, 05, 06, 07, 08, 09 wherever peat-node is
  cited.
- **Binaries/ports at the audited HEAD:** existence of a `peat-quickstart` binary, `peat-mesh-node`
  production-vs-demo status, `PEAT_BIND_PORT 4040` / `PEAT_HTTP_PORT 8080` defaults, the
  `PEAT_CONNECTION_RECYCLE_SECS` env var and its 60s/5s behavior.
- **Wire/runtime constants not re-located to `path:line`:** exact `SyncMessageType` discriminant byte
  values, connection-health RTT/loss thresholds, the discovery 60 s candidate-pool timeout, specific
  Prometheus metric names and `/api/v1/...` endpoints, stream-ID range *enforcement* at runtime.
- **Quantitative claims that are external specs or design targets (flagged, not asserted):** BLE
  rate/range, LoRa/Iridium SBD figures (datasheet specs, not PEAT measurements); "~10 MB Automerge /
  256 KB / ~40×" memory comparison (256 KB is an unenforced design target; the 10 MB figure has no
  source in code); the "<5 s P1" latency and "1,000+ node" validation (single-machine ~1023-node
  simulation ceiling, epics #724–#727); the "20-node plateau" industry premise (whitepaper thesis,
  no external citation); the 500,000/4,000 vs 9,120/384 connection-model disagreement (~10× apart).
- **Operator-guide config not traced into compiled code:** `[security.pki]` X.509 device-cert flow,
  CoT `bind_port 8087` / type strings, the `command_authority` enum surface — all labeled
  *Documented*, not *Shipped*.
- **Standalone-repo existence/publication:** crates.io / Maven Central publication of any PEAT crate;
  peat-sim, peat-registry, peat-sdk-go, a standalone peat-android repo (only ADR/epic anchors
  corroborate these — ADR-054, #724, Phase-3 roadmap).
- **Spec-defined, implementation unconfirmed:** Automerge `ActorId` byte construction; access-level
  ladder Public(0)…Critical(4) + audit-log retention tiers (90d/1y/2y); partition "original-leader
  keeps Cell ID, merge-negotiate on heal" rule; leader-election **score-over-message** convergence
  (`leader_election.rs:340-345` follows higher node id with an in-code "future work" comment — no
  tracking issue located).

---

## `learning/peat-learning-hub.html` — single-page learning hub

Rewrote the full single-page hub (now 13 sections; added a **6·5 Use Cases** module). Single-page
nav, accent system, and code-walkthrough/checkpoint pedagogy preserved. Verified self-contained
(publishes as an Artifact under CSP, no external fetch); balanced tags (div 151/151, svg 11/11, pre
18/18, section 13/13, table 17/17); every `data-go` target maps 1:1 to a section.

**Structural:**
- **Removed the Mermaid CDN dependency entirely** (the air-gap-breaking violation). Deleted the
  `cdnjs` script and all `renderMermaid` JS. Re-authored all ~11 diagrams as **inline SVG**, each in
  a `.fig` container with its own **Legend** (color/shape key + status note). Renders offline.
- Added a visible **status-badge system** (Shipped / In-flight / Proposed / Speculative) with a CSS
  pill class and an upfront legend card in Module 0; 165 badges applied so no capability is unlabeled.

**Corrected facts:** hierarchy leaf `Node` (fixed the m25 enum table, was "Platform (0)"); the four
identity schemes; FIPS posture (P-256 only, MLS relabeled Proposed, ChaCha20 flagged only); leader
election split into the two real formulas (spec-004 cell-formation 0.30/0.25/… vs mesh-runtime
mobility/CPU/mem/battery, timing constants flagged illustrative); removed invented `CellRole::Support`
in the m25 context and corrected RBAC roles; transports shown shipped-vs-Proposed (QUIC/BLE/lite-bridge
shipped, UDP-bypass a primitive, SBD/LoRa Proposed-only, relay off by default); CDC sinks (NATS+Webhook
shipped, **Kafka In-flight TODO, Redis absent**), OIDC shipped / SAML/CAC unverified, gateway reframed
control-plane-never-in-data-path; numbers ("93–99%" → whitepaper 79%/60–95%/"95%+"; "1,000+" → single-
machine sim ceiling 1023, #724–#727; "<5s P1" and O(n log n) flagged target/analytical); peat-node
added as a real gRPC sidecar (28/27 RPCs, not "25"); vendor names removed (WebTAK/ATAK → CoT consumer);
ADR count corrected to 75; peat-discovery retirement (peat#919) noted.

File: `/Users/bryanbui/code/peat/learning/peat-learning-hub.html`

---

## `learning/peat-constrained-networking.html` — Off the Grid / constrained track

Rewrote the full single-page track in place. Nav, accent system, self-contained inline SVG/CSS
preserved; no external assets. Added a four-state status-pill system + legend in the hero and a `.src`
source-note convention; every section carries at least one label and every factual/quantitative claim
a citation or flag. Verification: tags balanced (13 sections / 13 nav links / 24 SVGs / 102 divs); all
16 JS-referenced IDs resolve; cn9 NODE coordinates (A 95,150 · B 180,210 · H 335,195) match the SVG and
the highlight ring; 12 figures each have a legend.

**Corrections:**
- **Identity (cn1):** killed the fabricated "256-bit NodeId." Now `DeviceId` = first 16 bytes of
  SHA-256 over the Ed25519 key; constrained tiers use a 4-byte u32 `NodeId`. Diagram labels corrected.
- **Transports (cn0/cn4):** SBD (ADR-051) and LoRa (ADR-052) relabeled Proposed (no crate); QUIC/BLE/
  peat-lite Shipped. PACE ladder / `min_pace_level` relabeled Speculative over the shipped
  `preference_order`. BLE "2 Mbps" reframed as the LE-2M PHY symbol rate; all radio numbers flagged as
  datasheet specs (gbrain `quic-iridium-sbd-feasibility` cited); SBD cost corrected to ~$0.04–0.13/msg.
- **Sync (cn3/cn6):** anti-entropy digest schemes (version-vector reading-log, snapshot-since-T,
  hierarchical digest, IBLT) labeled Speculative and contrasted with shipped negentropy-over-QUIC.
  Tombstone GC noted Shipped (peat-node#136, 168h default), enforcement In flight; **mis-cited #857
  corrected** (#857 is ADR-046 Phase-4 selectors).
- **Gateway conflation (cn5/cn7/cn8/cn9):** every data-plane "gateway" renamed "relay/hub/bridge
  node"; reserved "peat-gateway" for the control plane (never in the data path).
- **Encryption (cn2/cn12):** FIPS framing modernized; residual ChaCha20 conflict confined to Proposed
  ADR-052 and stale READMEs (flagged, never propagated); approved-vs-CMVP caveat added (peat-btle#75).
- **Config (cn11):** YAML relabeled illustrative; `command_log` labeled Speculative (does not exist);
  `Operator` RBAC role replaced with `Commander`; autonomy-under-human-authority preserved/strengthened.
- **ADR list (cn12):** all listed ADRs annotated Proposed; ADR-041 noted as the only Accepted one.

File: `/Users/bryanbui/code/peat/learning/peat-constrained-networking.html`

---

## `learning/00-START-HERE.md`

Rewritten in place against ground truth §1. Corrected: **"over any transport"** → "pluggable transport
layer" with an explicit shipped-vs-proposed breakdown (QUIC/Iroh, BLE, peat-lite UDP bridge ship; SBD/
LoRa Proposed-only); **"TCP/IP for autonomy"** kept as framing but de-rated from a delivered SLA;
**"the five repos"** → the actual **six** code repos in a table; **peat-mesh "the node binary"** →
re-attributed to **`peat-node`** (gRPC sidecar, QUIC-only, no BLE); **"256 KB devices"** → design
budget (ADR-035), no allocator cap; **"AI models"** dropped from the connected-systems one-liner;
**gateway in the data path** → control-plane-only; **quickstart "3-node / ~10 min"** → forward pointer
validated in Module 8; added a positive FIPS statement and the four-row status legend, the five use
cases as labeled summaries, `command_log` flagged Speculative, autonomy-under-human-authority framing.

File: `/Users/bryanbui/code/peat/learning/00-START-HERE.md`

---

## `learning/00b-the-big-idea.md` — The Big Idea

Rewritten in place. Added the status-label system + intro legend table (the headline DoD failure: the
module previously had none). **Fixed the hierarchy-vocabulary heads-up** (the single most catchable
error): ADR-066 is *Proposed*, restored the dropped `Platform` tier (5 tiers, not 4), made the rename
mid-flight explicit (#904/#968/#970). Relabeled "trust as data" as Proposed/design-intent (peat#941);
reconciled the 500,000-vs-4,000 figure with the whitepaper's 9,120-vs-384 table (flagged O(n log n)
analytical); re-grounded capability composition as In-flight; added the real `CapabilityType` enum (no
`Weapon`); re-grounded "deep = peat-lite" (codec + bounded CRDTs, no transport/sync engine); added the
three real authority/role axes with the README role-name bug; flagged bandwidth percentages over
measured 79%/60–95%/"95%+"; flagged the "1,000+" validation as a 1,023-node sim ceiling (#724–#727);
marked the 20-node ceiling / standards-paradox arguments as Thesis; strengthened the shipped story
(Automerge + negentropy + deterministic election, FIPS nuance). Verified during this run: 
`peat/spec/draft-peat-protocol-00.md` exists on disk; whitepaper chapters 01–09 + Makefile exist.

File: `/Users/bryanbui/code/peat/learning/00b-the-big-idea.md`

---

## `learning/01-architecture-overview.md` — Module 1

Rewritten in place against ledger `01-architecture.md`, ground truth, and live source. Replaced "any
transport, automatic failover" with the labeled shipped set (QUIC/Iroh, BLE, peat-lite UDP bridge,
HTTP/REST, TAK/CoT TCP); marked simultaneous PACE failover Proposed; `UdpBypassChannel` noted a
primitive, not a `MeshTransport` (`bypass.rs:930`, ADR-042). Softened "256 KB RAM"; restated "1,000+"
as a 1023 sim ceiling (#724–#727). **Hierarchy (highest-risk catch):** shipped leaf is `Node`
(`beacon/types.rs:56-67`, `authorization.rs:331-343`), peat-btle fully legacy (`lib.rs:495-525`), proto
ships `SquadSummary`; ADR-066 Proposed, rename In-flight. Five → **six repos** (added peat-node, gRPC
sidecar, Iroh-only, no BLE); gateway "control plane, not a node, not in the data path." ADR count
"~60" → **75** (triage #695). Verified DEVELOPER_GUIDE exists, §3.1/§3.2 diagrams verbatim, HIVE
expansion at `:46`; QoS pipeline now carries an In-flight enforcement caveat; removed unverifiable date
+ superlatives. Gateway `=rc.1` pin noted intentionally stale (~40 RCs). Dropped unverifiable IWRP
attribution. Vendor "ATAK" → "CoT consumer." Renumbered Try-it 1-2-3-4. Added status-label legend,
the "code > ADRs > specs/READMEs" rule, autonomy framing, legends on all four diagrams, two checkpoint
questions. Verified `peat-mesh-node.rs` exists as a demo binary; production node attributed to peat-node.

File: `/Users/bryanbui/code/peat/learning/01-architecture-overview.md`

---

## `learning/02-peat-protocol.md` — Module 2 (peat-protocol)

Rewritten in place against `peat` HEAD `35d0f11` / `peat-mesh` rc.42, applying all 33 ledger rows.
**Hierarchy enum (Wrong, high-risk):** real shipped enum `HierarchyLevel { Node, Cell, Cohort,
Federation, Coalition }` (`beacon/types.rs:56-67`), leaf `Node`, ADR-066 Proposed, rename documented
(epics #904/#968/#970, ADR-068). **Removed invented `CapabilityType::Weapon`** → real enum
`Sensor/Compute/Communication/Mobility/Payload/Emergent` (`capability.proto:11-18`, weapons under
`Payload`). **CRDT-per-type table** relabeled Speculative/teaching aid (peat-protocol stores one
Automerge family; typed primitives are peat-lite's). **Six-level authority model** → real five-value
`AuthorityLevel` enum (`node.proto:61-67`). QoS labeled Shipped (framework) / In-flight (enforcement).
"Trust as data" relabeled design intent (ADR-048 Proposed, peat#941). Crypto: AES-256-GCM + ECDH-P256
in use, FIPS caveats added, MLS Proposed, ChaCha20 flagged not propagated. RBAC corrected to five real
roles. Vendor names genericized to "CoT/TAK consumer." Autonomy framing strengthened (§2.3 formation
gate, §2.7 authority boundaries). Code-fidelity: line refs corrected; confirmed `CellRole` includes
`Support` (7 variants) by **direct source read** (`models/role.rs:25-26,39`) — this contradicts the
merged ground-truth `peat.md` which omitted `Support`; per decision D1 the source read wins, `Support`
retained. Added Mermaid legend, DeviceId 16-byte caveat, two-layer leader-election clarification.

File: `/Users/bryanbui/code/peat/learning/02-peat-protocol.md`

---

## `learning/02b-formation-and-leadership.md` — Formation & leadership

Rewritten in place against directly-read source (`peat-protocol/src/cell/*`, `models/cell/mod.rs`,
`models/role.rs`, `peat-mesh/src/{security/formation_key.rs,beacon/types.rs}`, proto), resolving nearly
every "Unverifiable" ledger row.

**Corrected catchable errors:** hierarchy leaf was `Platform (0)` → `Node = 0`
(`beacon/types.rs:56-67`); ADR-066 reframed Proposed, rename In-flight, per-repo vocabulary split
spelled out. "Authoritative, current" overstatement removed. CRDT per-field table kept but the
mechanism made precise — OR-Set/LWW/G-Set are `CellStateExt::merge` over the **protobuf** `CellState`
(`models/cell/mod.rs:250-274`); the Automerge backend over Iroh QUIC syncs the encoded bytes (the prior
text conflated the two layers). Code snippet corrected to `CellConfig::new(max_size)`; imports fixed.
Handshake timeout corrected to **30 s** (raised under #373); MAC input confirmed
`HMAC-SHA-256(key, nonce ‖ formation_id)` (`formation_key.rs:162`), constant-time via `subtle`.

**Corrected the ledger itself (direct source overrode the audit summary, decision D1):** `CellRole`
has **7 variants including `Support`** (`role.rs:14-29`) — the ledger flagged `Support` as fabricated;
source confirms it exists, so it was retained. Leadership weights **0.30/0.25/0.20/0.15/0.10 are real
and tested** (`leader_election.rs:100-106`), not fabricated. `LeaderElectionManager`, `CellMessageBus`,
timing defaults (5 s/2 s/3-missed), `CellCoordinator` six gates, `FormationStatus`, `LeadershipPolicy`
(RankDominant/TechnicalDominant/Hybrid/Contextual), `Deputy` (spec-004), `RoutingContext`,
proto summary names, ADR-024 (Accepted) — all verified and relabeled Verified/Shipped with citations.

**Flagged for a human:** score-over-message convergence (`handle_leader_announce` follows higher node
id with an in-code "future work" comment, `leader_election.rs:340-345` — labeled In-flight, no tracking
issue located); peat-node origin delta; `LatestOnly` vs `FullHistory` compaction cited from the
ground-truth model, not re-read line-by-line.

File: `/Users/bryanbui/code/peat/learning/02b-formation-and-leadership.md`

---

## `learning/03-peat-mesh.md` — Module 3 (peat-mesh @ `00ab0c9` / 0.9.0-rc.42)

Rewritten in place. Added the four-label legend and tagged every capability (the module previously had
none).

**Corrected factual errors:** "ADR-061" transitive gossip + "DEVELOPER_GUIDE §6.4.1" — **both
fabricated** (peat-mesh's ADR index runs 0001-0013; no DEVELOPER_GUIDE in the repo); the gossip
behavior is real (`automerge_store.rs:529-541`, `automerge_sync.rs:905-925`) and re-cited to peat#891/
#907. Bandwidth-envelope "contract" table — unsourced, demoted to Speculative; removed the unverified
`max_connections = 7`. "X.509 enrollment" → custom Ed25519-signed `MeshCertificate` wire format
(`certificate.rs:118`). "token bucket per peer" → sized-semaphore frame-byte backpressure
(`sync_channel.rs:208`). `ConnectionState` numeric thresholds removed (only doc-comments). FIPS
"validated" → "approved primitives" + CMVP caveat (peat-btle#75), P-384 absent, ARM perf unbenchmarked
(#126). NodeId vs `DeviceId = SHA-256[..16]` clarified. Hierarchy leaf `Node`, ADR-066 Proposed
(#904/#968). SBD/LoRa marked Proposed-only "no transport registered" (ADR-052 ChaCha20 conflict
flagged). Tombstone GC misattributed "#857" → peat-node#136. `peat-mesh-node` noted reference/demo
binary; production node is peat-node.

**Verified-correct retained:** builder + `new`+`set_*` APIs; `node` feature; `PEAT_FORMATION_SECRET`→
HKDF-SHA-256 (ADR-062); redb backend; full `MeshConfig`; `SyncMessageType` enum/bytes/ADR-034 comments;
all source-layout types; three shipped transports; n0-relay-off default + rc.42 toggle; QoSClass 5
levels. Added legends to both Mermaid diagrams + a worked disaster-response example.

**Flagged:** negentropy "O(log n) rounds" and the Automerge-sync arxiv claim are the algorithms' own
analytical claims, not PEAT benchmarks; the ADR-063 cadence model is the proposal's reasoning;
`ADR-034` is cited in code comments but absent from the umbrella ADR index.

File: `/Users/bryanbui/code/peat/learning/03-peat-mesh.md`

---

## `learning/04-peat-btle-and-lite.md` — Module 4 (peat-btle `3d70f48`/0.4.0, peat-lite `7a8a8fb`/0.2.5, peat-mesh `00ab0c9`/rc.42)

Rewritten in place. Added the four-state status legend + per-capability tags (was entirely unlabeled).
**Removed the vendor name** ("ATAK") from the opening. **`BleConfig` API fixed:** dropped the
non-existent `with_power_profile()` / `with_phy()`; documented the real surface — `peat_lite()` preset
(`config.rs:428`), `power_profile` field (`:398`), `apply_power_profile()` (`:439`), PHY via
`BlePhy`/`coded-phy`. Battery/duty-cycle claims ("18–24h vs 3–4h", "<5% vs 20%+") relabeled uncited
marketing; removed the phantom "UltraLow/36h" profile. `SyncMessageType` path corrected to
`src/gatt/protocol.rs:68-77`. `PeatBeacon` framing clarified (16-byte body in Service Data; full
~40-byte advertisement needs extended advertising). Crypto: FIPS-clean AES-256-GCM/ECDH-P256/Ed25519;
README stale (ChaCha20/X25519); approved-vs-CMVP (peat-btle#75); BLAKE3 addressing-only. Reconnect
re-delivery flagged In-flight (peat-btle#73); iOS/Windows adapters honest In-flight. peat-lite table
rebuilt: `PnCounter` wire-byte + firmware-only Proposed-for-core; `OrSet` Speculative (reserved byte,
no struct); added the shipped `CannedMessageStore`/`CannedMessageAckEvent`; `CannedMessageEvent` size
corrected (22B unsigned / 86B signed, `wire.rs:21-27`). 256 KB → design target (ADR-035). Code snippets
fixed to compile: `LwwRegister::merge(other: Self)`, `GCounter::merge(other: &Self)` rewritten to the
real body (`counter.rs:92-104`). `Document 0x07` envelope: codec is peat-lite's own, only the opaque
body is `peat_mesh::Document`. Added the four identity schemes (no `btle_to_peat_node_id` function
exists), the `command_log` absence note, and that anti-entropy/digest/IBLT/version-vector are
Speculative. Legends on both diagrams; worked remote-sensor-over-constrained-link use case.

File: `/Users/bryanbui/code/peat/learning/04-peat-btle-and-lite.md`

---

## `learning/05-peat-gateway.md` — Module 5 (peat-gateway `8d16824`, crate `0.1.0`, mesh pin `=0.9.0-rc.1`)

Rewritten in place. **Vendor/house-rule fix:** removed "Keycloak / Okta / Azure AD / CAC-SAML" → "any
OIDC-compliant IdP via RFC 7662 token introspection"; SAML/CAC relabeled Proposed/unverified. AWS KMS /
HashiCorp Vault retained only as named backend *products* in the crate (ground truth §7), not in
generic prose. **CDC sinks corrected:** only NATS JetStream and Webhook are Shipped sinks; Kafka
In-flight stub (`engine.rs:78-80` `// TODO`, does not deliver); Redis does not exist. Mermaid diagram
distinguishes the four + carries an inline legend. Route `/healthz` → `/health`
(`api/health.rs:24`). `to_mesh_tier` 3-of-4 caveat (`Edge` never issued) + note that gateway tiers ≠
ADR-066 hierarchy (Proposed, leaf `Node`). RBAC = `u32` bitmask (`RELAY/EMERGENCY/ENROLL/ADMIN`),
distinct from the peat-protocol `Role` enum. FIPS caveat: crate is ChaCha20-free; local-KEK uses
non-CMVP software AES; KMS/Vault is the FIPS-140 boundary; ADR-052 ChaCha20 must not be attributed
here; zeroize gap (#55). Enrollment gaps surfaced: no proof-of-possession (length-only pubkey check,
`enroll.rs:237-285`); `Strict` policy unimplemented (always 403, `:89-93`); admin-token-open hazard.
Ingress posture added (#99/#97/#124/#125). Mesh pin `=0.9.0-rc.1` noted intentionally frozen (~40 RCs
behind rc.42). Status labels + `#![allow(dead_code)]` candor signal. CLI subcommands shown in user-typed
form (`serve`/`migrate-keys`/`load-test`). Delivery semantics flagged Unverified. Added "Where to
contribute" tied to real issues.

**Flagged:** CDC at-least-once egress (no code path found); SvelteKit UI screen inventory (only `/_/`
confirmed); SAML/CAC enrollment (no implementing code).

File: `/Users/bryanbui/code/peat/learning/05-peat-gateway.md`

---

## `learning/06-data-flows.md` — Module 6 (Cross-Cutting Data Flows)

Rewritten in place. **Fabricated enum:** Trace C's `HighestAttributeWins` does not exist → real
`ConflictPolicy` (`command.proto:91-99`): `LAST_WRITE_WINS / HIGHEST_PRIORITY_WINS /
HIGHEST_AUTHORITY_WINS / MERGE_COMPATIBLE / REJECT_CONFLICT`. **Non-existent method:**
`CommandRouter.resolve_targets()` → real `resolve_target` (singular, returns `TargetResolution`,
`command/routing.rs:80`) + `get_routing_targets()` (`:197`). "5 nodes → 1,000+" reframed (routing
invariant Shipped; 1,000 is a 1023 sim ceiling, #724–#727). Added the two distinct deterministic
elections (peat-protocol 0.30/0.25/… vs peat-mesh `dynamic_strategy.rs`); formula quoted verbatim
(`leader_election.rs:101-106`). Identity-continuity note: four schemes, re-derived at the bridge via
`Translator`/`BleTranslator` (ADR-059 codec Shipped, ADR Proposed); no `btle_to_peat_node_id`. QoS class
ordering Shipped; preemption + "<5 s P1" In-flight. Vocabulary: leaf `Node`, rename mid-flight. 256 KB →
ADR-035 design target. Vendor names removed → "CoT consumer." FIPS: Proposed peat-lora ADR-052
ChaCha20-Poly1305 flagged not-FIPS, never propagated. Added status-label key, legends on every diagram,
inline tags, `Document=0x07` detail, BLE reconnect-re-delivery In-flight (peat-btle#73), `command_log`
Speculative note. Verified live: `ConflictPolicy` variants, `resolve_target` singular, `max_retries: 3`
(`cell/messaging.rs:366`).

**Flagged:** discovery 60 s candidate-pool timeout softened (not re-located to `path:line`); negentropy
"O(log n)" is an arxiv claim; 1,000-node / "<5 s P1" / 256 KB are ceilings/targets.

File: `/Users/bryanbui/code/peat/learning/06-data-flows.md`

---

## `learning/07-repo-links-and-gaps.md` — Module 7

Rewritten in place. **peat-node** promoted from an uncertain "maybe a repackaging" to a confirmed
first-class **Shipped** production sidecar (now six repos), with a dedicated explainer: gRPC/Connect/
gRPC-Web on one port, **28 RPCs defined / 27 implemented** (corrected from stale "25"), Iroh-QUIC-only
(no BLE), Helm/Zarf/UDS; distinguished from the `peat-mesh-node` demo binary. **ADR count** "~60" →
**75** (verified `ls peat/docs/adr/*.md | wc -l` = 75; mesh 14, btle 6), #695 anchor. ADR-040 relabeled
(`040-nostr-protocol-lessons.md`; negentropy is the applied lesson; "O(log n)" flagged algorithmic).
Status caveats on ADR-006/011/059/060; ADR-041 named the only Accepted core ADR. DEVELOPER_GUIDE
authority reordered to **code > current ADRs > specs > guide/README** (2025-12-08 snapshot, known
drift). Removed the false "shallow clone / git pull" caveat on FORMATION_AND_LEADERSHIP.md (verified
present). Spec caveats added (all Draft 2025-01-07 except 005-security amended 2026-05-18 to
AES-256-GCM). Dropped unsourced "~47k lines." crates.io / Maven relabeled Unverified. Windows/WinRT BLE
flagged In-flight/untested. **New §7.7 capability-gaps table** (main ledger ask): command_log
(Speculative), targeted delivery (In-flight #853), SBD/LoRa (Proposed), single-burst digest
(Speculative over shipped negentropy), MLS (Proposed, overclaim), QoS enforcement (In-flight), tombstone
GC (In-flight; corrected "#857" → peat-node#136 / #850/#829/#873), control-plane/node auth (In-flight).
**New §7.8 FIPS posture** (approved-vs-CMVP; ChaCha20 isolated to ADR-052/stale READMEs; OSCORE/AES-CCM
the constrained fix; P-384 absent; ARM perf #126). peat-registry ADR-054 anchor verified, status
relabeled unverified.

**Flagged:** crates.io / Maven publication; peat-sim/peat-registry/peat-sdk-go standalone-repo
existence; standalone peat-android repo; the unaudited peat-node origin delta (could move "28/27 RPC"
or land gRPC auth #38).

File: `/Users/bryanbui/code/peat/learning/07-repo-links-and-gaps.md`

---

## `learning/08-running-and-operating.md` — Module 8

Full in-place rewrite, live-verified against `peat`/`peat-mesh`/`peat-node` source. Added a
reader-facing `[Shipped]/[In-flight]/[Proposed]/[Documented]` legend + tags on every section.
`peat-mesh-node` upgraded to **Shipped** (`peat-mesh/src/bin/peat-mesh-node.rs` + `Cargo.toml:164-186`,
`required-features = ["node"]`); binary table distinguishes `peat-quickstart` / `peat-mesh-node` /
`peat-node` / `peat-sim` (Documented). Env-var-per-binary confusion fixed:
`PEAT_FORMATION_SECRET`/`PEAT_BROKER_PORT` confirmed at `peat-mesh-node.rs:47,54`; §8.3–8.8 banner-noted
as the `peat-sim` surface. **mDNS label corrected** `_peat-node._tcp.local` → real `_peat._udp.local`
(`discovery/mdns.rs:23`, quickstart log `main.rs:368`; explains UDP 5353).
`PEAT_CONNECTION_RECYCLE_SECS` citation fixed (#892 runtime-override; #873/#874 mitigation; flagged
that #435 is actually the negentropy issue; 60s default confirmed). **MLS relabeled** Proposed
(ADR-044), not implemented. Crypto rewritten to real FIPS posture (operator guide's ChaCha20 flagged
stale; AES-256-GCM + ECDH-P256 under `aws-lc-rs`; approved-vs-CMVP peat-btle#75; **P-384 dropped**;
ADR-060 Proposed-but-§5-implemented; ChaCha20 not propagated). QoS honesty (classes Shipped, preemption
+ "<5s P1" In-flight). n0 relay default OFF (#833). Tombstone/recovery reframed (Automerge + negentropy;
168h/7-day retention; "#857" → peat-node#136 + #850/#829/#873). Leader-election troubleshooting refined.
Vendor names removed ("CoT consumer"); `command_authority` framed as autonomy-under-human-authority;
gossip-diagram legend added. Internal inconsistencies removed ("~10 min" → ~20 min laptop / ~45 min
with Pi; deterministic-NodeId scoped to "this example binary").

**Flagged Documented (not Shipped):** `[security.pki]` X.509 device-cert flow (`OPERATOR_GUIDE.md:722`
only); CoT `bind_port 8087` / type strings / `command_authority` enum (`OPERATOR_GUIDE.md:931`,
subsystem paths verified but config values guide-documented). Stale-process `nohup` Pi gotcha kept as a
generic-Unix field tip. **Remaining unverifiable:** exact `command_authority` surface and `[security.pki]`
runtime wiring not traced into compiled code.

File: `/Users/bryanbui/code/peat/learning/08-running-and-operating.md`

---

## `learning/09-protocol-specs.md` — Module 9 (Protocol Specs)

Rewritten in place. Added a spec-vs-code contract banner + per-capability labels (none existed).
**Device ID width:** led with the shipped 16-byte `DeviceId` (`SHA-256(pubkey)[0..16]`,
`security/device_id.rs:39-47`); noted spec 001/005 say 32 B; replaced "one derivation everywhere" with
the four real schemes. **RBAC:** spec's `Observer/Member/Operator/Leader/Supervisor` → shipped
`Leader/Member/Observer/Commander/Admin` (`authorization.rs:50-64`); explained Operator/Supervisor leaked
from `AuthorityLevel` (`node.proto:61-66`). **peat-lite & BLE wire:** spec 001 §6/§7 flagged
aspirational; gave shipped peat-lite (`"PEAT"` magic, 16-B header, port 5555, `Document` 0x07, u32
NodeId) and BLE GATT code-of-record `0xA1B2` (spec's `0x1826` is the Fitness Machine UUID). **Leader
election** split into two formulas; human-authority weighting labeled Proposed/spec-layer. **MLS**
relabeled Proposed (ADR-044, no `openmls`/`mls-rs`). **FIPS:** AES-256-GCM/ECDH-P256/Ed25519 under
`aws-lc-rs` Shipped + algorithm-vs-CMVP caveat (peat-btle#75), P-256 only, ADR-060 Proposed. **Stale-
ChaCha20 narrative corrected:** spec 005 was amended 2026-05-18 and is FIPS-clean; repointed the "find
the ChaCha20" exercise to peat-mesh/peat-btle READMEs + ADR-052. **ADR-066** Proposed, leaf `Node`.
**ADR-065** version negotiation labeled Proposed. §9.3 schema: verified package list + `<__peat>`
(double-underscore) tag against spec 003; corrected prose's `<_peat_>`. §9.2: "DQL filters + geo bbox" →
peat-node's actual simple predicate language. Added §9.7 worked use cases (TAK/CoT, UGV tasking with
autonomy-under-human-authority, disconnected-cell reconnect), the verified 7-variant `CellRole`. Vendor
names removed → "CoT/TAK consumer."

Note: ground-truth.md §5 said `CellRole` has no `Support`; direct read of `models/role.rs:25` confirms
`Support` **does** exist (decision D1, source read wins) — module reflects the verified 7-variant enum.

**Flagged:** spec line count "~3,750" (dropped); Automerge `ActorId` byte construction; access-level
ladder + audit retention tiers (likely Phase-3); stream-ID range *enforcement*; partition Cell-ID rule;
`EdgeIntelligence` / "≥3 sensors" threshold (dropped/softened); peat-node origin delta.

File: `/Users/bryanbui/code/peat/learning/09-protocol-specs.md`
