# Ground Truth — `peat-gateway`

**Repo:** `peat-gateway` (subdir `./peat-gateway` of the Peat umbrella)
**Audited HEAD:** `8d16824` — `chore(ci): add dependabot config; group /ui (pnpm) updates into one PR (#134)`
**Branch:** `main`, up to date with `origin/main` (`git rev-list --count HEAD..@{u}` = 0). No pull/modify performed.
**Crate version:** `0.1.0` (`Cargo.toml:3`)
**peat-mesh dependency pin:** `=0.9.0-rc.1` with features `automerge-backend`, `broker` (`Cargo.toml:23`). **Pin is stale relative to the ecosystem** — crates.io has `peat-mesh 0.9.0-rc.35` extracted locally and the sibling `peat-mesh` checkout is at `0.9.0-rc.42`. The SKILL.md treats the `=0.9.0-rc.1` pin as intentional (`SKILL.md:74`).
**Audited:** README.md, Cargo.toml, CLAUDE.md, SKILL.md, all of `src/` (crypto, tenant, cdc, ingress, api, storage), `docs/`, and the pinned-version peat-mesh security API (cross-checked against rc.35 source in the cargo cache; rc.1 tarball was not extracted, see Assumption A1).
**Test surface:** 371 `#[test]`/`#[tokio::test]` functions across `src/` + `tests/` (22 integration test files).

---

## 1. What this repo actually is

`peat-gateway` is the **enterprise control plane** for Peat meshes (ADR-055). It is a Rust binary + library (`peat_gateway`) plus a SvelteKit admin UI (`ui/`), a Helm chart (`chart/`), a Zarf manifest (`zarf.yaml`), and a UDS bundle (`bundle/`).

**Critical framing (README.md:26-29, verified in code):** the gateway is **NOT a mesh node and NOT a data plane.** It does not run mesh transports, does not sync CRDTs, and does not sit in the peer-to-peer data path. It *observes and manages*. This is enforced structurally:
- It depends on `peat-mesh` as a **library** for `MeshGenesis`, `MeshCertificate`, `MeshBrokerState`, and `DocumentStore` types — it does not open QUIC/Iroh endpoints itself.
- `MeshStateRegistry` (`src/api/formations.rs:31-70`) holds `Arc<dyn MeshBrokerState>` handles that **external mesh nodes register into** the gateway so the read-only API can query them. The doc comment is explicit: *"The gateway control plane does not run mesh nodes itself — external mesh nodes register their broker state handles here so the API can query them"* (`src/api/formations.rs:28-30`).

So for the transport question the headline is: **peat-gateway implements zero wire transports.** All transport reality lives in `peat-mesh` and the leaf crates. What the gateway ships are **control-plane integration surfaces.**

---

## 2. Transports — what this repo implements

**Verdict: the gateway implements no Peat mesh transports (QUIC/Iroh, BLE, peat-lite, SBD, LoRa).** None appear in `src/` and none are dependencies. This is correct and by design (§1).

What it *does* speak, as a control-plane service, are **enterprise integration protocols** — all of which are **Shipped** unless noted:

| Integration | Status | Evidence | Notes |
|---|---|---|---|
| HTTP REST admin API (axum 0.7) | **Shipped** | `src/api/mod.rs:35-93`, `Cargo.toml:29` | The consumer-facing API surface (§6) |
| NATS JetStream — **CDC sink** (egress) | **Shipped** (feature `nats`, default) | `src/cdc/nats_sink.rs`, `src/cdc/engine.rs:54-70` | Publishes CRDT-change CDC events outward |
| NATS JetStream — **control-plane ingress** (inbound commands) | **Shipped** (feature `nats`) | `src/ingress/mod.rs` (whole module) | Per-org durable push consumers, DLQ, panic recovery; see §5 |
| HTTP webhook — CDC sink | **Shipped** (feature `webhook`, default) | `src/cdc/webhook_sink.rs`, `engine.rs:81-90` | |
| OIDC token introspection (RFC 7662) | **Shipped** (feature `oidc`, default) | `src/api/enroll.rs:305-392` | Identity federation; userinfo fallback |
| Kafka CDC sink | **In flight / stubbed** (feature `kafka`, non-default) | `src/cdc/engine.rs:78-80` is `// TODO: Kafka delivery` | The `rdkafka` dep + feature flag exist (`Cargo.toml:61,83`) but the dispatcher branch is an empty TODO. **The README table (line 19, 96-98) advertises Kafka as a sink — it does not actually deliver.** |
| Redis Streams CDC sink | **Proposed only** | README.md:19 lists it; **no `CdcSinkType::Redis` variant, no feature, no code.** `CdcSinkType` (`src/tenant/models.rs:79-84`) is `Nats`/`Kafka`/`Webhook` only. |
| AWS KMS (key provider) | **Shipped** (feature `aws-kms`) | `src/crypto/kms.rs` | Envelope encryption backend |
| HashiCorp Vault Transit (key provider) | **Shipped** (feature `vault`) | `src/crypto/vault.rs` | Envelope encryption backend |
| Postgres (state backend) | **Shipped** (feature `postgres`) | `src/storage/postgres.rs` | Alt to default redb |

