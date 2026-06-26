# Design-note discrepancies — `peat-addressing-transport-sync.md`

Cross-check of the addressing/transport/sync design note against
`learning/review/ground-truth.md` (the verified current state, audited at `peat` HEAD
`35d0f11`). Format per discrepancy: **claim → reality → correction.**

The design note's *thesis* (three independently-substitutable layers; CRDT commutativity lets
you discard QUIC's delivery guarantees; dialect-plus-gateway-interpreter pattern; anti-entropy
is the genuinely open layer) is sound and consistent with ground truth. The discrepancies below
are factual/citation errors and status mislabels in the supporting detail. Most of the sync-layer
content (version vectors, snapshot-since-T, hierarchical digest, IBLT/Bloom) is **teaching-only
design** — that is correct for a design note, but the note does not flag it as not-yet-implemented.

---

## A. Identity / addressing (Layer 1)

1. **Claim** (line 44): "The canonical identifier is the **`NodeId`** — in the full mesh, the
   SHA-256 of the node's Ed25519 public key (ADR-006)."
   **Reality** (ground truth §4): there is no single canonical identifier; **four+ identity
   schemes** coexist. The mesh crypto identity is `security::DeviceId = [u8;16]` =
   **first 16 bytes (128 bits) of SHA-256(Ed25519 verifying-key)**, NOT a full SHA-256.
   `transport::NodeId` is a `pub struct NodeId(String)` — a transport-assigned string, **not a
   hash**. iroh uses the raw Ed25519 `EndpointId` (no SHA-256 wrap). Spec 001/005's "32 bytes of
   SHA-256" is itself spec drift the code contradicts.
   **Correction:** state that Peat's mesh crypto identity is `DeviceId` = first **16 bytes** of
   SHA-256(Ed25519 pubkey) (`peat-mesh/src/security/device_id.rs:33,39-47`); that `NodeId` is a
   separate transport-string type; and that identity is **not** unified across layers. Drop the
   "ADR-006 / canonical NodeId = full SHA-256" framing.

2. **Claim** (lines 46-47): the lite↔mesh NodeIds "are bridged at the gateway
   (`peat-mesh/src/transport/btle.rs::btle_to_peat_node_id` / `peat_to_btle_node_id`)."
   **Reality** (ground truth §4): `btle_to_peat_node_id` is **not found by that name** in
   peat-mesh or peat-btle (grep clean). Cross-transport identity bridging lives behind the
   `Translator` trait / `BleTranslator` (peat-mesh, feature `bluetooth`), and the u32↔DeviceId
   hop is non-trivial and partly unverified. The note also overstates "at the gateway" —
   peat-gateway implements zero mesh transports and is not in the bridging path (§7).
   **Correction:** replace the named functions with the `Translator`/`BleTranslator` seam; mark
   the exact u32↔DeviceId derivation as unverified; remove the implication that bridging happens
   in the gateway.

3. **Claim** (line 45): peat-lite NodeId is "a 32-bit `NodeId`."
   **Reality** (ground truth §4): correct — `u32`, `#[repr(transparent)]`, **bare integer, no key
   derivation** (`peat-lite/src/node_id.rs:9-34`). Note that peat-btle's NodeId is a *different*
   u32 (first 4 bytes of BLAKE3(pubkey)).
   **Correction:** none required for the bit-width; optionally add that the lite NodeId has no key
   derivation (so the "name derived from a keypair" framing in Layer 1 does not apply to lite).

---

## B. Transport (Layer 2)

4. **Claim** (line 67): "Peat already treats this as pluggable via the `MeshTransport` /
   `Transport` traits (`peat-protocol/src/transport/`, ADR-032)."
   **Reality** (ground truth §1, §2): the transport traits and impls live in **`peat-mesh`**;
   `peat-protocol`'s transport/network modules are thin `pub use peat_mesh::…` re-export shims.
   **ADR-032 is Status: Proposed** (the trait seam ships, the ADR is not Accepted).
   **Correction:** cite `peat-mesh/src/transport/` as the home of the traits (re-exported by
   peat-protocol); label ADR-032 **Proposed (seam shipped)**.

