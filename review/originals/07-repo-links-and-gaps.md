# Module 7 — Repo Map, Gaps & Links to Add

**Goal:** know exactly what's in this local folder, what's referenced but **missing**, and the full
set of links so the team can pull down what's not here. This is the "what do I download next" map.

---

## 7.1 What's local (in this folder)

These five are present and are what the rest of this track covers:

| Local dir | Upstream | crates.io / artifact |
|-----------|----------|----------------------|
| `peat/` (umbrella: `peat-protocol`, `peat-schema`, `peat-transport`, `peat-persistence`, `peat-ffi`, `examples/`, `spec/`, `docs/adr/`) | https://github.com/defenseunicorns/peat | `peat-protocol`, `peat-schema` on crates.io; `peat-ffi` on Maven Central |
| `peat-mesh/` | https://github.com/defenseunicorns/peat-mesh | `peat-mesh` on crates.io |
| `peat-btle/` | https://github.com/defenseunicorns/peat-btle | `peat-btle` on crates.io + Maven Central |
| `peat-lite/` | https://github.com/defenseunicorns/peat-lite | `peat-lite` on crates.io + Maven Central |
| `peat-gateway/` | https://github.com/defenseunicorns/peat-gateway | server-side; not an SDK crate |

---

## 7.2 Referenced but NOT present locally — pull these down

These are PEAT repos referenced in code comments, `Cargo.toml`s, READMEs, or ADRs that are **not**
checked out in this folder. Listed roughly by how useful they'd be to an onboarding intern.