**Feature flags (`Cargo.toml:80-90`):** `default = ["nats", "webhook", "oidc"]`; `full` adds `kafka`, `postgres`, `aws-kms`, `vault`. CI splits per-feature to avoid OOM (`SKILL.md:10,57`).

---

## 3. CRDT types

**Verdict: peat-gateway defines no CRDT types.** It is not a CRDT host. It *observes* CRDT-document mutations emitted by `peat-mesh`'s `DocumentStore` and converts them to flat CDC events:

- The watcher subscribes via `peat_mesh::sync::traits::DocumentStore::observe()` and consumes `peat_mesh::sync::types::ChangeEvent` (`Initial` / `Updated` / `Removed`) — `src/cdc/watcher.rs:6-7, 76-206`.
- It emits a `CdcEvent` (`src/tenant/models.rs:86-95`): `{org_id, app_id, document_id, change_hash, actor_id, timestamp_ms, patches: serde_json::Value}`. `patches` is just the document's current fields serialized to JSON — **not** an Automerge change/op log.
- The `change_hash` is a **non-cryptographic** `DefaultHasher` (SipHash) over doc_id+timestamp+sorted fields, used only for restart-dedup cursoring (`src/cdc/watcher.rs:238-258`). It is explicitly "unique-enough," not a content hash.

The actual CRDT engine (Automerge) lives behind the `automerge-backend` feature of `peat-mesh`. The `command_log` CRDT and peat-lite's `LwwRegister`/`GCounter`/`PnCounter`/`OrSet` are **not present here** (correctly — they're not this layer's concern).

---

## 4. Identity / addressing

The gateway is a **certificate-issuing authority front-end**, delegating all crypto-identity primitives to `peat-mesh`:

- **MeshGenesis** (`peat_mesh::security::MeshGenesis`) is created per formation: `MeshGenesis::create(&app_id, mesh_policy)` derives `mesh_id`, the authority keypair, and the formation secret (`src/tenant/manager.rs:234-237`). The gateway stores the genesis bytes (envelope-encrypted, §7).
- **mesh_id** is 8 hex chars derived via HKDF-SHA256 from the mesh seed (peat-mesh `genesis.rs:194`, `mesh_id()` at `:196`).
- **Certificate issuance:** on approved enrollment, `genesis.issue_certificate(pk_bytes, &node_id, mesh_tier, mesh_perms, 24h_ms)` is called with a client-supplied 32-byte Ed25519 public key and node_id string (`src/api/enroll.rs:256-264`). Validity is hardcoded to 24h (`24 * 60 * 60 * 1000`).
- **NodeId / DeviceId derivation** (in peat-mesh, NOT re-implemented here): `DeviceId::from_public_key` = `SHA-256(ed25519_public_key)[0..16]` (peat-mesh `security/device_id.rs:38-47`); `DeviceId → NodeId` via `NodeId::new(device_id.to_hex())` (`device_id.rs:109-112`). The prompt's claim of "NodeId = SHA-256 of Ed25519 pubkey" is **directionally correct but imprecise**: it's the first **16 bytes** of SHA-256, surfaced as a hex `DeviceId` that becomes the `NodeId`. The gateway accepts `node_id` as an opaque client-supplied string in the enroll request (`src/api/enroll.rs:54-56, 229-235`) — it does **not** derive or verify it against the public key.
- The enroll request requires a **hex 32-byte Ed25519 public key**, validated for length only (`src/api/enroll.rs:273-285`). No proof-of-possession / signature challenge is performed at enrollment (a gap; see §9).

