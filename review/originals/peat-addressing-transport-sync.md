---
type: concept
title: "PEAT layering: addressing vs. transport vs. sync — the non-IP decomposition"
tags:
  - architecture
  - transport
  - sync
  - non-ip
  - constrained
  - peat
  - adr-candidate
status: draft — design note, candidate for ADR
---

# PEAT layering: addressing vs. transport vs. sync

> **Status:** Draft design note. Candidate for promotion to an ADR in `peat/docs/adr/`
> (next free number is ≥ 070; ADR-069 is the highest currently in the tree). Written to
> consolidate a recurring question — *"how do we run PEAT where QUIC can't go?"* — into a
> stable decomposition that the constrained-transport ADRs (032, 035, 039, 051, 052, 059)
> can all hang off of. See [[peat-hierarchy-vocabulary]] and
> [[research/quic-iridium-sbd-feasibility]].

## Context

The constrained-networking question is usually framed as *"PEAT uses QUIC for the
networking/addressing protocol, so how do we use it in non-IP, low-bandwidth situations where
QUIC can't run?"* That framing bundles together three concerns that PEAT actually keeps
separate. Untangling them is most of the answer, because each concern has a different
"QUIC-era default" and a different substitute once you leave IP behind.

QUIC (via Iroh, per ADR-011) is **one transport**. It is not PEAT's addressing layer, and it is
not PEAT's sync engine. The reason it *looks* load-bearing is that Iroh derives its node
identity from the same kind of Ed25519 key PEAT uses, so on the QUIC path several layers
collapse onto one library. They are still logically distinct, and they come apart cleanly when
you need them to.

## Decision: treat PEAT as three independently-substitutable layers

### Layer 1 — Addressing & identity

**What it is:** *who* a node is, independent of how you reach it.

- The canonical identifier is the **`NodeId`** — in the full mesh, the SHA-256 of the node's
  Ed25519 public key (ADR-006). In `peat-lite` it is a 32-bit `NodeId` (`peat-lite/src/node_id.rs`).
  The two are bridged at the gateway (`peat-mesh/src/transport/btle.rs::btle_to_peat_node_id` /
  `peat_to_btle_node_id`); the same bridging pattern applies to any future constrained transport.
- A `NodeId` has **nothing to do with IP, ports, or QUIC**. It is a stable name derived from a
  keypair, like the name on an envelope rather than the street address.

**QUIC-era default:** identity ≈ Iroh node id ≈ IP reachability, all one thing.

**Non-IP substitute:** keep the `NodeId` unchanged; each transport owns a mapping from `NodeId`
to its native handle — IMEI for SBD (ADR-051), radio endpoint / pre-shared peer entry for LoRa
(ADR-052), 32-bit node id over GATT for BLE (ADR-039). ADR-032's "Identity bridging" amendment
already specifies this contract: `peat_mesh::NodeId` is the unified key; per-transport code
converts it to the native address, exactly as `connect()` already does.

**Consequence:** "porting PEAT to non-IP" never means changing identity. The address book is
per-transport; the names are universal.

### Layer 2 — Transport (reachability)

**What it is:** physically moving bytes between two `NodeId`s, plus discovery (learning a peer
exists) and link security.

PEAT already treats this as pluggable via the `MeshTransport` / `Transport` traits
(`peat-protocol/src/transport/`, ADR-032). A transport advertises `TransportCapabilities`
(bandwidth, latency, range, max message size, reliability, broadcast, power) and the
`TransportManager` selects among registered transports using PACE policy
(Primary / Alternate / Contingency / Emergency).

QUIC bundles three sub-services here; each degrades differently off-IP:

| Sub-service | QUIC-era default | Non-IP substitute |
|---|---|---|
| **Discovery** | mDNS / holepunch | BLE advertisement; LoRa beacon frame (ADR-052 §Discovery); pre-provisioned IMEI registry (SBD); gateway-held peer table |
| **Reliable ordered stream** | QUIC streams | **Usually unnecessary — see Layer 3.** Datagram + idempotent delta is enough |
| **Link encryption** | TLS 1.3 in QUIC (hop-by-hop) | App-/object-layer E2EE that survives relays: per-peer E2EE (peat-btle), candidate OSCORE/AES-CCM (FIPS-aligned) over constrained links |