5. **Claim** (lines 70-72): the `TransportManager` "selects among registered transports using
   PACE policy (Primary / Alternate / Contingency / Emergency)" — presented as operational.
   **Reality** (ground truth §2): only QUIC/Iroh, BLE, and the peat-lite UDP bridge are real
   `MeshTransport` impls. `WifiDirect`/`LoRa` appear in `preference_order` but have **no backing
   impl** (`manager.rs` returns `None`). **Multi-transport PACE failover is README Phase-2
   Planned = Proposed**, not shipped.
   **Correction:** label PACE multi-transport failover **Proposed/Planned**; note the
   `TransportType` enum is a taxonomy, not an implementation inventory.

6. **Claim** (line 79, "Link encryption" row): non-IP substitute is "per-peer E2EE (peat-btle)"
   alongside "candidate OSCORE/AES-CCM."
   **Reality** (ground truth §6): peat-btle's shipped per-peer E2EE is **AES-256-GCM + ECDH-P256**
   (migrated off ChaCha20/X25519 at rc.12, 2026-05-18). This is FIPS-aligned already — fine to
   cite, but the surrounding "candidate FIPS-aligned" framing implies the constrained path is not
   yet FIPS-clean when peat-btle's shipped path is.
   **Correction:** state peat-btle ships AES-256-GCM/ECDH-P256 (FIPS-approved primitives; modules
   not CMVP-validated, peat-btle#75 open); reserve "candidate OSCORE/AES-CCM" only for the
   Proposed SBD/LoRa links.

7. **Claim** (lines 77-78, Discovery/stream rows referencing ADR-052 §Discovery, SBD IMEI
   registry): presented as available substitutes.
   **Reality** (ground truth §2): **ADR-051 (SBD) and ADR-052 (LoRa) are Proposed-only — no crate,
   no module.** The `Satellite`/`LoRa` enum variants exist but resolve to "no transport
   registered."
   **Correction:** label all SBD/LoRa references **Proposed (no crate)** wherever they appear in
   the table.

---

## C. Sync semantics (Layer 3)

8. **Claim** (line 110): "`peat-lite` bounded CRDTs (`LwwRegister`, `GCounter`, `PnCounter`,
   `OrSet` — `peat-lite/src/protocol/crdt_type.rs`)."
   **Reality** (ground truth §3): the published peat-lite **library** ships only `LwwRegister<T>`
   and `GCounter<const N=32>` (+ hybrid `CannedMessageStore`). **`PnCounter` is firmware-only**
   (workspace-excluded `firmware/`, not in the published library). **`OrSet` has no struct
   anywhere** — `crdt_type.rs:10` is a reserved **wire byte (0x04)** only.
   **Correction:** list the shipped library CRDTs as `LwwRegister` + `GCounter`; mark `PnCounter`
   **firmware-only** and `OrSet` **reserved wire code, no implementation**. Cite
   `peat-lite/src/{lww,counter}.rs` for the structs, `crdt_type.rs:6-11` for the wire codes.

9. **Claim** (line 121): "The existing `Query` message type (0x04) is the natural pull hook."
   **Reality:** **correct.** `Query = 0x04` is a real message type
   (`peat-lite/src/protocol/message_type.rs:21`), distinct namespace from the CRDT-type byte 0x04
   (OrSet). No change needed — flagged here only because 0x04 is reused across two enums.

10. **Claim** (lines 95-99): Automerge "wants `std` and ~10 MB; a 256 KB microcontroller can't
    host it."
    **Reality** (ground truth §8): the **256 KB is a real design target (ADR-035), not
    enforced/measured** (no allocator cap, no static-RAM assertion); the **"~10 MB" Automerge
    footprint is unverified**.
    **Correction:** keep the qualitative point but flag both numbers — 256 KB as ADR-035 design
    target (unenforced), ~10 MB as an unverified estimate.