---

## 5. Hierarchy / tier enum

This is the most likely place a skeptical reader gets misled, so it is spelled out precisely.

**The gateway defines its OWN tier enum, separate from any hierarchy vocabulary.**

```rust
// src/tenant/models.rs:154-188
pub enum MeshTier { Authority, Infrastructure, Endpoint }   // gateway-local, u32 perms
```

It maps to the **peat-mesh wire tier** via `to_mesh_tier()` (`src/tenant/models.rs:179-188`):

| Gateway `MeshTier` | → peat-mesh `security::MeshTier` |
|---|---|
| `Authority` | `Enterprise` (0) |
| `Infrastructure` | `Regional` (1) |
| `Endpoint` | `Tactical` (2) |

The peat-mesh wire enum is `Enterprise / Regional / Tactical / Edge` (peat-mesh `certificate.rs:35-44`, verified rc.35). **Note:** the gateway only maps to 3 of the 4 mesh tiers — `Edge` is unreachable from the gateway.

**Key finding for the curriculum:** *neither* the gateway tier names *nor* the peat-mesh wire tier names are the **ADR-066 hierarchy vocabulary** (Platform / Cell / Cohort / Federation / Coalition). The gateway uses an enterprise-RBAC framing (Authority/Infrastructure/Endpoint), and the mesh wire format uses a distribution-tier framing (Enterprise/Regional/Tactical/Edge) explicitly modeled on peat-registry artifact routing (peat-mesh `certificate.rs:31-32`). Any curriculum claim that the gateway "uses the ADR-066 hierarchy" is **Wrong** for this repo. The gateway's tier is a **role/trust** concept, not the formation-hierarchy concept ADR-066 governs.

**RBAC permissions** are a bitmask, not five named roles, at this layer:
- Gateway perms (u32): `RELAY=0x01, EMERGENCY=0x02, ENROLL=0x04, ADMIN=0x08` (`src/tenant/models.rs:191-216`), narrowed to mesh u8 perms via `to_mesh()`.

---

## 6. Crypto primitives actually used — FIPS posture

**Verdict: peat-gateway is FIPS-clean. No ChaCha20-Poly1305 anywhere.** A repo-wide grep for `chacha|poly1305|xsalsa|salsa20` across `*.rs/*.md/*.toml` returns **zero hits.**

Primitives in this repo:
- **AES-256-GCM** for all local symmetric crypto (`aes-gcm = "0.10"`, `Cargo.toml:53`): envelope DEK encryption (`src/crypto/mod.rs:79-110`, `local.rs:55-82`), and the KMS provider's internal test cipher (`src/crypto/kms.rs:99,128`).
- **AWS KMS** `GenerateDataKey`/`Encrypt`/`Decrypt` for the `aws-kms` backend (`src/crypto/kms.rs`).
- **Vault Transit** `/v1/transit/encrypt|decrypt/{key}` for the `vault` backend (`src/crypto/vault.rs:35-36`).
- **OsRng** (`rand_core` getrandom) for DEK/nonce/id generation; **zeroize** on DEKs and the local KEK (`src/crypto/mod.rs:99,218`, `local.rs:49-53`).
- Ed25519 / SHA-256 / HKDF-SHA256 / HMAC-SHA256 are used transitively via **peat-mesh** (genesis, certificates, device-id) — all FIPS-approvable primitives.

**No ADR-060 violation originates in this repo.** (The known ChaCha20 exposure is in `peat-btle` / ADR-052, not here — do not attribute it to the gateway.) There is **no explicit ADR-060/FIPS reference** in the gateway's own docs or code (grep for `adr-060|fips` = empty), so the FIPS posture is *implicit by primitive choice*, not asserted. Worth noting for compliance reviewers: the AES-GCM implementation is the pure-Rust `aes-gcm` crate, **not** a FIPS-validated module (e.g. not a CMVP-certified OpenSSL). For an actual FIPS 140 boundary, the KMS or Vault backends (which can be backed by validated HSMs) are the path — the local-KEK path uses non-validated software AES (`src/crypto/local.rs`). This is an honest caveat, not a code defect.