**Key point:** the constrained-transport ADRs (051, 052) are mostly Layer-2 work —
framing, fragmentation, duty cycle, modem drivers, cost budgeting. That work is real but
largely mechanical, and the per-crate pattern (one transport per crate, implement the
`Transport` trait) is already established by peat-btle.

### Layer 3 — Sync semantics (the hard layer)

**What it is:** making independent replicas converge to the same state.

This is where the QUIC-era assumptions actually bite, and it is the layer that genuinely
needs design rather than plumbing.

- The default stack (ADR-011) is **Automerge + Iroh**, and Automerge's sync protocol is
  *interactive*: peers exchange have-heads / Bloom-filter messages in a low-latency
  back-and-forth until converged. That assumes a chatty, bidirectional, ~millisecond-RTT
  channel — precisely what SBD (5–20 s RTT, point-to-point, ~$0.10/msg, one message per
  window) and LoRa (1.5–9.1 kB/s, half-duplex, duty-cycle-bounded) do **not** provide.
- It is also too big: Automerge wants `std` and ~10 MB; a 256 KB microcontroller can't host it
  (ADR-035).

**The enabling insight:** CRDT deltas are **commutative and idempotent**. Apply the same delta
twice → no change; apply two deltas in any order → same state. When your data structure has
those properties, QUIC's reliable-ordered-exactly-once machinery is *redundant*. You can lose,
duplicate, and reorder packets and still converge. This is the deep reason PEAT can move to
non-IP gracefully where a TCP/RPC-shaped (ordered, request/response) protocol could not — you
get to **discard QUIC's delivery guarantees and lean on the CRDT's math instead**.

**Non-IP substitutes for the interactive sync protocol:**

1. **`peat-lite` bounded CRDTs** (`LwwRegister`, `GCounter`, `PnCounter`, `OrSet` —
   `peat-lite/src/protocol/crdt_type.rs`) replace Automerge at the constrained edge. Same
   conflict-free semantics, kilobytes instead of megabytes.
2. **State-based shipping over op-based.** On a metered/slow link, ship *current collapsed
   state*, not operation history. A register that changed 50 times while a node was dark ships
   once (value + timestamp). Counters ship their current per-node totals. This is the classic
   CvRDT-vs-CmRDT trade, and constrained links strongly favor state-based because state is
   bounded, idempotent, and collapses history. peat-lite's CRDTs are already small bounded-state
   types — ideal for this.
3. **Non-interactive anti-entropy / digest reconciliation** replaces the chatty sync handshake
   (see next section). The existing `Query` message type (0x04) is the natural pull hook.

## The load-bearing pattern: dialects + a gateway interpreter

You do **not** run the same stack everywhere. Two dialects with an interpreter between them:

- **Constrained edge** (ESP32, watch, LoRa sensor) speaks the `peat-lite` dialect: bounded
  CRDTs in the 16-byte-header wire protocol (ADR-035), carried in the universal `Document`
  envelope (`peat-lite/src/protocol/document.rs`) or typed CRDT `Data` frames.
- **Gateway node** (phone, Pi, Linux box) is bilingual: speaks lite CRDTs out to the edge over
  BLE/LoRa/SBD, and full Automerge-over-QUIC into the rich mesh, translating between the two
  (peat-btle "dual-mode", ADR-039 / ADR-041; LoRa gateway bridge, ADR-052; SBD relay, ADR-051).

This means no constrained transport ever has to make Automerge itself survive a 1% duty cycle —
it only has to get a *delta* across, and let a well-resourced gateway do the heavy merge. The
edge nodes speak pidgin; the gateway is the interpreter that lets that state join the larger
conversation.

## Anti-entropy over store-and-forward (sync-layer design)

The genuinely open problem. CRDTs guarantee convergence *if* every replica eventually sees every
delta. On QUIC "eventually" is milliseconds and the interactive protocol makes completeness
cheap. Over SBD you cannot run that handshake. Decisions:

- **Reconcile edge ↔ relay, not edge ↔ edge.** The relay holds authoritative merged mesh state,
  so a field node tracks only its frontier relative to the relay, not an N-peer version vector.
  This collapses the problem.