| Repo / component | GitHub | Why it's referenced | Priority to add |
|------------------|--------|---------------------|-----------------|
| **peat-sim** | https://github.com/defenseunicorns/peat-sim | Network simulator for multi-node / partition testing; cited in `peat/docs/ARCHITECTURE.md` and research docs. Great for experimenting without hardware. | High |
| **peat-node** | https://github.com/defenseunicorns/peat-node | A node distribution referenced in PEAT URLs. (Note: a `peat-mesh-node` binary already lives *inside* `peat-mesh`; confirm whether this repo is a separate packaging/deployment of it.) | Medium |
| **peat-registry** | https://github.com/defenseunicorns/peat-registry | UDS registry replication for DDIL environments (ADR-054). Relevant to gateway/air-gapped deployment. | Medium |
| **peat-sdk-go** | https://github.com/defenseunicorns/peat-sdk-go | Go SDK for PEAT — useful if any consumer services are written in Go. | Low–Medium |
| **peat-android / android demo** | (referenced as a standalone repo in `peat-mesh` issue #828; a demo also lives locally at `peat/examples/android-peat-demo`) | Android consumer integration + UAT. Confirm whether the standalone repo exists or whether the in-repo example is the canonical one. | Medium (mobile work) |

### peat-inference — a real, documented crate that isn't cloned here

- **peat-inference** — the edge AI/ML pipeline (ONNX runtime, YOLOv8 object detection, ByteTrack
  tracking, GStreamer video capture, TensorRT GPU acceleration, plus model distribution). It is
  **fully documented** in the developer guide
  ([`DEVELOPER_GUIDE.md`](../peat/docs/guides/developer/DEVELOPER_GUIDE.md) §5.7 and §11) and listed
  in that guide's project layout — so it is a real component, *not* a deprecated one. It is simply
  **not present in this local checkout** and is **not** a member of the current `peat` workspace
  `Cargo.toml` (nor in the README's ecosystem table), which suggests it now lives in its own repo or
  was de-vendored from the umbrella since the guide's 2025-12-08 snapshot. **Action:** ask the team
  for its current home (likely `github.com/defenseunicorns/peat-inference`) and clone it if you'll
  do edge-AI work.

### peat-sim — documented, referenced as its own repo

- **peat-sim** — the network simulator. The developer guide runs it via `cargo run --bin peat-sim`
  and lists it in the project layout, but the README points to it as a standalone repo
  (`github.com/defenseunicorns/peat-sim`) and it isn't a member of the local workspace. Same
  situation as peat-inference: real and documented, just extracted. (Also listed in the table above.)

### Defense Unicorns platform repos (ecosystem, not PEAT core)

These show up because of UDS/Zarf packaging in `peat-gateway`. Pull only if you're doing
deployment/packaging work:

- **uds-core** — https://github.com/defenseunicorns/uds-core (the UDS platform the gateway bundles into)
- **pepr** — https://github.com/defenseunicorns/pepr (Kubernetes operator framework used by UDS)

---

## 7.3 External (non-PEAT) dependencies worth a bookmark

Not repos to clone, but the upstreams you'll read docs for while working in PEAT:

- **Iroh** (QUIC P2P) — https://github.com/n0-computer/iroh — the heart of `peat-mesh` transport.
- **Automerge** (CRDT) — the document model; sync protocol paper: https://arxiv.org/abs/2012.00472
- **negentropy** (set reconciliation) — the O(log n) sync optimization (ADR-040).
- **redb** — pure-Rust embedded key-value store used for persistence.
- **bluer** (Linux BlueZ), **objc2-core-bluetooth** (Apple), **WinRT** (Windows), **NimBLE/esp-idf**
  (ESP32) — the BLE platform backends behind `peat-btle`.
- **UniFFI** — generates the Kotlin/Swift bindings in `peat-ffi` and the mobile crates.
- **Axum** — the HTTP/WS framework in `peat-mesh`'s broker and `peat-gateway`.
- **TAK / Cursor-on-Target / MIL-STD-2525** — the tactical interop standards the CoT layer targets.

---

## 7.4 Suggested clone command (for whoever sets up the workspace)

To complete the picture alongside what's already here:

```bash
# from the directory that contains peat/, peat-mesh/, etc.
for r in peat-sim peat-node peat-registry peat-sdk-go; do
  git clone https://github.com/defenseunicorns/$r.git
done
# Optional, for deployment/packaging work:
# git clone https://github.com/defenseunicorns/uds-core.git
# git clone https://github.com/defenseunicorns/pepr.git
```

> **Before cloning `peat-node` and the standalone `peat-android`:** confirm with the team that
> these are distinct from the in-repo `peat-mesh-node` binary and `peat/examples/android-peat-demo`.
> They may be redirects/mirrors rather than separate sources of truth.

---

## 7.5 Primary sources inside the repo (start here)

Before the ADRs, two in-repo docs are the strongest foundations and should be read directly:

- **[`peat/docs/guides/developer/DEVELOPER_GUIDE.md`](../peat/docs/guides/developer/DEVELOPER_GUIDE.md)**
  — the canonical onboarding doc (2025-12-08): environment setup, the runtime architecture, core
  concepts, crate reference, testing, mobile, edge AI, and "Extending Peat." This learning track is a
  guided path through it; when the track and the guide differ, the guide (plus the code) wins.
- **[`peat/docs/ARCHITECTURE.md`](../peat/docs/ARCHITECTURE.md)** — the crate/layer model and security
  overview (older, 2025-01-07; good for the packaging lens, but check it against the manifests).
- **[`peat/docs/guides/developer/FORMATION_AND_LEADERSHIP.md`](../peat/docs/guides/developer/FORMATION_AND_LEADERSHIP.md)**
  — code-level how-to for cell formation, the auth handshake, and leader election (the source for
  **Module 2·5**). *Newer than this local checkout — fetched from `main`; the shallow clone here may
  not contain it yet (run `git pull` in `peat/`).*
- **[`peat/docs/guides/QUICKSTART.md`](../peat/docs/guides/QUICKSTART.md)** &
  **[`peat/docs/guides/operator/OPERATOR_GUIDE.md`](../peat/docs/guides/operator/OPERATOR_GUIDE.md)**
  — the hands-on run-a-mesh walkthrough and the operator/deploy reference (the source for **Module 8**).
- **[`peat/docs/spec/`](../peat/docs/spec/)** — the five normative IETF-style specs
  (001 transport · 002 sync · 003 schema · 004 coordination · 005 security) — the source for
  **Module 9**. The precise contract; read when you need exact wire formats, enums, or rules.
- **[`peat/docs/whitepaper/`](../peat/docs/whitepaper/)** — the conceptual/positioning narrative
  (chapters 01–09): the scaling crisis, the missing coordination layer, the hierarchy insight, the
  open-architecture imperative. Distilled in **Module 1·5 (The Big Idea)**. It has its own build
  system (`make html`/`pdf`/`docx`) and large appendices: `10b-spec-appendix.md` (the IETF-style
  protocol spec) and `11-adr-appendix.md` (every ADR concatenated — ~47k lines). Read the narrative
  chapters for the "why"; treat the appendices as the spec/ADR reference.

## 7.6 The ADR archive — your deepest primary source

`peat/docs/adr/` holds ~60 Architecture Decision Records; `peat-mesh/docs/adr/` and
`peat-btle/docs/adr/` hold more. When you want to know *why* something is the way it is, these beat
any summary (including this one). High-value starting ADRs:

- **ADR-007 / ADR-011** — the CRDT backend choice (Automerge + Iroh).
- **ADR-006 / ADR-048** — security architecture & membership certificates.
- **ADR-019** — QoS classification.
- **ADR-040** — negentropy set reconciliation.
- **ADR-049** & peat-mesh **ADR-0002** — the `peat-mesh` extraction (explains the dependency-arrow flip).
- **ADR-059** — cross-transport document bridging (explains the `peat-btle ↔ peat-mesh` cycle break).
- **ADR-060 / ADR-062** — FIPS crypto posture & HKDF key derivation.

---

## Checkpoint

- Which five repos are local, and which are referenced but need cloning?
- Which referenced component's status is uncertain, and what would you do before relying on it?
- Where do you go to learn *why* a design decision was made?

---

That's the track. Loop back to [Module 0](00-START-HERE.md) any time, or open the interactive
[`peat-learning-hub.html`](peat-learning-hub.html) for the navigable version.