**Open security issue:** decrypted genesis bytes are **not** wrapped in a zeroizing container after `open()` returns (`peat-gateway#55`, OPEN, P2). The DEK is zeroized but the recovered plaintext genesis is a plain `Vec<u8>`.

---

## 7. Envelope encryption (a genuine shipped capability)

`MeshGenesis` authority key material is encrypted at rest via a pluggable `KeyProvider` trait (`src/crypto/mod.rs:37-41`):
- Trait contract: `wrap_dek` / `unwrap_dek` (DEK = data encryption key; KEK = key encryption key).
- Four backends: **Local** AES-256-GCM (`LocalKeyProvider`), **AWS KMS** (`AwsKmsProvider`), **Vault Transit** (`VaultTransitProvider`), and **Plaintext** (dev/test no-op that errors on wrap, `mod.rs:45-55`).
- Selection priority (`build_key_provider`, `mod.rs:124-172`): KMS → Vault → local KEK → plaintext fallback.
- Envelope format is custom, versioned, magic-tagged `PENV` v1 (`mod.rs:57-69`): `[magic][ver][wrapped_dek_len][wrapped_dek][12B nonce][ciphertext+16B GCM tag]`. `open()` returns `None` for non-`PENV` data, enabling transparent **legacy-plaintext → encrypted migration** (`mod.rs:178-185`).
- `migrate-keys` CLI subcommand re-seals existing plaintext genesis records (README.md:120-124, `src/cli.rs`, `src/main.rs:22`).

This is the most polished, best-tested subsystem in the repo (full roundtrip/tamper/wrong-KEK tests, `mod.rs:223-358`).

---

## 8. Public API a consumer integrates against

**Admin API** (bearer-token protected when `PEAT_ADMIN_TOKEN` set; **open in dev mode when unset** — README.md:143). Routes assembled in `src/api/mod.rs:71-93`:

- **Orgs:** `POST/GET /orgs`, `GET/DELETE /orgs/{org_id}` (`src/api/orgs.rs`)
- **Formations:** `POST /orgs/{org}/formations`; read-only `GET .../formations/{app}/peers`, `/documents`, `/certificates` (sourced from registered `MeshBrokerState` or genesis; `src/api/formations.rs:239-258`)
- **Enrollment tokens:** `src/api/tokens.rs` (`peat_`-prefixed tokens)
- **CDC sinks:** `src/api/sinks.rs`
- **Identity:** per-org IdP CRUD, policy rules, audit log (`src/api/identity.rs:162-179`)
- **Enrollment (public, no admin auth):** `POST /orgs/{org}/formations/{app}/enroll` (`src/api/enroll.rs:466-470`) — the node-onboarding entrypoint. Returns `{decision, audit_id, certificate?, mesh_id?, authority_public_key?}`.
- **Health/metrics:** `GET /health`, `GET /metrics` (Prometheus) (`src/api/health.rs:22-26`)
- **Admin UI:** static SvelteKit served at `/_/` when `PEAT_UI_DIR` set (`src/api/mod.rs:48-56`)

**Library surface** (`src/lib.rs`): `api`, `cdc`, `cli`, `config`, `crypto`, `ingress` (nats), `storage`, `tenant` modules. The crate is `#![allow(dead_code)]` with a "Scaffolding — stubs will be wired incrementally" header (`src/lib.rs:1`) — a candor signal that not everything is wired.

**Enrollment policy** (`src/tenant/models.rs:21-29`): `Open` (anyone → Endpoint/0-perms), `Controlled` (enrollment token OR OIDC introspection + policy rules), `Strict` (always 403, admin approval not yet implemented — `src/api/enroll.rs:89-93`). Per-org **rate limiting** via `max_enrollments_per_hour` (default 1000, `models.rs:41-43`, `enroll.rs:78-86`).

---