- **One session = one round trip, by construction.** An Iridium `+SBDIX` session sends MO and
  receives MT in a single satellite pass. Design each session to be maximally productive: MO
  carries (my new deltas since last session) + (my digest); MT carries (the deltas the relay
  selected for me) + (relay digest). Converges in `ceil(divergence / capacity)` sessions.
- **Digest form depends on whether the node persists state:**
  - *Persistent edge/full nodes:* a **version vector** keyed by `NodeId` → highest `seq_num`
    seen (the lite header already carries a per-node `seq_num`). 8 bytes/node; a ≤100-node cell
    fits in one SBD message with room for deltas.
  - *Ephemeral lite nodes* (peat-lite is ephemeral-first, often no durable vector across reboot):
    send only own latest `seq` + a "last contact" timestamp; relay ships "everything changed
    since T, newest-state-only, priority-ordered." Snapshot-since-T, not vector diff.
- **Scale the digest with the topology.** A federation-scale version vector (1000 nodes ≈ 8 KB)
  won't fit. Use a **hierarchical digest** that leans on PEAT's existing summaries
  (`CellSummary → CohortSummary → …`, ADR-066 vocabulary): carry your cell's vector plus a hash
  per higher tier; the relay knows the hierarchy.
- **Priority selection when capacity < divergence** (ADR-046 targeted delivery): the relay fills
  each scarce MT message critical-collection-first (commands/emergency before telemetry
  history).
- **Tombstones / GC keyed to the reconnect window** (issue #857; `DOC_FLAG_TOMBSTONE`,
  `peat-lite/src/protocol/ttl.rs`): a node dark for a week must still learn of a delete, so
  tombstone TTL is keyed to the slowest expected reconnect interval, with the **relay as the
  retention authority** (it keeps tombstones longer than edge nodes, which trust the relay's
  view).
- **Adapt the anti-entropy strategy to the link, like LoRa adapts spreading factor.** SBD
  (one message/hour, RTT-bound) → single-round-trip digest + state-based snapshots. LoRa
  (message-rich, bandwidth-bound, seconds RTT) → can afford a **Bloom/IBLT set-difference
  digest** sized to the *expected diff* (echoing Automerge's own Bloom-based sync), trading a
  few exchanges for a tighter delta. The strategy is a property of the transport's
  RTT-vs-bandwidth profile, not a global constant.

## Implications for adding a new constrained transport

1. Reuse `NodeId`; write only the `NodeId ↔ native-handle` mapping (Layer 1).
2. Implement the `Transport` trait + capabilities + a fragmentation strategy fit to the MTU and
   duty cycle (Layer 2). Reuse the `Document` envelope; keep the codec transport-agnostic.
3. Decide the node's sync role (Layer 3): pure edge (lite CRDTs, relayed) vs. gateway
   (bilingual). Pick an anti-entropy strategy from the menu above based on the link's
   RTT-vs-bandwidth profile. Do **not** attempt to tunnel QUIC/Automerge interactive sync.

## Open questions

- Exact digest wire format and whether it rides existing `Query`/`Ack` types or earns a new
  message type.
- Whether the relay's tombstone-retention authority needs a formal lease/epoch, or TTL suffices.
- FIPS posture for constrained-link E2EE: ADR-052/ADR-039 currently specify ChaCha20-Poly1305,
  which violates the FIPS-only hard rule in `peat/CLAUDE.md`. An OSCORE/AES-CCM envelope would
  resolve this and standardize object-security across constrained transports — worth its own ADR.
- Cold-start cost ceiling: a long-dark ephemeral node's snapshot-since-T may exceed one message;
  define the multi-window catch-up contract and its priority ordering.

## References

- ADR-011 (Automerge + Iroh), ADR-032 (pluggable transport), ADR-035 (peat-lite embedded),
  ADR-039 (peat-btle), ADR-041 (multi-transport embedded), ADR-046 (targeted delivery),
  ADR-051 (SBD satellite), ADR-052 (LoRa), ADR-059 (cross-transport bridging),
  ADR-063 (persistent sync streams), ADR-066 (hierarchy vocabulary).
- [[peat-hierarchy-vocabulary]], [[research/quic-iridium-sbd-feasibility]],
  [[peat-open-issues-snapshot]] (issue #857 sync-layer relay/GC).
</content>
</invoke>
