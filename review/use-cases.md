# Peat — Concrete End-to-End Use Cases

Five worked use cases, each grounded in shipped code (`learning/review/ground-truth.md`) and
explicitly labeled **Shipped / In-flight / Proposed / Speculative** leg by leg. Every transport,
collection, hierarchy placement, and number is tied to ground truth; where a leg is not implemented
it says so. Numbers that are external hardware specs or design targets are flagged, not presented as
Peat measurements.

**Shared facts used below (all from ground truth):** shipped transports are QUIC/Iroh, BLE (peat-btle,
`bluetooth`), peat-lite UDP bridge (`lite-bridge`), HTTP/REST, TAK/CoT TCP. peat-lite wire: 16-byte
header, `MAX_PACKET_SIZE = 512`, `MAX_PAYLOAD = 496`, `Document = 0x07`, `DEFAULT_PORT = 5555`,
multicast `239.255.72.76`. CRDT engine = Automerge over Iroh, reconciled by negentropy. QoS 5
classes (Critical=1…Bulk=5); `commands`/`contact-reports`/`alerts` → Critical, `cells`/`nodes` →
High, `tracks`/`beacons`/`platforms` → Normal. Hierarchy leaf is `Node` (not `Platform`); ADR-066 is
Proposed. peat-gateway is a control plane, never in the data path. SBD/LoRa are Proposed-only.

---

## Use case 1 — TAK/CoT operator picture (mostly **Shipped**)

**Scenario.** Dismounted teams carry phones; a vehicle runs a peat-node sidecar; a CoT/TAK client at
the command post consumes the common operating picture.

- **Devices.** Android phones (peat-btle AAR via UniFFI); a vehicle compute node running peat-node
  (gRPC sidecar) co-located with a CoT/TAK client; the command-post TAK client.
- **Transports.** Phones ↔ phones over **BLE** (`bluetooth`, **Shipped**); phone ↔ vehicle node via a
  BLE→QUIC bridge (the peat-mesh `bluetooth` translator, ADR-059 codec **Shipped**, ADR itself
  Proposed); vehicle node ↔ command post over **QUIC/Iroh** (**Shipped**); command post bridged to the
  TAK client over **TAK/CoT TCP** (peat-transport `src/tak/`, ADR-020/028/029, **Shipped**).
- **Collections / profiles.** `tracks` (Normal QoS), `contact-reports` (Critical), `beacons` (Normal).
  Tactical streaming profile 256 KiB/30 s.
- **Hierarchy placement.** Each phone/vehicle is a `Node`; a dismounted team forms a `Cell` (4–13
  nodes, deterministic leader election); the command post observes `CohortSummary` roll-ups.
- **Data flow.** Phone publishes a position to `tracks` (Automerge doc) → BLE-syncs within the cell →
  the cell leader's vehicle node bridges it to the mesh over QUIC (negentropy reconciles, deltas
  carry) → peat-transport's CoT layer encodes it as Cursor-on-Target XML (MIL-STD-2525 type mapping +
  `<_peat_>` extension, ADR-028) → the TAK client renders the icon. A contact report is Critical and
  is ordered ahead of position updates at congested hops (**but cross-class wire-level preemption is
  not enforced in v1 — In-flight**; "<5 s P1" is a target).
- **Worked example.** A position update is a small Automerge change (tens of bytes of delta). Over BLE
  at the declared 250 kB/s capability constant (a BLE-spec figure, *not* a Peat measurement) the
  intra-cell hop is sub-millisecond on the wire; the QUIC hop to the command post adds ~1–2 RTT after
  the formation handshake. The CoT encoding is a fixed XML transform. **Shipped end to end**, with the
  caveat that QoS preemption and the "<5 s" SLA are not yet enforced.

---

## Use case 2 — UGV / robot joining a cell and being tasked (**Shipped** join; tasking **In-flight/Proposed**)

**Scenario.** A ground robot powers on, discovers a nearby cell, authenticates, is assigned a role,
and receives a task.

- **Devices.** A UGV running peat-node (Iroh QUIC only — **no BLE in peat-node**); peers in the cell
  (other nodes / a leader).
- **Transports.** **QUIC/Iroh** (**Shipped**). Discovery via mDNS, static peering (`PEAT_NODE_PEERS`),
  or Kubernetes EndpointSlice (**all Shipped**); deterministic iroh identity via
  `HKDF-SHA256(shared_key, "iroh:"+node_id)` (**Shipped**, 0.4.3).
- **Collections / profiles.** `nodes` (High QoS), `cells` (High), `commands` (Critical), `tracks`
  (Normal). QoS-priority relay fanout drains Critical first.
- **Hierarchy placement.** The UGV is a `Node`; on join it becomes a `Member` of a `Cell` (or `Leader`
  if its deterministic score wins). Capability assignment via `CellRole` (`Sensor/Compute/Relay/
  Strike/Follower` — **note: no `Support` variant**).