## 9. Control-plane ingress (NATS) — substantial and well-hardened, but AuthZ is a stub

The `src/ingress/` module (feature `nats`) is real and detailed (ADR-055 Amendment A, peat#842, peat-gateway#91):
- Per-org JetStream durable **push** consumers with subject pattern `{org}.*.ctl.>`; org-level events use reserved sentinel `_org` app_id (`src/ingress/mod.rs:355-437`).
- **DLQ** to `peat-gw-dlq` stream on `max_deliver` exhaustion, with metadata headers and secret-leakage guard rails (`mod.rs:112-159, 858-896`) — peat-gateway#108.
- **Handler panic recovery** via `catch_unwind` so a panicking handler can't orphan the consumer (`mod.rs:730-767`) — peat-gateway#118.
- TOCTOU mitigation on stream-config mutation (process-local mutex + retry; true cross-instance CAS not available) — peat-gateway#105/#92 (`mod.rs:161-193`).

**Honest gaps the code itself flags:**
- **AuthZ is a PERMISSIVE STUB** — every ingress event is allowed; logged loudly at boot (`mod.rs:282-293`). The real policy engine is peat-gateway#99 (OPEN, P2).
- **Broker-level NATS account ACLs** (the primary tenant boundary) are peat-gateway#97; in-process org revalidation is only defence-in-depth (`mod.rs:947-972`).
- **NATS auth (nkeys/JWT/creds) and TLS/mTLS** to the broker are NOT implemented — peat-gateway#124, #125 (both OPEN).
- Peers/certs/idp ingress handlers are stubbed pending #99 (`mod.rs:44-45`).

---

## 10. Quantitative claims — verifiable vs not

All quantitative claims found are in `docs/`, generated by an in-repo loadtest harness (`src/loadtest.rs`, feature `loadtest`).

**Verifiable in-principle (the harness exists and the numbers cite it), but NOT independently reproduced in this audit:**
- Enrollment throughput **~137 req/s** (debug build, AES-256-GCM + Ed25519); "release should 3-5x" (`docs/horizontal-scaling.md:15`). Performance table: enroll p50 71.7ms / p99 142.1ms, **136.8 req/s** (`docs/performance.md:111-116`); read path **10,239 req/s**, p50 0.96ms (`:125-130`); mixed **283.7 req/s** (`:139-144`).
- Scaling limits: >100 formations/instance (~10-50MB Iroh mem each), >500 enroll req/s single-core saturation, >3,000 CDC cursor writes/s redb single-writer limit (`docs/horizontal-scaling.md:42-44`).

These are **self-reported single-run debug-build microbenchmarks**, environment-unspecified. Flag in the curriculum as "vendor's own loadtest harness, debug build, not third-party validated."

**Not present / not verifiable in this repo:** SBD ~1,960B MO / ~1,890B MT; LoRa 7-87km / 1.5-9.1kB/s; BLE 100-400m / ~2Mbps; "93-99% bandwidth reduction"; "<5s P1 latency"; "O(n log n)"; **"1,000+ node" validation.** None of these appear in peat-gateway and none can be sourced here — they belong to other layers (peat-mesh, peat-btle, the SBD/LoRa ADRs). The gateway's own scaling doc speaks in formations-per-instance and req/s, not node counts.

---

## 11. Assumptions register

- **A1 — peat-mesh rc.1 vs rc.35:** The exact pinned `peat-mesh 0.9.0-rc.1` tarball was not extracted in the cargo cache; I verified `MeshGenesis`, `MeshTier`, `issue_certificate`, and `DeviceId` derivation against **rc.35** (closest extracted) and the local `peat-mesh` checkout (rc.42). The security API shape (tier names, SHA-256[0..16] device-id, HKDF-SHA256 genesis) is stable across these. **Consequence if false:** minor signature drift; the *shape* of the findings (tier names, identity derivation) is robust. Logged and proceeding.
- **A2 — stale pin is intentional:** SKILL.md:74 says the `=0.9.0-rc.1` pin is deliberate and bumps need full integration validation. Treated as fact; flag for curriculum that the gateway lags the mesh by ~40 RCs.
- **A3 — performance numbers:** assumed generated by `src/loadtest.rs`; not re-run. Treated as self-reported.
- **A4 — `Strict` enrollment:** admin-approval flow is not implemented; `Strict` simply 403s. Assumed intentional placeholder (no issue found explicitly tracking it).

---

## 12. Gaps / contribution map (this repo)

| Gap | Status | Issue/evidence |
|---|---|---|
| Kafka CDC delivery is a TODO | In flight | `src/cdc/engine.rs:78-80` |
| Redis Streams sink advertised but absent | README overclaim | README.md:19 vs `models.rs:79-84` |
| Ingress AuthZ permissive stub | In flight, P2 | peat-gateway#99 |
| NATS broker ACLs (primary tenant boundary) | In flight, P2 | peat-gateway#97 |
| NATS auth (nkeys/JWT/creds) + TLS/mTLS | Proposed | peat-gateway#124, #125, #92 |
| Decrypted genesis not zeroized | In flight, P2 | peat-gateway#55 |
| Postgres integration tests not in CI | In flight, P2 | peat-gateway#53 |
| Admin DLQ inspection/replay API | Proposed, P2 | peat-gateway#119 |
| No proof-of-possession at enrollment | Gap (no issue found) | `src/api/enroll.rs:273-285` validates pubkey length only |
| `Strict` enrollment policy not implemented | Gap | `src/api/enroll.rs:89-93` |
| Gateway tier maps to only 3 of 4 mesh tiers (Edge unreachable) | Design note | `src/tenant/models.rs:179-188` |

---

## 13. Headline corrections for the curriculum

1. **peat-gateway implements no mesh transports** — it's a control plane, not a node/data-plane. Any module 05 claim implying it carries QUIC/BLE/etc. is wrong.
2. **No ADR-066 hierarchy vocabulary here.** The gateway tier is `Authority/Infrastructure/Endpoint` (RBAC roles), mapping to mesh wire tiers `Enterprise/Regional/Tactical` (distribution tiers). Neither set is `Platform/Cell/Cohort/Federation/Coalition`.
3. **FIPS-clean repo** — AES-256-GCM only, zero ChaCha20. The ChaCha/ADR-060 issue is a *peat-btle* problem; do not attribute it to the gateway. Caveat: local-KEK path uses non-CMVP software AES — for a FIPS boundary use KMS/Vault.
4. **Kafka and Redis CDC sinks are over-advertised** in the README. Kafka is a TODO; Redis doesn't exist.
5. **Ingress AuthZ is a permissive stub** and broker auth/TLS are absent — the NATS control-plane is not production-secure yet despite being feature-rich.
6. **Performance numbers are self-reported debug-build microbenchmarks**; no "1,000+ node" claim exists here.
7. **No proof-of-possession at enrollment** — the gateway trusts the client-supplied public key and node_id.

---

### 2026-06-22 delta — `8d16824 → bece4d6` (crate still 0.1.0)

- **`peat-mesh` pin jumped `=0.9.0-rc.1 → =0.9.0-rc.40`** (Dependabot peat-gateway#144,
  `Cargo.toml:23` + dev-dep). The long-standing "intentionally frozen, ~40 RCs stale by design"
  framing is now obsolete — the gateway lags the ecosystem (rc.43) by only ~3 RCs and is being kept
  current via Dependabot. The bump was **not** a one-line change: it adapted the CDC watcher to the
  newer `ChangeEvent` surface — `Updated`/`Removed` gained an `origin` field, `Initial` a `collection`,
  with a catch-all arm added (`src/cdc/watcher.rs:81,154,200`) — and carried a `redb` 2→4 major
  (`ReadableDatabase` import, `src/storage/redb_backend.rs`). Confirms the "tracking the mesh is real
  integration work" caveat. The CDC **sink set** (NATS/Webhook/Kafka) is unchanged — `src/cdc/engine.rs`,
  the sink modules, and the `CdcSinkType` enum (`src/tenant/models.rs`) are all untouched in
  `8d16824..bece4d6` — so M-027/H-010 facts hold.
- Other commits in range are pure dependency bumps (reqwest 0.12→0.13, tower-http 0.5→0.6,
  rdkafka 0.37→0.39, the UI/actions groups).
