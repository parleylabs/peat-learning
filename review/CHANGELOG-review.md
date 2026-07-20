# Peat Curriculum Review — Change Log

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
  rate/range, LoRa/Iridium SBD figures (datasheet specs, not Peat measurements); "~10 MB Automerge /
  256 KB / ~40×" memory comparison (256 KB is an unenforced design target; the 10 MB figure has no
  source in code); the "<5 s P1" latency and "1,000+ node" validation (single-machine ~1023-node
  simulation ceiling, epics #724–#727); the "20-node plateau" industry premise (whitepaper thesis,
  no external citation); the 500,000/4,000 vs 9,120/384 connection-model disagreement (~10× apart).
- **Operator-guide config not traced into compiled code:** `[security.pki]` X.509 device-cert flow,
  CoT `bind_port 8087` / type strings, the `command_authority` enum surface — all labeled
  *Documented*, not *Shipped*.
- **Standalone-repo existence/publication:** crates.io / Maven Central publication of any Peat crate;
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
analytical claims, not Peat benchmarks; the ADR-063 cadence model is the proposal's reasoning;
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
aspirational; gave shipped peat-lite (`"Peat"` magic, 16-B header, port 5555, `Document` 0x07, u32
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

---

## 2026-06-19 — Incremental refresh (CI, delta-driven)

First scheduled incremental on top of the 2026-06-18 full sweep. Three of six repos drifted; the
refresh re-audited and rewrote only the affected docs (and their diagrams). No full sweep due
(`last_full_run` 2026-06-18, < 30-day interval).

**Repos moved (old → new):**
- **peat** `35d0f11 → 68e9c3c` — workspace `0.9.0-rc.25 → rc.26`; +ADR-071 (Proposed) and its
  additive Phase-1 convergence seam (#991); distribution impl re-exported after relocation (#993);
  peat-mesh dependency floor raised to `>=0.9.0-rc.43`.
- **peat-mesh** `00ab0c9 → 71fc3d5` — `rc.42 → rc.43`; **provider gossip** (`peat/blob-announce/1`
  ALPN, #262), the **relocation** of blob/file-transfer distribution from peat-protocol into
  peat-mesh (peat#992, "canonical iroh consumer"), and a sync fix registering inbound-accepted peers
  into `known_peers` (#261).
- **peat-node** `4e1b5c8 → bbe3b68` — `v0.4.3 → v0.4.7`; attachment **outbox watcher** for
  bidirectional hands-off file sync (#167, env `PEAT_NODE_ATTACHMENT_OUTBOX_WATCH`, off by default),
  a **30 s peer-status heartbeat** logging `connected_peers` vs `known_peers` (#169), default log
  filter widened to cover the whole sync stack, and an empty-env startup-crash fix (#164).

peat-btle, peat-lite, peat-gateway: **no drift**.

**Docs changed.** Version/commit stamps updated across `00b`, `01`, `02`, `03`, `04`, `05`, `07`
and both HTML tracks + `changelog.html`. New labeled content:
- **Module 3 §3.4b (new)** — "Blob distribution & provider gossip": the relocation to peat-mesh, the
  `DistributionScope` targeting table (`AllNodes`/`Nodes` work; `Formation`/`Capable` are
  reserved-but-stubbed — warn + fall back to all `known_peers`, `resolve_targets:873`), provider
  gossip (**Shipped**: ALPN, `DEFAULT_ANNOUNCE_TTL=3`, re-announce-on-acquire, `classify_announce`
  trust gating), a new mermaid (**M-038**), and the **ADR-071** interest-driven-convergence seam
  (**Proposed**; `NeedEvaluator`/`CollectionSubscriptionNeed`/`with_need_evaluator` present but inert,
  publish still writes `collection: None`).
- **Module 2** — facade note that `storage/file_distribution` + `model_distribution` are now
  re-export shims, pointing at Module 3.
- **Module 8** — peat-node v0.4.7 ops note (outbox watcher, peer-status heartbeat, sync-stack logging;
  ties `known_peers` to delivery targeting and the #261 fix).
- **Module 9** — distinguishes the **durable** ADR-071 subscription from the existing `Subscribe`
  change-stream predicate.
- **index.html** — two new hub mesh cards (blob distribution / provider gossip; ADR-071) mirroring
  §3.4b; **peat-constrained-networking.html** — §07 note on multi-hop blob reach.

**House rules.** Provider gossip labeled **Shipped** (code + e2e tests); ADR-071 labeled **Proposed**
with the seam called out as Shipped-but-inert; relocation cited to peat#992; FIPS/crypto untouched; no
vendor names introduced; autonomy framing untouched.

**Resolved.** Prior open item "peat-node 5 commits ahead (v0.4.4/v0.4.5 untested)" — now audited:
RPC count held at **27**, gRPC auth #38 did **not** land, tombstone/RBAC unchanged.

**New flags (unverifiable / follow-on).** ADR-071 convergence path wired but never exercised
end-to-end (publish writes `collection: None`); provider-gossip multi-hop reach read from source +
e2e test names, not independently benchmarked; outbox watcher gate/poll behaviour not exercised by a
live run (read-only audit). New `open_todos` track ADR-071 collection plumbing and Phases 2–3.

---

## 2026-06-22 — incremental refresh (CI mode)

**Repos moved:** `peat` `68e9c3c → 8a94796` (workspace 0.9.0-rc.26 → rc.27), `peat-gateway`
`8d16824 → bece4d6` (crate still 0.1.0), `peat-node` `bbe3b68 → 9fcdabd` (v0.4.7 → v0.4.8).
`peat-mesh` (71fc3d5), `peat-btle` (3d70f48), `peat-lite` (7a8a8fb) — no drift.

**Headline correction — gateway mesh pin.** The most consequential fact change this cycle:
`peat-gateway`'s `peat-mesh` pin jumped `=0.9.0-rc.1 → =0.9.0-rc.40` (Dependabot peat-gateway#144).
The curriculum's prior "intentionally frozen, ~40 RCs stale by design" narrative is now wrong — the
gateway lags the ecosystem (rc.43) by only ~3 RCs and is being kept current via Dependabot. Corrected
in **Module 1 §1.5** (prose + M-006 mermaid edge label), **Module 5 §5.6**, and the **hub** gateway
card. The bump was real integration work (adapted the CDC watcher to the new `ChangeEvent` `origin`
field + `redb` 2→4), which we now cite as evidence for the "tracking the mesh is real work" caveat
rather than the obsolete "frozen by design" framing.

**Other changes:**
- **Module 8** — replaced the v0.4.7 ops box with **v0.4.8**: inbox now mirrors the sender's outbox
  layout with path re-sanitisation + `<distribution_id>.bin` fallback (#173), startup version banner,
  `--print-config` / `PEAT_NODE_PRINT_CONFIG`, post-write `blob_size` validation. Added a note that
  the rc.27 `relay-n0-hosted` facade-forwarding fix makes enabling the flag at `peat-protocol`/`peat-ffi`
  actually flip the relay posture (was a silent no-op).
- **Module 2** — facade callout now flags **ADR-072 (Proposed)** as the synced-folder lifecycle
  follow-on to ADR-071, and documents the rc.27 `relay-n0-hosted` forward fix + `connect_peer_nowait`.
- **Module 3 §3.4b** — added the **ADR-072 (Proposed)** publisher-declared lifecycle/handling policy
  paragraph atop the ADR-071 seam.
- **Module 00b** — peat-ffi deep-integration note gains `connect_peer_nowait` (Shipped, rc.27).
- **Module 9** — unchanged this run (ADR-071 wording already current).
- Version/commit stamps advanced across Modules 00b/1/2/5/7, `index.html`,
  `peat-constrained-networking.html`, `changelog.html` (Current-sync card + new history row).

**Diagrams.** M-006 (Cargo dependency graph) re-derived — edge label changed from "stale by ~40 rc"
to "rc.40 (~3 rc behind)". Spot-checked + advanced to 2026-06-22: H-003 (twin SVG; arrow facts
unchanged, no rc label in the SVG), M-027/H-010 (gateway CDC sink set unchanged — `src/cdc/engine.rs`,
the sink modules, and `CdcSinkType` in `src/tenant/models.rs` untouched), M-033 + M-035 (peat-node
moved but proto unchanged, RPC count still 27).

**House rules.** ADR-072 and ADR-059 ("Data as a Capability") both labeled **Proposed** (no code);
`connect_peer_nowait` and the relay forward labeled **Shipped** with `path:line`. FIPS/crypto
untouched; no vendor names; autonomy framing untouched.

**New flags / follow-on.**
- **ADR-059 number collision** (NEW open_todo): peat#997 added `docs/adr/059-data-as-a-capability.md`,
  but `docs/adr/059-cross-transport-document-bridging.md` already owns ADR-059 (the one the curriculum
  cites for the peat-btle "Amendment 4" cycle-break). Existing ADR-059 citations remain correct
  (cross-transport bridging); the new ADR is logged but deliberately not woven into prose to avoid
  conflating the two. Watch for the source repo to renumber one of them.
- **Unverifiable (read-only audit):** ADR-072 / ADR-059 are Proposed with no code; `connect_peer_nowait`
  and the v0.4.8 inbox layout / size check were read from source + CHANGELOG, not exercised live.

## 2026-06-29 — incremental refresh (CI mode)

**Repos moved:** `peat` `8a94796 → 871776d` (workspace `0.9.0-rc.27 → rc.28`; `peat-ffi` `0.2.8 → 0.2.9`,
Maven AAR `0.1.3 → 0.1.4`), `peat-mesh` `71fc3d5 → c863d16` (no version change; still `rc.43`),
`peat-gateway` `bece4d6 → 4d82282` (still `0.1.0`), `peat-flutter` `369f05d → 4a6554f` (still `0.0.1`).
`peat-btle`, `peat-lite`, `peat-node`, `peat-sapient` did not move. No open `curriculum-feedback` issues.
No full sweep due (`last_full_run` 2026-06-18, < 30 days).

**`peat-ffi` mobile-resilience surface (peat#1000) — Shipped.** All change in this peat delta is additive
on the FFI binding; **no `peat-protocol`/`peat-schema`/`peat-mesh` source change** in the umbrella repo.
- Persistent **peer roster** (`RosterStore`, plain JSON, non-secret reachability hints — **no FIPS
  concern**): six new UniFFI exports (`roster_remember/upsert/remove/get/list/list_by_group`) + a
  `RosterEntry` Record (`peat-ffi/src/roster.rs:35,57`; `lib.rs:2533-2569`).
- Per-peer **reconnect supervisor**: `Idle → Connecting → Connected → Backoff` with exponential backoff
  (`BACKOFF_BASE_MS=2_000`, `BACKOFF_MAX_MS=300_000`, `supervisor.rs:23,25`), per-peer jitter, ≤8 concurrent
  dials, cross-transport dedup (union of iroh + BLE-reachable peers). Three exports
  (`reconnect_known_peers/wake_reconnect/on_peer_observed`, `lib.rs:2477,2490,2511`).
- **Origin-tagged `DocumentChange`**: new `origin: ChangeOrigin` (`Local`/`Remote { peer_id }`, `lib.rs:608`),
  so consumers can notify only on remote deltas. → Module 00b (deep-integration note), Module 6 (data-flow note).

**`peat-mesh` Android mDNS interop (peat-mesh#266) — Shipped.** Peat-controlled `_peat._udp` advertise +
browse mirroring the advertiser (`from_formation_with_discovery_at_addr`, `src/network/iroh_transport.rs`)
for Android, where iroh's own `MdnsAddressLookup` browse never fires; formation-identity iroh-key derivation
via `derive_iroh_node_secret` = `HKDF-SHA-256(salt=None, ikm=formation_secret, info="iroh:"+node_id)` (ADR-049,
FIPS-approved). No new transport/wire/strategy. → Module 3 §3.3. Discovery diagrams (M-017/M-018/H-006)
spot-checked — strategies unchanged, rows advanced.

**`peat-gateway` security dep bump (peat-gateway#151) — Shipped.** `async-nats` `0.38 → 0.49` to clear
`rustls-webpki` CVEs; **mesh pin (`=0.9.0-rc.40`) and CDC sink set unchanged**; NATS ingress gaps (#99/#97/
#124/#125) unchanged. → Module 5 §5.6 + hub gateway card. M-027/H-010 spot-checked, unchanged.

**`peat-flutter` (peat-flutter#17) — Shipped.** Reconnect-supervisor facade (`reconnectKnownPeers`/
`wakeReconnect`/`onPeerObserved`) + relay-URL fix on `endpointAddr` (`lib/src/peat_node.dart`). **BLE
non-FIPS published-vs-source split (C1) UNCHANGED** — `rust/Cargo.lock` still pins `peat-btle 0.4.0`
(bundling `chacha20poly1305`/`x25519-dalek`); only unrelated transitive minors moved. → ground-truth/peat-flutter.md.

**Gates:** all §0b gates run (fact-check, house-rules, cohesion, diagram, blast-radius, published-artifact/
reference, fact-wide-occurrence). FIPS posture unchanged across the run; no ChaCha20/X25519 reintroduced into
prose; the published-vs-source split re-confirmed for peat-btle 0.4.0.

**New unverifiable claims:** the `peat-ffi` reconnect supervisor + roster + origin-tagging and the peat-mesh
`_peat._udp` Android browse are **read from source, not exercised live** (NEEDS_RUNTIME) — backoff timing,
cross-transport dedup behaviour, and Android browse delivery are not benchmarked here.

---

## 2026-07-06 — incremental refresh (CI mode)

**Repos moved:** `peat` `871776d → 2778bd9` (workspace `0.9.0-rc.28 → rc.29`; `peat-ffi` `0.2.9 → 0.2.10`),
`peat-mesh` `c863d16 → b410d7c` (`rc.43 → rc.45`), `peat-node` `9fcdabd → 5df3130` (`v0.4.8 → v0.4.9`),
`peat-sapient` `8ec24d4 → 5118afa` (still `0.1.0`; major restructure). `peat-btle` `3d70f48 → bcfa954`,
`peat-gateway` `4d82282 → 1da5002`, `peat-flutter` `4a6554f → 291d7e4` all advanced **CI/toolchain-only** (no
curriculum-affecting source change — heads advanced, no doc edit routed). `peat-lite` did not move. **No open
`curriculum-feedback` issues.** No full sweep due (`last_full_run` 2026-06-18, < 30 days).

**peat (rc.29) — Shipped, additive.** UniFFI blob surface with a **poll-based `blob_fetch_start`** returning an
`Arc<BlobFetchHandle>` you poll via `status()` (`peat-ffi/src/lib.rs:3358` on; `Some(peer)` → direct
`fetch_blob_from_peer`, `None` → mesh-sync `fetch_blob`; peat#1017) → Module 6. Android mDNS: nullable
`bindAddress` + formation-identity through the `createNodeJni` JNI (**breaking JNI arity**, peat#1006) and a
peat-controlled `_peat._udp` browse consumer that dials discovered peers (`sync/automerge.rs:2682`, peat#1007)
→ Module 3 §3.3. Dropped the `[patch.crates-io]` git pin on peat-mesh and raised the floor to **rc.45**
(peat#1016). `cot/event.rs` quick-xml 0.37→0.41 (RUSTSEC-2026-0194/0195) is behaviour-equivalent. **ADR count
78→79** (ADR-073 peer-ejection Rayfish review, **Proposed, no code**, no change to ADR-056); **11 Accepted
unchanged** (re-derived across both `**Status:**`/`**Status**:` formats). The `peat-tak-bridge` example
**moved to a standalone `peat-tak` repo** (peat#1020) — dead curriculum links fixed in Modules 2 & 6.

**peat-mesh (rc.45) — Shipped.** `fetch_blob_from_peer` — direct, pull-only, single-peer, **no-fallback**
fetch that bypasses the candidate list + health filter (`storage/iroh_blob_store.rs:1497`, peat-mesh#274), and
`add_peer_from_hex_id` (register a peer by id only, `:1347`, #270) → Module 3 §3.4b + hub + constrained track.
The rc.44–rc.45 mDNS follow-ups (peat-mesh#268) make Android browse **dialable** (advertise hex EndpointId, dial
discovered peers, drop loopback/link-local, self-respawning browse daemon) → Module 3 §3.3. Crypto posture
unchanged (`derive_iroh_node_secret` still HKDF-SHA-256; no ChaCha20/X25519).

**peat-sapient (major restructure) — Shipped.** Now a **3-crate workspace** (`peat-sapient` core codec +
`peat-mesh-sapient` mesh adapter + `peat-sapient-bridge` deployable binary consuming `peat-tak`). Ships
**opt-in TLS** (default-off; plain TCP still supported) for the SAPIENT DLMM↔HLDMM link and the TAK-server link
(mTLS) — **flips the prior "plain TCP, no TLS" flag**. FIPS caveat kept precise: `fips_crypto_provider()`
restricts cipher suites to AES-GCM but leaves `aws_lc_rs::default_provider()` kx_groups intact so **X25519 is
still offered**, and it is the standard provider, **not** the aws-lc-rs *fips* module — a **source-level** note
(all crates 0.1.0, unpublished). SAPIENT reaches TAK/CoT via the shared mesh `tracks` collection, not a direct
converter. Rewrote Module 7 §7.1 (repo row) + §7.8 (FIPS note); ground-truth `peat-sapient.md` delta supersedes
the old "peat-schema only / plain TCP" facts.

**peat-node (v0.4.9) — no new runtime surface.** `grpc_test` port-collision fix (test-only) + a zero-friction
two-node attachment quick-start (docs/examples). `proto/sidecar.proto` untouched — RPC count still 27/27.
Module 8 heading advanced to v0.4.9 with a one-line note; capability facts unchanged.

**Diagrams:** M-006/H-003 (dep graph — gateway edge label rc.40 `~3 → ~5` behind, node/edge set otherwise
version-independent; SVG twin carries no rc label), M-017/M-018/H-006 (discovery — dialable/durable is a
property of the same browse path, no new strategy), M-038 (`fetch_blob_from_peer` is a parallel direct path,
gossip facts hold), M-027/H-010 (gateway CI-only) all re-derived and advanced to 2026-07-06. peat-sapient's
3-crate/TLS restructure has no current diagram (prose only; logged as a backlog candidate).

**New unverifiable claims (NEEDS_RUNTIME):** `blob_fetch_start`/`BlobFetchHandle` poll semantics + the
`fetch_blob_from_peer` transfer (feature-gated e2e, not benchmarked here); the Android mDNS discovered-peer
dial + self-respawning browse daemon recovery; and the peat-sapient TLS handshake's **actually-negotiated
key-exchange group** (X25519 is offered by construction but selection needs a runtime capture). Maven AAR
version for rc.29 could not be confirmed from the tree (no bump in range) — logged, not asserted.

## 2026-07-13 — incremental refresh (CI mode)

Delta-driven sync on the latest model, multi-agent (orchestrator + five read-only per-repo ground-truth
research agents + the §0b validation gates: independent fact-check, house-rules, cohesion, diagram,
blast-radius, published-artifact/reference, fact-wide-occurrence). No full sweep due (last_full_run
2026-06-18, 25 days < 30). **No open curriculum-feedback issues.**

**Repos moved (5 of 8):**
- **peat** `2778bd9 → cb6a818` (workspace rc.29 → rc.30; peat-ffi 0.2.10 → 0.2.11).
- **peat-mesh** `b410d7c → b86c2c2` (rc.45 → rc.47).
- **peat-node** `5df3130 → 7942be5` (v0.4.9, dependency-only — no version bump).
- **peat-flutter** `291d7e4 → 129c74c` (0.0.1; re-pins published peat-ffi 0.2.10).
- **peat-sapient** `5118afa → 93d51ac` (0.1.0; +12 commits).
- peat-btle / peat-gateway / peat-lite: no drift.

**Headline facts (all code-verified, path:line in the per-repo ground-truth deltas dated 2026-07-13):**
- **iroh reached 1.0 stable** — peat-mesh moved `iroh` from `=1.0.0-rc.1` to `1.0.2` (peat-mesh#276), and
  peat-node followed. The "iroh is pre-1.0" framing is retired across Modules 1/3/8 + hub. Knock-on: iroh
  1.0's pull-based address lookup forced `connect_by_id` to resolve via `address_lookup()` before dialing
  (peat-mesh#299).
- **New Shipped `fleet/{id}/{kind}` prefix QoS classifier** (peat-mesh#293) — maps fleet collections to
  QoS class / sync mode / deletion policy / propagation by naming convention (Module 2 §2.6, full table).
- **Store-bounding [Shipped]** — write coalescing (200 ms), adaptive compaction (live only once wired into
  `start_sync`, peat-mesh#296/#297), byte-bounded LRU cache (~4 MiB), bounded sync-receive (Module 3 §3.4).
- **peat-schema `Kinematics` + `PositionError`** on Track/NodeState [Shipped]; `velocity`/`cep_m`/
  `vertical_error_m` now `[deprecated]` but still populated by consumers (dual-write) — Module 9 §9.3.
- **ADR-074 (Proposed) "peat-schema single source of truth"** — first migration tranche Shipped: peat-ffi
  records shrunk to proto-backed fields, `MarkerInfo`/`CommandInfo` + `get_markers`/`get_commands` removed
  (Module 6). Labelled precisely: ADR Proposed, first code step Shipped.
- **peat-flutter Dart client** now ships a real blob (`blobDownload` poll-stream) + marker (OR-Set
  soft-delete tombstone) surface against published peat-ffi 0.2.10 (Module 6). C1 non-FIPS-published-btle
  caveat unchanged.
- **peat-sapient**: peat-schema lag closed (rc.24 → rc.30), peat-tak 0.0.2 → 0.0.3, git-dep patches dropped,
  BSI Flex 335 v2.0 **interop** compliance CI added (not crypto-FIPS), operator guide, `--tak-protocol`
  encoding flag. TLS/X25519 crypto caveat **unchanged** (connection.rs empty diff); the `tls` feature is now
  compile-time default-on in the bridge crate but runtime TLS stays opt-in — Module 7 §7.8.
- **ADR count 79 → 80** (`ls docs/adr/*.md` = 80; 76 numbered incl ADR-074 + 4 reference). **11 Accepted
  unchanged.** CellRole still 7. **FIPS source posture unchanged** across all repos.
- Gateway now lags the mesh ecosystem (rc.47) by ~7 RCs (rc.40 pin, gateway did not move this cycle).

**Docs touched:** Modules 01, 02, 03, 06, 07, 08, 09; `index.html` (audited-against box, gateway lag ~7,
doc-fix rc.47, four new hub mesh cards, SYNC stamp); `peat-constrained-networking.html` (SYNC stamp);
`changelog.html` (Current-sync card + 8 tags + SYNC block + prepended row); `review/ground-truth/{peat,
peat-mesh,peat-node,peat-flutter,peat-sapient}.md` (2026-07-13 deltas); `review/diagrams.md` (M-006/H-003,
M-017/M-018/H-006, M-038 advanced; maintenance note); this file; `REVIEW-STATE.json`.

**New unverifiable / NEEDS_RUNTIME this run:** the rc.46–rc.47 store-bounding RSS figures (~930 MB → <60 MB,
OpTree 250–300×) are field-profile numbers, not benchmarked here; `dialer_resolves_acceptor_by_id_via_mdns`
is CI-verified only; the peat-flutter marker-binding forward-incompat with peat-ffi 0.2.11 (bindings will
fail the UniFFI checksum on the next re-pin) is logged as an open todo. peat-sapient's actually-negotiated
TLS key-exchange group still needs a runtime handshake capture (X25519 offered by construction).

---

## 2026-07-20 — full sweep (monthly, CI mode)

Monthly full sweep (previous full sweep 2026-06-18; 32 days elapsed > 30-day interval). Every prose
claim and **every diagram row (Phase 6b)** re-derived against current code. Four repos moved; peat-btle,
peat-gateway, peat-lite, peat-sapient unchanged.

**Repos:** peat `cb6a818→a1ce620` (rc.30→rc.31; `peat-ffi` 0.2.11→0.2.12), peat-mesh `b86c2c2→fa5c403`
(rc.47→rc.49), peat-node `7942be5→23a2707` (v0.4.9→v0.4.10), peat-flutter `129c74c→1770bc9` (0.0.1→**0.1.0**,
first real release).

**Headline — TAK/CoT transport removed from `peat-transport` (peat#1015, `492fb54`).** `peat-transport/src/tak/**`
(3861 deletions across 17 files) deleted; `peat-transport` is now HTTP/REST only. The TAK/CoT TCP bridge migrated
to the standalone `peat-tak` repo. CoT *translation* stays in `peat-protocol/src/cot/` (unchanged). This fact
recurred in **9 files + 3 diagram twins** and was propagated everywhere (fact-wide occurrence gate): Modules
00/00b/01/02/06/08/09, `index.html` (crate card + H-002 SVG node + layer ASCII + Trace-A ASCII + shared-facts),
Module 07 (new `peat-tak` row in §7.2). Diagrams re-derived: M-003/H-002 (layer model, peat-transport → HTTP/REST
only), M-006/H-003 (dep graph node label), M-028/M-029 (Trace A CoT-bridge crate → peat-tak), M-032 (system arch).

**peat-flutter forward-incompat RESOLVED (peat-flutter#29).** The 2026-07-13 open_todo — hand-maintained marker
bindings would fail the UniFFI checksum on the peat-ffi 0.2.11 re-pin — is closed. peat-flutter now pins
`peat-ffi =0.2.12` with `default-features = false`, **owns the Dart `FFIBuffer` adapter itself**
(`rust/src/dart_ffi.rs`), and keeps `MarkerInfo`/`CommandInfo` as Flutter-owned JSON DTOs. This is the
consumer-repo migration peat-ffi's new default-on `dart-ffi` feature (peat#1030/#1031, rc.31) was built for.
Module 6 rewritten accordingly.

**New Shipped (peat-mesh):** on-disk file vacuum `redb::Database::compact()` (rc.49/#300/#301 — `automerge.redb`
observed at 14.5 MB/90 min without it), an IPv6 reachability probe + ULA exemption (rc.48/#304/#305), and retry
of partially synced distributions (rc.49/#307). All in Module 3 §3.4 + hub cards. **New Shipped (peat-node):**
relay-fanout starvation fix (#189); mesh consumption lag closed (pins rc.49 = mesh HEAD). Module 8.

**Codex migration:** `peat/CLAUDE.md → AGENTS.md` (FIPS hard-rule intact, `AGENTS.md:25-42`); SKILL.md → `.agents/skills/`.

**Stable anchors re-derived (direct `ls`+`grep`, not trusted from prior pass):** ADR count **80** (76 numbered),
**11 Accepted**, ADR-074 **Proposed**, `CellRole` **7**, peat-node RPC **27/27**, peat-mesh floor `>=rc.45`. FIPS
source posture unchanged across every diff. C1 published-vs-source non-FIPS BLE split (peat-btle 0.4.0, checksum
`a57dd351`) unchanged. Gateway lags mesh **~9 RCs** (rc.40 vs rc.49).

**§1c fold-in of `peat-tak` attempted, BLOCKED:** the repo is not reachable in this environment (clone routed to the
parleylabs-scoped git proxy and 403'd; egress-proxy `curl` also 403s GitHub API). Cannot audit it against code, so
it is **not** folded into the tracked-clone set / routing tables — added as a §7.2 "referenced, not audited" row and
kept as an open_todo with the clone-access blocker noted.

**Verified-then-wrong miss caught this sweep (fact-wide occurrence):** Module 2 §2.8 still stated `peat-tak =0.0.2`
while Module 7 and the 2026-07-13 run_log said 0.0.3; the consumer pin `peat-sapient/peat-sapient-bridge/Cargo.toml:22`
confirms `=0.0.3`. Corrected. Logged to the §9b retrospective (misses_found=1).

**New unverifiable / NEEDS_RUNTIME this run:** the rc.49 redb file-vacuum shrink efficacy (14.5 MB figure is a
device-profile observation from the commit message, not benchmarked here); the rc.48 IPv6 reachability-probe
behaviour; rc.49 partial-sync distribution retry under a lossy link; peat-node relay-fanout starvation fix under
load; peat-flutter 0.1.0 live-device blob/BLE behaviour. All code-confirmed, none exercised live.

---

## 2026-07-20 — prompt-amendment (issue #18, human-approved)

Not a curriculum refresh; no code repos audited and no reader-facing prose changed. This is a
**tighten-only methodology amendment** filed by the monthly retrospective (issue #18) and applied
after human approval.

**What changed.** The single monotonic `quality_trend.unverifiable_count` is split into two lines
that must not be conflated:

- **`needs_runtime_count`** — claims code-confirmed from source but not benchmarkable in the cloud
  env (NEEDS_RUNTIME / declared-constant / not-measured-here). **Expected to grow** as new Shipped
  capabilities land each run; **neutral, not a health signal.**
- **`unresolved_drift_count`** — `misses_found` (verified-then-wrong) plus claims genuinely
  unconfirmed against code at audit time (incl. external/registry unknowns). **The health line;
  should trend DOWN.**

**Why.** `unverifiable_count` climbed 8 → 9 → 11 → 13 (→ 14 per the 07-20 retrospective) across five
runs while `misses_found` stayed 0 — the entire rise was new-capability NEEDS_RUNTIME, yet a single
count read as the loop failing to learn. The 06-29 / 07-06 / 07-13 `quality_trend` notes each asked
for exactly this carve-out.

**Retrofit.** All five `quality_trend` snapshots (2026-06-26 … 2026-07-20) were re-expressed in the new
shape from the `unverifiable_claims` list: `unresolved_drift_count` held flat at **3** (the
external/registry/standalone-repo unknowns) the whole window, while `needs_runtime_count` grew **5 → 6
→ 8 → 10 → 11** — making explicit that the health line never moved. No count total changed (each split
sums to the former `unverifiable_count`). This supersedes the `unverifiable_count: 14` / `status:
"proposed"` values the same-day full sweep (above) recorded: the amendment it *filed* is now *applied*.

**Files touched:** `PROMPT-peat-curriculum-refresh.md` §6, `PROMPT-peat-training-doc-improvement.md`
§9b (both the miss-signal gather step and the trend-record step), `REVIEW-STATE.json`
(`quality_trend` shape retrofit + `self_improvement.proposed_amendments` entry flipped
proposed → `applied (human-approved this session)`), and this file. Longer-term follow-up (unchanged,
still open): a standing cloud runtime-verification harness so NEEDS_RUNTIME items can be *retired*
rather than only accumulated.
