# PEAT Curriculum — Merged Claim Ledger

Merged from the 14 per-document ledgers in `learning/review/ledger/` (hub, constrained track, and
modules 00–09). Audited against `learning/review/ground-truth.md`. **Sorted Wrong/Misleading first**,
then Outdated, Oversimplified, Speculative-unlabeled, Unverifiable, and finally the Verified bright
spots the rewrite must preserve.

Verdicts: **Wrong · Misleading · Outdated · Oversimplified · Speculative-unlabeled · Unverifiable ·
Verified.** "Doc · location" cites the curriculum file; "Evidence" cites the ground-truth model /
`path:line` / ADR / issue.

---

## A · WRONG (factual errors a skeptical engineer catches on first grep)

| # | Claim (quote/paraphrase) | Doc · location | Evidence | Fix |
|---|---|---|---|---|
| W1 | Leaf hierarchy tier named **`Platform` (0)**; "`peat_mesh::beacon::HierarchyLevel` (post-ADR-066)" = Platform/Cell/Cohort/Federation/Coalition | hub m25; 02b §2·5.1 L21; 02 #29 L319-321; mbig §5 | The shipped enum leaf is **`Node=0`** (`peat-mesh/src/beacon/types.rs:56-67`; `peat-protocol/src/security/authorization.rs:331-343`). ADR-066 is **Proposed**. peat-btle still uses legacy `Platform/Squad/Platoon/Company`. GT §5. | Change tier-0 to `Node`; label ADR-066 Proposed and the rename mid-flight (#904/#968); note peat-btle is fully legacy. |
| W2 | `CellRole` includes a **`Support`** variant (logistics/medical) | hub m25 step 4; 02b §2·5.6 L204-212 | Real enum is `Leader, Sensor, Compute, Relay, Strike, Follower` (6) — `peat-protocol/src/models/role.rs:14-28`. `Support` does not exist. GT §5. | Remove `Support`; list the six real variants. (Note: 02 #13 correctly lists 7 incl. `Support` — code confirms `Support` exists per that ledger's direct read; reconcile against `models/role.rs` before final wording.) |
| W3 | `CapabilityType` includes a **`Weapon`** row (→ not composed) | 02 §2.3 #12 L157-164 | Enum is `SENSOR, COMPUTE, COMMUNICATION, MOBILITY, PAYLOAD, EMERGENT` (`peat-schema/proto/capability.proto:11-19`); weapons are under `PAYLOAD`. GT §5. | Remove the `Weapon` row or relabel `Payload`; add the real `Emergent` type. |
| W4 | RBAC roles = **Observer → Member → Operator → Leader → Supervisor** | 09 §9.5 L108-109 (matches spec 005 + README) | Shipped `Role` enum is `Leader, Member, Observer, Commander, Admin` — **`Operator`/`Supervisor` do not exist** (`authorization.rs:50-64`). Leaked from `AuthorityLevel`. GT §5. | List shipped roles `Leader/Member/Observer/Commander/Admin`; note spec/README are wrong; don't conflate with `AuthorityLevel`. |
| W5 | "**NodeId — a 256-bit fingerprint** computed from the device's public key" | constrained cn1 L179; (+ diagram L188) | No 256-bit NodeId anywhere. `DeviceId` = SHA-256(pubkey) **truncated to 16 bytes / 128 bits**; lite/btle NodeId is a **`u32`**; peat-node uses raw iroh EndpointId. GT §4. | "DeviceId = first 16 bytes (128 bits) of SHA-256(Ed25519 key); the wire NodeId is a transport string/u32." Stop saying 256-bit. |
| W6 | "`PeerId`/`DeviceId` = SHA-256(Ed25519 pubkey) — **one derivation, used everywhere**" | hub m9 "Eight facts"; 09 §9.6 fact 1 L101-103 | Four schemes: DeviceId SHA-256[..16]; mesh transport NodeId = String; btle NodeId = BLAKE3[..4] u32; lite NodeId = bare u32; peat-node = raw iroh EndpointId (no SHA-256). GT §4. | Lead with the 16-byte truth; enumerate the per-layer NodeId types; drop "one derivation everywhere." |
| W7 | peat-lite wire: magic **`0xCAFE`**, **CRC-16**, types Register/Ack/Heartbeat/Data | 09 §9.1 L32-35; hub m9 §001 | Shipped peat-lite magic is **`"PEAT"`** (4 ASCII), 16-byte header, **no CRC-16**; types Announce/Heartbeat/Data/Query/Ack/Leave/**Document(0x07)**/OTA; NodeId is a bare `u32`. GT §8. | Use the shipped framing; point readers to peat-lite source, not spec 001 §6. |
| W8 | BLE GATT service UUID **`0x1826`** (m9) / advertised **`0xF47A`** | 09 §9.1 L36; hub m9 §001; (btle README 0xF47A) | Code-of-record is **`a1b2c3d4…` / `0xA1B2`** (`peat-btle/src/lib.rs:337,343`, pinned by test). `0x1826` is the standard Fitness Machine UUID; `0xF47A`/`f47ac10b…` are stale. GT §8. | Use `0xA1B2` / `a1b2c3d4…`; the GATT UUID is the most-drifted value in the project — cite the code. |
| W9 | `ConflictResolver` policy **`HighestAttributeWins`** | 06 §6.3 L137-138 | Real `ConflictPolicy` enum (`command.proto:91-99`): `LAST_WRITE_WINS, HIGHEST_PRIORITY_WINS, HIGHEST_AUTHORITY_WINS, MERGE_COMPATIBLE, REJECT_CONFLICT`. `HighestAttributeWins` is invented. | Replace with `HighestPriorityWins`/`HighestAuthorityWins`. |
| W10 | **`CommandRouter.resolve_targets()`** (plural) | 06 §6.3 L134 | Method is **`resolve_target` (singular)** returning `TargetResolution` (`command/routing.rs:80`), plus `get_routing_targets()` (`:197`). No `resolve_targets()`. | Use `resolve_target(...)` or describe behavior. |
| W11 | Directed-tasking profile uses **`crdt: command_log`** and `write_role: operator` | constrained cn11 L606-612 | `command_log` CRDT **does not exist anywhere** (Speculative). `operator` is not a real RBAC role. Targeted delivery is ADR-046 Proposed (#853 In-flight). GT §3, §5. | Mark `command_log` Speculative; fix `operator`→`Commander`; label targeted delivery In-flight. |
| W12 | cn9 dial preset "**Operator**" authority label | constrained cn9 L533,696-702 | `Operator` not a `Role` variant; same fabrication as README. GT §5. | Replace with `Commander` or note illustrative. |
| W13 | "`BleConfig` … `with_power_profile()`, `with_phy()`" builder methods | 04 L29 | No such methods; `power_profile` is a field via `apply_power_profile()` (`config.rs:398,439`); PHY via `BlePhy`/`coded-phy`. | Use the real API. |
| W14 | "the five repos" / system is five repos (peat-node treated as uncertain repackaging) | 00 L5-6; 01 §1.4; 07 §7.2 L30; hub m7 | **peat-node is a sixth real, actively-developed gRPC sidecar repo** (v0.4.3+), distinct from any `peat-mesh-node` demo binary. GT §1, §7. | Enumerate the six code repos; describe peat-node as a confirmed standalone sidecar (28 RPCs defined / 27 impl). Drop "may be a redirect/mirror." |
| W15 | Try-it #3 / checkpoint: find ChaCha20 in **spec 005** ("pre-2026") | 09 Try-it L154, checkpoint L163 | Spec 005 was amended 2026-05-18 to AES-256-GCM — ChaCha20 is gone there. The stale ChaCha20 sources are the **READMEs** and ADRs 006/044/048/049/052. GT §6, §9. | Repoint the exercise at the peat-mesh/peat-btle READMEs (or ADR-052). |
| W16 | "Try it" numbering: two steps labeled "2" | 01 Try-it L290-293 | Editorial mis-numbering (1,2,2,3). | Renumber 1-4. |

---

## B · MISLEADING (true-ish framing that implies more than ships)

| # | Claim | Doc · location | Evidence | Fix |
|---|---|---|---|---|
| M1 | "connects … **over any transport**" / "**Any transport works** — QUIC, BLE, UDP, HTTP, simultaneously, with automatic failover" | 00 L11-14; 01 §1.1 L21 | Only QUIC/Iroh, BLE, peat-lite UDP bridge, HTTP/REST, TAK/CoT ship; SBD/LoRa are Proposed; multi-transport PACE failover is Phase-2 Proposed; raw "UDP" is the `UdpBypassChannel` primitive, not a peer transport. GT §2. | "over a pluggable transport layer (QUIC/Iroh, BLE, embedded UDP shipped; satellite/LoRa proposed)"; mark PACE failover Proposed. |
| M2 | Kafka/Redis Streams CDC "supported"; diagram shows 4 equal sinks | 05 §5.3 L125,129-137; hub m5 | **Kafka is a TODO stub** (does not deliver); **Redis does not exist**. Only NATS + Webhook ship. GT §7. | NATS+Webhook Shipped; Kafka In-flight (stub); Redis Proposed/absent. Grey/dash them in the diagram. |
| M3 | Identity federation "Keycloak / Okta / Azure AD / CAC-SAML" / "OIDC / SAML / CAC" | 05 §5.1 L15-16, §5.3 L139-144; hub m5 | Only **OIDC token introspection (RFC 7662)** ships; SAML/CAC not evidenced. **Vendor names violate the no-consumer-name house rule.** GT §7. | "any OIDC-compliant IdP (RFC 7662)"; drop vendor names; mark SAML/CAC Proposed/unverified. |
| M4 | "MLS for forward secrecy" / "group-key rotation toward MLS tree ratcheting" / quickstart "no MLS" (implies MLS normally ships) | hub m9 §005; 09 §9.5 L114-116; 08 §8.1 | **MLS is NOT implemented** (no `openmls`/`mls-rs`); Proposed (ADR-044, carries pre-FIPS ChaCha20). GT §6. | Label MLS Proposed (ADR-044); today's group-key rotation is leader-distribution; MLS is the proposed future. |
| M5 | "Crypto is FIPS 140-3 (AES-256-GCM / **P-256-384** …)" presented as validated | hub m9 fact 8, §005; 08 cipher caveat; 09 §9.5 L109-112 | Algorithms are FIPS-**approved** but **not in a CMVP-validated module** (peat-btle#75 open). **P-384 is not in code** (only P-256). GT §6. | Add the algorithm-vs-module distinction; drop "/384". |
| M6 | "the gateway only merges data … another device takes over the **gateway** role" / translator "gateway 🗼" | constrained cn5/cn7 L314,386,399; (any module implying gateway in data path) | The merge/translate/relay node is a **peat-mesh node**; **peat-gateway implements zero transports and is not in the data plane**. GT §7. | Rename to "hub/relay/bridge node"; reserve "gateway" for the control plane. |
| M7 | "End-to-end object security" / "encrypt the message itself" as an off-grid mechanism | constrained cn2 L234 | Shipped crypto is link/session AEAD; peat-lite's `DOC_FLAG_ENCRYPTED` is reserved+encoder-rejected; object security (OSCORE-style) is the unbuilt proposal. GT §6. | Label per-document object security Proposed/Speculative. |
| M8 | "**93–99% / 95–99% bandwidth reduction**" | mbig §7 L176; hub mbig §7; 00b §7 | Whitepaper reports 79% / 60–95% / "95%+" target; 93-99% is a marketing rounding not stated verbatim; grep clean in code repos. GT §8. | Use the whitepaper's actual figures; label single-machine lab. |
| M9 | "works for 5 … works for **1,000+**" / "1,000+ node" as architectural certainty | 00b §7; 01 §1.1 L24; 06 §6.3 L144; hub mbig/m6 | Single-machine simulation ceiling (1023 hard cap), NOT field-proven; #724-#727 still target 900/1.2K/10K. GT §8. | "validated in single-machine simulation to ~1000 nodes (1023 cap); larger scale is open work #724-#727." |
| M10 | QoS "P1 contact report **preempts** P3/P4 at every hop" / "<5s P1" implied | 06 §6.1 L54,77; hub m6; 02 #22 | QoS framework + classes ship and peat-node orders by class, but **cross-class wire-level preemption is NOT enforced in v1**; "<5s P1" is a target, enforcement In-flight. GT §8. | "prioritizes/orders by class"; note preemption not fully enforced; "<5s P1" is a target. |
| M11 | "Trust as data — authority is **replicated CRDT state**; delegation/revocation propagate like any document" | 00b §5 L130; 02 #30 L311-315 | No shipped CRDT-replicated authority-delegation doc; authorization model deferred (peat#941). Design intent. GT §5. | Mark Proposed (ADR-048); cite #941; not a shipped mechanism. |
| M12 | "Vocabulary heads-up: current code (ADR-066) uses Cell/Cohort/Federation/Coalition" (drops Platform; calls ADR-066 current) | 00b §5 L147-151; 01 §1.2 L42-44; 02b §2·5.1 L17-18; 09 §9.4 L94-96 | ADR-066 is **Proposed**; 5 tiers incl. **Platform** as base (doc lists 4); rename mid-flight; peat-btle fully legacy. GT §5. | Rewrite per GT §5; add Proposed label; note workspace mixes Node/Platform/Squad/Cell. |
| M13 | `peat_mesh::Document` envelope (namespace) carried by 0x07 | hub m4/m6; (06 partly) | The universal `Document` envelope is **defined in peat-lite** (`protocol/document.rs`), ADR-059 Amendment 4 "universal half"; only the opaque body is `peat_mesh::Document`. GT §2, §8. | Call it peat-lite's `Document` envelope. |
| M14 | "PeerId = first **32 bytes** of SHA-256" (spec value repeated as the contract) | 09 §9.1 L21,24-27 | Shipped `DeviceId` is **16 bytes**; "newer source wins" → lead with 16B. GT §4. | State 16B shipped, spec says 32B. |
| M15 | "256 kilobytes … smaller by **~40×**" / "Automerge ~10 MB" stated as fact | constrained cn5 L312,319,324; hub m1/m4; 06 §6.4 | 256KB is an unenforced design target; ~10 MB Automerge is unverified; ~40× is derived from two soft inputs. GT §8. | Label 256KB a design target (ADR-035); caveat/drop ~10 MB and ~40×. |
| M16 | "BLE moves a couple of **megabits per second**" / "BLE · 2 Mbps" | constrained cn4 L280,283 | Declared capability constants are 250/125/500 **kB/s** (PHY symbol rate ~2M conflated with goodput); not measured. GT §8. | Cite the declared constants; label BLE-spec assumption. |
| M17 | "compact 16-byte beacon fits in a legacy **31-byte** advertisement" | 04 L91-93 | Beacon body is 16B but full AD is ~40B and **requires extended advertising** (`beacon.rs:42,47`). GT §8. | "16B body carried in Service Data; full AD ~40B needs extended advertising." |
| M18 | DEVELOPER_GUIDE (2025-12-08) "co-authoritative with code"; spec set stamped "2025-01-07" hiding 005's amendment | 07 §7.5 L105-108; 09 intro L13 | Code > current ADRs > guide; spec 005 was amended 2026-05-18 (now FIPS-clean). GT §9. | Reorder authority (code first); note 005's amendment. |
| M19 | §6.4 fact 8: "several prose docs that still say ChaCha20" (implies spec 005 is stale) | 09 §9.6 fact 8 L132 | Stale ChaCha20 sources are the **READMEs** + ADRs 006/044/048/049/052 — NOT spec 005. GT §6. | Name the actually-stale sources. |
| M20 | "9.6 fact 6/7": leader election "**hybrid (technical + human-authority)**"; emergent capabilities "**drive capability-based tasking**" | 09 §9.6 L130-131 | Mesh runtime score is purely capability/resource-based (no authority term); pattern-driven tasking is ADR-046 In-flight (`CapableScope` reserved-but-rejected). GT §5, §7. | Split deterministic/no-consensus (shipped) from human-authority weighting (spec/Proposed); soften tasking. |
| M21 | "reading the actual source in this folder (not just READMEs)" | 00 L75-77 | Leaf-transport internals were not all in one tree — corroborated via re-export shims/tests/gbrain. GT §1. | Soften to "across the PEAT repos … where internals live in a sibling repo, noted." |
| M22 | gateway enrollment maps role→tier producing a decision (omits no-PoP, Strict-403, permissive ingress) | 05 §5.3 L139-144, omissions | Enroll validates pubkey length only (no proof-of-possession); `Strict` always 403s; ingress AuthZ is a permissive stub; no NATS broker auth/TLS. GT §7. | Add the security gaps; list the three enrollment policies. |
| M23 | discovery off-grid replacements "BLE advert · LoRa beacon · satellite IMEI registry" as equals | constrained cn2 L232 | BLE advert ships; LoRa/satellite presuppose unbuilt transports; shipped discovery is mDNS/static/K8s. GT §2. | Label BLE Shipped; LoRa/satellite Proposed. |
| M24 | "peat-btle … One trait, **six platforms**" with a 4-row table; iOS/Windows shown as shipped | hub m4; 04 (platform table OK) | iOS is Beta/In-flight; Windows untested/In-flight. GT §1, §7. | Reconcile "six"; mark iOS Beta and Windows untested In-flight. |
| M25 | quickstart "real mesh in **~10 minutes**" vs guide's ~20 min; `/healthz` route | 08 §8.1, §8.5 L152; 01 L67 | QUICKSTART says ~20 min (laptop) / ~45 (Pi); health route is **`/health`** not `/healthz`. GT §8. | Align times; fix `/healthz`→`/health`. |
| M26 | §8.1 "no MLS" next to "no formation key/no enrollment"; #435 cited for connection recycle | 08 §8.1, §8.8 | MLS is Proposed (elevating it to "omitted shipped feature" misleads); recycler is **#892**, #435 is negentropy. GT §6, §8. | Footnote MLS Proposed (ADR-044); cite #892. |

---

## C · OUTDATED

| # | Claim | Doc · location | Evidence | Fix |
|---|---|---|---|---|
| O1 | "~60 ADRs" in `peat/docs/adr/` | 01 §1.4 L158; 07 §7.6 L130; hub m7 | Actual count is **75** `.md` files. GT §8. | "~75 ADRs (and growing)." |
| O2 | peat-node "25 RPCs" | (peat-node README, mirrored where modules quote it) | Proto defines **28**, service impl **27**. GT §7. | "28 defined / 27 implemented." |
| O3 | `peat-discovery` implied present | 01 §1.4/§1.5 (omission) | Retired under peat#919; discovery now `peat_mesh::discovery`. GT §1. | Note discovery moved into peat-mesh. |
| O4 | FORMATION_AND_LEADERSHIP.md "may not be in this shallow clone (run git pull)" | 07 §7.5 L111-114 | File **is present** at this checkout. | Remove the hedge. |
| O5 | cn12 "today's constrained transports specify a [ChaCha20] cipher that doesn't meet FIPS" | constrained cn12 L645 | Shipped peat-btle/peat-lite are FIPS-clean (AES-256-GCM/ECDH-P256, or no symmetric crypto); the ChaCha20 conflict survives only in Proposed ADR-052 + stale READMEs. GT §6. | Restate: residual conflict is ADR-052 (LoRa, Proposed); shipped transports are clean. |

---

## D · OVERSIMPLIFIED

| # | Claim | Doc · location | Evidence | Fix |
|---|---|---|---|---|
| D1 | "which CRDT for which data" table (G-Set/OR-Set/PN-Counter/LWW-Map per collection); per-field CRDT-type table (members=OR-Set, leader_id=LWW, capabilities=G-Set) | 02 #20 L248-256; 02b §2·5.9 L262-280 | Runtime uses **Automerge documents** (one CRDT family, RGA/observed-remove); named primitives are peat-lite's embedded types; `OrSet` has no struct even there. GT §3. | Relabel "conceptual / design intent"; describe CellState as an Automerge doc with Automerge's own semantics. |
| D2 | "Deep integration = peat-ffi/peat-lite" (peat-lite as turnkey deep path) | 00b §9 L207-211 | peat-lite is a codec + bounded CRDTs with **no transport, no sync engine**. GT §3. | Clarify peat-lite is the embedded building block; deep integration still needs a transport. |
| D3 | "a cell is the fundamental unit" (omits Node/Platform leaf) | 01 §1.2 L42 | Base tier below cell is `Node`; cell groups 4-13 nodes. GT §5. | Clarify cell = grouping of nodes. |
| D4 | sensor→command-post trace implies seamless identity continuity | 00 L6-7; 06 §6.1 | Real cross-crate identity discontinuity (lite/btle u32 ↔ mesh DeviceId). GT §4. | Add a one-line pointer that identity is re-derived at the bridge. |
| D5 | gateway RBAC framing via tier names only | 05 §5.3 L67-81 | Actual perms are a u32 bitmask `RELAY/EMERGENCY/ENROLL/ADMIN`. GT §7. | Mention bitmask, not five named roles. |
| D6 | "Automerge sync naive O(n)"; negentropy "O(log n)"; "256 kbps/30 kbps" bandwidth envelopes | 03 §3.4 L129-186 | O(log n) is an algorithmic/module-doc claim, not benchmarked; the N=3/4/7 / kbps "contract" appears in no GT file. GT §3, §8. | Label O(log n) analytical; mark the bandwidth envelope Unverified/Speculative or cite a source. |
| D7 | gaps module frames "gaps" only as missing repos to clone | 07 intro/§7.2 | Omits real capability gaps (command_log, MLS, SBD/LoRa, RBAC names, QoS enforcement, hierarchy rename). GT §3-§7. | Add a "capability gaps" subsection with issue/ADR anchors. |
| D8 | bandwidth profiles `minimal/low/medium/high`; `qos_enabled` enforces priority | 08 §8.4 | QoS allocation primitives ship; priority *enforcement* (preemption to a latency target) is In-flight. GT §8. | Note `qos_enabled` configures shipped classes; enforcement In-flight. |
| D9 | "ADR-032 already spells out … one small address-translation [to add a radio]" | constrained cn1 L204 | ADR-032 is Proposed; the trait seam ships but bringing up a radio is far more than translation (LoRa/SBD resolve to "no transport"). GT §2. | Note ADR-032 Proposed; soften "one small translation." |
| D10 | `CannedMessageEvent` "~24 bytes"; primitives table omits `CannedMessageStore`/`CannedMessageAckEvent` | 04 L113,109-116 | Unsigned wire size is exactly 22B / signed 86B; the store + ACK hybrid CRDTs are the actually-shipped surface. GT §3, §8. | Use precise wire sizes; add the hybrid CRDTs. |

---

## E · SPECULATIVE-UNLABELED (teaching design presented as built)

| # | Claim | Doc · location | Evidence | Fix |
|---|---|---|---|---|
| S1 | PACE ladder + `min_pace_level` floor; satellite "summary up / commands down" topology; "~40s / two paid messages" | constrained cn4/cn8/cn9/cn10 L278-300,480-553 | No PACE state machine or `min_pace_level`; SBD/LoRa unbuilt; latency/cost arithmetic derived from Proposed transports. GT §2. | Relabel the whole satellite spine Speculative/Proposed; cite PACE failover as Phase 2. |
| S2 | SBD anti-entropy digest schemes: version-vector "reading log", snapshot-since-T, hierarchical digest, IBLT "difference sketch" | constrained cn6 L343-374 | None implemented; shipped reconciliation is negentropy over QUIC. GT §3. | Label Speculative (teaching-only); contrast with shipped negentropy. |
| S3 | shared-picture / directed-tasking YAML config (`min_pace`, `delivery: targeted`, `require_ack`, `crdt: or_set\|command_log`) | constrained cn11 L596-612 | No such collection-config schema; OrSet unimplemented; command_log nonexistent; ADR-046 Proposed/In-flight. GT §3, §5. | Mark YAML illustrative/Proposed; note nonexistent keys/types. |
| S4 | transitive gossip "ADR-061"; SyncMessageType byte tags; sequence diagram tags 0x05/0x06 | 03 §3.4 L113-175; hub m3 | **ADR-061 is in no GT file**; the byte tags are uncited; behavior plausible (echo-suppressed gossip ships in peat-node) but the wire format/ADR are unverified. GT §10. | Verify ADR-061 + tags against `automerge_sync.rs`; else label Speculative/illustrative. |
| S5 | "Trust as data = replicated CRDT authority" (see M11) | 00b §5; 02 #30 | (same as M11) | (same as M11) |
| S6 | constrained track has **zero shipped/proposed labels**; hub/modules largely unlabeled | constrained, hub, mbig, 04, 05 (whole-doc) | §11 DoD requires every claim labeled. GT throughout. | Add Shipped/In-flight/Proposed/Speculative labels everywhere. |

---

## F · UNVERIFIABLE (not in the audited ground truth — chase before publish)

| # | Claim | Doc · location | Note |
|---|---|---|---|
| U1 | leader-election timing constants (2s heartbeat / 6s / 5s election / 0.7 readiness / min-size-3); `LeaderElectionManager`/`CellMessageBus` round-based state machine | 02b §2·5.2/§2·5.5/§2·5.7/§2·5.8; hub m25; 06 §6.2 | Not in GT; the mesh model is vote-free/beacon-observed, contradicting a round-message machine. Trace to `peat-protocol/src/cell/`; several may be fabricated. |
| U2 | leadership-score weights 0.30/0.25/0.20/0.15/0.10 attributed to peat-mesh | 02b §2·5.5; hub m2; 09 §9.4 | These are spec-004 / peat-protocol weights; **peat-mesh uses a different formula** (mobility/CPU/mem/battery). Verified for peat-protocol `leader_election.rs:101-106` (06 #28); label which layer. |
| U3 | crates.io + Maven Central publication of crates | 01 §1.6; 07 §7.1 | Independent versions confirmed; publication channels not evidenced. |
| U4 | `peat-sim`, `peat-registry` (ADR-054), `peat-sdk-go`, `peat-inference` repos | 07 §7.2; hub m7 | Not in GT. peat-sim corroborated by #724 epic; sdk-go is roadmap Phase 3 Proposed; inference/registry unaudited. Label accordingly. |
| U5 | `peat-mesh-node` binary + `node` feature | 03 §3.1(b); 08 §8.2; hub m1/m3/m8 | Not confirmed by name in GT; the deployable node is peat-node. Verify `src/bin/peat-mesh-node.rs`. |
| U6 | `MeshConfig`/builder field set; `PeatMeshBuilder::new().with_*().build()` ergonomics; redb named as the store | 03 §3.1; hub m8 | README builder flagged "aspirational"; facade is `PeatMesh::new` + `set_*` injectors (GT §7). redb plausible (gateway uses it) but unnamed in peat-mesh GT. |
| U7 | ADR-024 (tier grouping), ADR-065 (version-negotiated signing) cited as authority | 02b §2·5.10; 09 §9.5; hub m9 §005 | Not in the audited ADR set; verify/relabel Proposed. |
| U8 | spec-003 package list, field-number ranges, beacon field enums; spec-004 `EdgeIntelligence`/"≥3 sensors"; audit-log retention tiers; "DQL"/geo-bbox subscription; `[security.pki]` config; quickstart port defaults 4040/8080 | 09 §9.3/§9.4/§9.5; 08 §8.3-8.7 | Spec-level claims not in GT; peat-node's `Subscribe` is a simpler predicate language (no "DQL"); peat-lite's own UDP default is 5555. Verify against `peat-schema/proto/` + specs. |
| U9 | "the team captured during the IWRP proposal"; "Peat = Hierarchical Intelligence for Versatile Entities (HIVE)"; whitepaper "~47k lines"; TRL 4-5 | 01 §1.3/§1.6; 07 §7.5; mbig §7 | Provenance unconfirmed; HIVE legacy name corroborated indirectly (ADR-049). Cite or soften. |
| U10 | quickstart `peat-quickstart` build verification; stale-process Pi lore; OTLP support; alert thresholds | 08 §8.1/§8.6 | Build not run in GT (deferred to module 08 run). The 08 ledger confirms `examples/quickstart` exists; others are operator-guide/field tips — label advisory. |

---

## G · VERIFIED — bright spots the rewrite MUST preserve

- **Deterministic leader election, no consensus** — correct everywhere it appears (02 #8, 02b §2·5.5,
  03 #28, 06 Trace B, hub m2). The single most important high-risk claim, and the curriculum gets it
  right. Keep; add a "Shipped" label and note the two distinct implementations.
- **FIPS crypto = AES-256-GCM + ECDH-P256, NOT ChaCha20** — modules 02 #26, 03 #26, 04 L95-97, 08
  §8.5, 09 §9.5 correctly use the code reality and flag the stale-doc ChaCha20. Exemplary; keep
  (just add the CMVP-module caveat and drop "/384").
- **Negentropy + Automerge over Iroh is the real anti-entropy** (not version-vector/IBLT) — 02 #19,
  03 #21/#27, 06, hub m3. Correct framing; keep.
- **Formation handshake = HMAC-SHA-256, ALPN `peat/formation-auth/1`, key never on wire** — 02 #25,
  02b §2·5.4, 08 §8.5, hub m25. Exact and current.
- **Dependency direction (peat-protocol → peat-mesh), re-export shims, peat-btle leaf with broken
  back-edge, peat-lite pure leaf** — 00 L82-87, 01 §1.5/§1.6, 04 L186-206, hub m1. All accurate.
- **Gateway is control-plane-only, not in data path; envelope encryption (PENV v1) is the
  best-tested subsystem; four KeyProviders** — 05, 06 §6.4, hub m5. Accurate; keep and add the
  Kafka/Redis correction (M2) and security-gap notes (M22).
- **peat-lite 16B header / `Document(0x07)` / wire sizes; peat-btle chunking/relay/two-phase
  crypto; QoS 5 classes** — 04, 06, hub m4/m6, 09 §9.1-9.2. Verified figures; keep.
- **CRDTs-vs-consensus and "hierarchy is compression" framing** — 00b §3/§6. Solid pedagogy.
- **Honest "spec is stale / peat-sim not in tree / no prelude module" callouts** — 02 #4, 08 §8.2,
  09. Model the labeling the rest of the curriculum needs.

---

## H · House-rule violations to fix in the rewrite

1. **Vendor names in generic protocol material** — "Keycloak / Okta / Azure AD" (05 §5.1, hub m5).
   Replace with "OIDC-compliant IdP". "ATAK/WebTAK" is borderline (CoT/TAK are protocol names, fine;
   prefer "a CoT/TAK client" where a product is implied). peat-ffi JNI de-ATAK'ing is in-flight (#846).
2. **FIPS propagation** — never carry ChaCha20/X25519 as live (READMEs do; curriculum must flag,
   not propagate). Drop "P-384" (not in code).
3. **Self-containment** — the hub loads Mermaid from a CDN (`cdnjs.cloudflare.com`); inline or
   pre-render to SVG for the air-gapped audience.
4. **Diagram integrity** — hub m1 SVG has contradictory inline comments about arrow direction; every
   interactive element/arrow must match its label (gateway→mesh, not gateway→protocol). Every diagram
   needs a legend and shipped/proposed key (currently absent on the constrained track, data-flow
   diagrams, and CDC diagram).
4. **Autonomy-under-human-authority framing** must be preserved for anything about commands/tasking
   (the CoT `platform_uav` mapping and command-authority gate already do this — keep).