- **Data flow (join — Shipped).** Discovery yields candidates → `connect_and_authenticate(peer_id)`
  over QUIC (3-attempt/200 ms-backoff retry around peat#759) → **formation HMAC-SHA-256 handshake**
  over ALPN `peat/formation-auth/1` (pre-shared formation key, key never on wire) → membership
  granted, role determined by deterministic capability scoring → the UGV starts Automerge sync of the
  cell's collections.
- **Data flow (tasking — In-flight/Proposed).** A command is published to the `commands` collection
  (Critical QoS). Today it is an **ordinary JSON document** (`PutCommand`, `sidecar.proto:342-373`) —
  **there is no `command_log` CRDT** (Speculative). Targeted delivery to a specific node/role is
  **ADR-046 (Proposed), epic #853 (In-flight)**; `CapableScope` distribution is reserved-but-rejected
  in peat-node v1. The command's conflict policy (if two issuers collide) uses the real
  `ConflictPolicy` enum (`LAST_WRITE_WINS / HIGHEST_PRIORITY_WINS / HIGHEST_AUTHORITY_WINS /
  MERGE_COMPATIBLE / REJECT_CONFLICT`). **Autonomy-under-human-authority:** a mission-critical
  capability requires human approval at formation (`CellCoordinator`, readiness ≥ 0.7).
- **Worked example.** Join: discovery window (operator-guide default order of seconds) + 1 QUIC RTT +
  one HMAC handshake (30 s timeout, completes in ms on a good link). Tasking: a command JSON doc is a
  few hundred bytes; it syncs as an Automerge change. The *targeted* delivery that would send it only
  to `role:strike` is **not shipped** — today it propagates to the cell and the receiver filters.

---

## Use case 3 — Maritime / beyond-line-of-sight relay (relay legs **Shipped**; satellite leg **Proposed**)

**Scenario.** A surface vessel relays a coastal sensor cell's summary to a shore command post when
the line-of-sight link drops, falling back to satellite.

- **Devices.** Coastal sensor nodes; a vessel acting as a relay/bridge node; a shore command post; a
  (proposed) Iridium SBD modem on the vessel.
- **Transports.** Sensor↔vessel and vessel↔shore over **QUIC/Iroh** when in range (**Shipped**); the
  beyond-line-of-sight fallback to **Iridium SBD is Proposed (ADR-051) — no crate, no code.** A LoRa
  long-range leg is likewise **Proposed (ADR-052)**.
- **Collections / profiles.** `tracks` (Normal), `cells`/`CohortSummary` (High), `alerts` (Critical).
  Edge streaming profile 64 KiB/10 s on the constrained leg.
- **Hierarchy placement.** The sensor cell is a `Cell`; the vessel is its `Leader` (or a `Cohort`
  coordinator) routing upward; the shore post observes `CohortSummary`/`FederationSummary` roll-ups.
  **Leaders route upward; non-leaders cannot cross-cell or reach the zone** (`hierarchy/router.rs`).
- **Data flow.** In range: standard Automerge-over-QUIC sync with negentropy reconciliation. Out of
  LOS: the design intent is to send only a **compact CohortSummary up and commands down** over the
  satellite link using a single-burst digest — **but the satellite transport and the digest scheme
  are both Proposed/Speculative**; the shipped reconciliation (negentropy) assumes an interactive QUIC
  channel and would not work over a 1,960 B one-shot frame as-is.
- **Worked example (clearly labeled Proposed math).** A single Iridium SBD frame caps near **1,960 B
  MO / 1,890 B MT** (an *Iridium hardware limit* from gbrain `quic-iridium-sbd-feasibility`, **not a
  Peat measurement**) at **5–20 s** latency and **~$0.04–0.13/msg**. A round-trip summary exchange
  would be **~40 s and two paid messages** — *illustrative arithmetic over an unbuilt transport.* The
  in-range QUIC legs are **Shipped and real**; the BLOS leg is a teaching design tied to ADR-051.

---

## Use case 4 — Disaster-response disconnected cell reconciling on reconnect (**Shipped**)

**Scenario.** A response team operates in a comms-denied building, accumulates state locally, then
reconnects to the wider mesh and converges automatically.

- **Devices.** Responder phones/tablets (peat-btle and/or peat-node); a team-lead node; the regional
  coordination node reachable after egress.
- **Transports.** Intra-team **BLE** (**Shipped**) and/or **QUIC** on local Wi-Fi; reconnect to the
  region over **QUIC/Iroh** (**Shipped**). n0 relay is **off by default** — correct for this
  air-gapped posture (#833).
- **Collections / profiles.** `tracks` (Normal), `contact-reports`/`alerts` (Critical),
  `cells`/`nodes` (High). Tactical profile 256 KiB/30 s.
- **Hierarchy placement.** The disconnected team is a partitioned `Cell` that elects a `Leader`
  locally (deterministic scoring — no quorum needed, so **no split-brain stall**); on heal it rejoins
  its `Cohort`.
- **Data flow (Shipped).** While partitioned, every node commits Automerge changes **locally first**
  (offline-first); partition detection + autonomous operation are shipped (`peat-mesh/src/topology/`).
  On reconnect, the auto-reconnect watchdog (5 s→120 s exponential backoff, peat-node) re-establishes
  QUIC; **negentropy reconciles the document sets** and only the genuinely-missing deltas transfer;
  Automerge merges deterministically (LWW per map key by Lamport ts, RGA for lists). Two
  independently-elected leaders converge deterministically — the surviving `leader_id` resolves by
  Automerge's last-writer semantics, no special reconciliation code.
- **Caveat.** **peat-btle reconnect re-delivery of pending CRDT state is In-flight (#73)** — the
  BLE-path embedded leg may not re-deliver everything queued during a long outage; the QUIC/peat-node
  path is the robust one. Tombstone retention (168h/7-day default) governs what survives a long
  offline window (#136).
- **Worked example.** A team offline for 30 minutes accumulates, say, 200 position/contact changes.
  On reconnect, negentropy exchanges fingerprints in O(log n) rounds (an *algorithmic* claim, not
  benchmarked) and transfers only the ~200 missing deltas — not the full document history (unless the
  collection is `FullHistory` mode). Convergence is automatic; **this is the strongest shipped story
  in Peat.**

---

## Use case 5 — Remote sensor field over a constrained link (embedded legs **Shipped**; long-range leg **Proposed**)

**Scenario.** Battery ESP32 sensors report readings to a better-resourced bridge node, which
aggregates them into the full Automerge mesh.

- **Devices.** ESP32/microcontroller sensors running **peat-lite firmware** (Wi-Fi/UDP); a bridge
  node running peat-mesh with the `lite-bridge` feature; the upstream mesh + (optional) peat-gateway
  control plane observing.
- **Transports.** Sensor↔bridge over **peat-lite UDP** (codec **Shipped** in peat-lite; sockets in
  peat-lite `firmware/`; bridged by peat-mesh `LiteMeshTransport`, **Shipped**, feature
  `lite-bridge`). Bridge↔mesh over **QUIC/Iroh** (**Shipped**). A LoRa long-range variant is
  **Proposed (ADR-052)** — not built. OTA firmware push to the sensors is **Shipped** (`lite_ota.rs`,
  ADR-047, stop-and-wait, SHA-256, optional Ed25519 signing).
- **Collections / profiles.** Sensor readings ride the peat-lite `Document` envelope (`0x07`) into a
  `tracks` or sensor collection (Normal QoS). CRDT types on the wire: `LwwRegister` (latest reading)
  and `GCounter` (event counts) — **the only core peat-lite CRDTs that actually ship** (`PnCounter`
  is firmware-only; `OrSet` is a reserved wire byte with no implementation).
- **Hierarchy placement.** Each sensor is a `Node`; the bridge node is the `Cell` leader / boundary
  translator (a **mesh node**, *not* the peat-gateway control plane). The gateway, if present,
  *observes* CDC events but **does not translate or sync** — it is not in the data path.
- **Data flow (Shipped).** Sensor encodes a reading as a peat-lite frame: **16-byte header + payload**,
  ≤ 512 B total, `MessageType::Document (0x07)` carrying the universal envelope (collection + doc_id +
  timestamp + opaque body). The bridge's `LiteMeshTransport` ingests it into the `DocumentStore`; the
  `AutomergeBackend` wraps it as a CRDT doc; observers fire; negentropy + Automerge sync carry it
  upstream over QUIC. The gateway's CDC watcher (`DocumentStore::observe()`) emits a flat `CdcEvent`
  to a NATS/Webhook sink (**Shipped**; Kafka is a TODO stub, Redis absent).
- **Worked example.** A sensor reading fits in one UDP datagram: 16 B header + ~12 B `Position`
  (fixed-point lat/lon microdegrees + alt cm) + envelope framing, well under the 496 B payload limit.
  A `GCounter` over 32 nodes is ~260 B (a *design-budget* figure consistent with the type, not a
  measured RAM figure — the **256 KB total budget is a design target, ADR-035, not enforced**). The
  embedded→bridge→mesh path is **Shipped**; a LoRa variant of the first leg (7–87 km, 1.5–9.1 kB/s —
  *external LoRa hardware specs, not Peat measurements*) is **Proposed**.

---

## Summary: what ships across all five

| Capability | Status |
|---|---|
| Automerge-over-QUIC sync + negentropy reconciliation + offline-first convergence | **Shipped** (the backbone of every use case) |
| BLE mesh, peat-lite UDP bridge, OTA, TAK/CoT bridge, deterministic election, formation HMAC handshake | **Shipped** |
| Cross-transport bridging codec (ADR-059) | **Shipped codec**, ADR Proposed |
| QoS class ordering | **Shipped**; cross-class preemption + "<5 s P1" enforcement **In-flight** |
| peat-gateway CDC (NATS/Webhook), envelope encryption, OIDC enrollment | **Shipped**; Kafka stub, Redis absent; ingress AuthZ/PoP **In-flight** |
| Targeted command delivery (ADR-046) | **In-flight** (#853); `command_log` CRDT **Speculative** |
| Satellite (SBD) / LoRa transports, PACE failover, satellite digest schemes | **Proposed / Speculative** |
| MLS forward secrecy | **Proposed** (ADR-044) |