11. **Whole §"Anti-entropy over store-and-forward" (lines 138-175):** version vectors,
    snapshot-since-T, hierarchical digest, Bloom/IBLT set-difference — presented as the design.
    **Reality** (ground truth §3): the **only shipped anti-entropy is `negentropy` set
    reconciliation** over Automerge sync on Iroh QUIC (`peat-mesh/src/storage/negentropy_sync.rs`,
    ADR-040/#435). **Version-vector, snapshot-since-T, hierarchical-digest, and IBLT schemes are
    NOT implemented — teaching-only design.** peat-lite has **no sync engine, no anti-entropy, no
    leader election at all.**
    **Correction:** add an explicit "Speculative / teaching-only" banner to this section and name
    `negentropy` as the actual shipped reconciliation mechanism on the QUIC path; this is a design
    note so the speculation is legitimate, but it must be labeled.

---

## D. Citations, issues, ADR numbering

12. **Claim** (lines 165, 204): "Tombstones / GC keyed to the reconnect window (issue #857)" and
    "[[peat-open-issues-snapshot]] (issue #857 sync-layer relay/GC)."
    **Reality** (ground truth §10): **#857 is ADR-046 Phase-4 selectors, NOT tombstone GC.**
    Tombstone GC knobs live in **peat-node#136**; sync-GC epic context is #857's sibling set, not
    #857 itself.
    **Correction:** cite **peat-node#136** for tombstone GC (and #853/#854-#859 for ADR-046
    targeted-delivery context); drop the "#857 = sync GC" mapping.

13. **Claim** (line 18): "next free number is ≥ 070; ADR-069 is the highest currently in the
    tree."
    **Reality** (verified `ls peat/docs/adr/`): **ADR-070 (sapient-protocol-bridge) exists**, and
    there are **75 ADR `.md` files** (with a duplicated 064). ADR-069 is not the highest.
    **Correction:** next free number is **≥ 071**; note ADR-070 is the current highest.

14. **Claim** (line 131): the dialect/interpreter pattern cites "peat-btle 'dual-mode', ADR-039 /
    ADR-041."
    **Reality** (ground truth §9): **ADR-041 is Accepted** (the only Accepted ADR in the
    constrained set); **ADR-039 is Proposed** (peat-btle 0.4.0 ships). Also, peat-btle 0.4.0
    **dropped its peat-mesh back-edge** (ADR-059 Amendment 4), so "dual-mode" bridging now runs
    through the peat-mesh `BleTranslator`, not inside peat-btle.
    **Correction:** label ADR-041 **Accepted**, ADR-039 **Proposed (shipped)**; note the
    bridging now lives in peat-mesh, not peat-btle.

15. **Assumption** (lines 148-149): "An Iridium `+SBDIX` session sends MO and receives MT in a
    single satellite pass."
    **Reality** (ground truth §8): this is an **external Iridium hardware fact, not a Peat fact**;
    peat-sbd is Proposed-only and there is no `+SBDIX` code (grep clean).
    **Correction:** label it explicitly as a hardware assumption (cite
    `research/quic-iridium-sbd-feasibility`), not a Peat capability.

---

## E. Status-label fixes applied IN PLACE

Per the task, only provably-wrong labels were corrected in the note itself (proposals mislabeled
as shipped, wrong issue/ADR-number facts). The design content and speculative sync menu were left
intact:

- Header status line: added an explicit note that the constrained transports it hangs off
  (032/035/039/051/052/059) are **Proposed** and ADR-041 is the lone **Accepted** one.
- "next free number ≥ 070 / ADR-069 highest" → corrected to **≥ 071 / ADR-070 highest**.
- ADR-032 reference (Layer 2) → marked **Proposed (trait seam shipped)**.
- ADR-051/052 references → marked **Proposed (no crate)**.
- PACE multi-transport failover → marked **Planned/Proposed**.
- ADR-039/ADR-041 in the dialect section → marked **Proposed (shipped)** / **Accepted**.
- Issue **#857** (tombstone GC) → corrected to **peat-node#136**.
- peat-lite CRDT list → `PnCounter` marked firmware-only, `OrSet` marked reserved-wire-code-only.

Not changed (left as legitimate design speculation, but cross-referenced above): the Layer-1
"canonical NodeId = SHA-256" framing and the `btle_to_peat_node_id` function names are factual
errors but are woven through the prose; they are listed here for a follow-up prose pass rather
than a wholesale rewrite, per the "do not rewrite wholesale" instruction.
