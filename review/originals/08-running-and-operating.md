# Module 8 — Running, Deploying & Operating PEAT

**Goal:** get hands-on. Build and run a real mesh, then learn how PEAT is configured, deployed,
secured, monitored, and troubleshot in production. Sources: the repo's
[`QUICKSTART.md`](../peat/docs/guides/QUICKSTART.md) and
[`OPERATOR_GUIDE.md`](../peat/docs/guides/operator/OPERATOR_GUIDE.md).

> **Want to *see* it work before reading any more theory?** Do §8.1 first — it's a 3-node mesh on
> your laptop in ~10 minutes. Everything else here is the operator's reference.

---

## 8.1 The Quickstart — a real mesh in ~20 minutes

The quickstart uses one tiny, readable example crate — `examples/quickstart` — that produces the
binary `peat-quickstart`. It deliberately has **no formation key, no MLS, no enrollment**: just the
core property (multi-transport eventual consistency) made visible.

```bash
git clone https://github.com/defenseunicorns/peat.git && cd peat
cargo build -p peat-quickstart --release        # binary: target/release/peat-quickstart
./target/release/peat-quickstart --help          # flags: --name --bind --peer --mdns --storage
```

Prereqs: **Rust 1.70+** and **`protoc`** (`apt install protobuf-compiler` / `brew install protobuf`).

### Reading the logs — three event prefixes

Once running, you watch for three log prefixes — learn these, they're how you reason about the mesh:

- `discovery: …` — a peer was *configured* (`--peer`) or *found* (`--mdns`). Not yet reached.
- `connection: peer X connected / reconnected / lost` — transport-level QUIC connection state.
- `sync: NAME (new)` / `sync: NAME (updated) M → N` — **a remote document arrived/changed. This is
  the proof that state is actually replicating.**

A periodic `[peers=N] me:alpha=99 | bravo=98 | …` line is a 2-second heartbeat snapshot (own state
left of `me:`, remote nodes after). The binary defaults to a *quiet* log filter; set
`RUST_LOG=debug` for the full firehose.

### The four scenarios (each builds on the last)

1. **Two nodes, one host, static peers** — fastest proof of sync. Start `alpha` (`--bind
   127.0.0.1:39001`), copy its node id from the log, start `bravo` with
   `--peer <ALPHA_ID>@127.0.0.1:39001`. Within seconds both show `sync:` lines.
2. **Three nodes, static peers — transitive gossip.** `charlie` connects only to `alpha`, yet sees
   `bravo`'s state *via* `alpha` (`sync: bravo (new)` even though bravo isn't a direct peer). `alpha`
   shows `peers=2`. This is the gossip relay from Module 3 §3.4:

   ```mermaid
   flowchart TD
       B["bravo<br/>:39002"] -->|"--peer alpha"| A["alpha (hub)<br/>:39001 · peers=2"]
       C["charlie<br/>:39003"] -->|"--peer alpha"| A
       B <-.->|no direct link · gossip via alpha| C
   ```
3. **Three nodes, mDNS — zero config.** Drop `--peer` entirely, add `--mdns`; nodes advertise as
   `_peat-node._tcp.local` and find each other. *Won't work where multicast is blocked (enterprise
   Wi-Fi, across subnets/VPNs) — fall back to `--peer`.*
4. **Three nodes across two Raspberry Pis + a laptop** — cross-compile with `cross build -p
   peat-quickstart --release --target aarch64-unknown-linux-gnu`, `scp` to the Pis, run. The repo's
   `Cross.toml` already configures the aarch64 target.

> The NodeId is **deterministic from `--name`** (same name ⇒ same id), so you only copy it once.

### Two gotchas worth pre-loading

- **`PEAT_CONNECTION_RECYCLE_SECS`** — there's an iroh memory-growth workaround that recycles every
  QUIC connection at 60s, which for a low-traffic workload turns continuous sync into a ~4–6s outage
  every minute. For demos/light workloads set `PEAT_CONNECTION_RECYCLE_SECS=0` to disable it (default
  `60`; use e.g. `600` for ~10-min sessions). See issues #435 / #892.
- **Stale processes on the Pis** — peers launched over SSH with `nohup … &` survive disconnect; a
  local `pkill` won't reach them. Pre-flight with `ssh pi-a 'pgrep -af peat-quickstart || echo
  CLEAN'` before relaunching, or you get a confusing "port in use → sync stuck at fuel_minutes=0".

---

## 8.2 The binaries (don't mix them up)

| Binary | Where | Purpose |
|--------|-------|---------|
| `peat-quickstart` | `examples/quickstart` | the minimal learning binary above |
| `peat-sim` | the operator guide's main binary (network simulator) | run/simulate multi-node deployments |
| `peat-mesh-node` | `peat-mesh` (feature `node`) | the all-in-one production mesh node (Module 3 §3.1) |

> **Heads-up:** the **operator guide leans on `peat-sim`**, which is **not in this local checkout**
> and not a member of the current `peat` workspace — like `peat-inference`, it appears to live in
> its own repo now (Module 7). Treat the operator-guide commands as the documented operator
> workflow; for a node you can build from this checkout, `peat-mesh-node` (Module 3) is the concrete
> one. Env-var and port conventions also differ per binary (the node binary uses
> `PEAT_FORMATION_SECRET` / `PEAT_BROKER_PORT`; the operator guide uses the set below).

---

## 8.3 Configuration (operator guide)

PEAT is configured by **environment variables** or a **`peat.toml`** file. The must-set trio:

```bash
export PEAT_APP_ID="your-app-id"          # distinguishes logical deployments
export PEAT_SECRET_KEY="your-secret-key"   # shared secret for formation auth
export PEAT_FORMATION_KEY="$(openssl rand -base64 32)"
```

Common variables (operator-guide defaults):

| Variable | Default | Purpose |
|----------|---------|---------|
| `PEAT_PERSISTENCE_DIR` | `./peat_data` | where CRDT state is persisted |
| `PEAT_NODE_ID` | auto UUID | node identity |
| `PEAT_CELL_SIZE` | `5` | target cell size |
| `PEAT_DISCOVERY_MODE` | `mdns` | `mdns` / `static` / `hybrid` |
| `PEAT_BIND_ADDRESS` / `PEAT_BIND_PORT` | `0.0.0.0` / `4040` | P2P |
| `PEAT_HTTP_PORT` | `8080` | HTTP API |
| `RUST_LOG` | `info` | log level (`RUST_LOG=peat_protocol::cell=debug` for per-module) |

A `peat.toml` mirrors these with `[node] [network] [discovery] [cell] [hierarchy] [security]
[storage] [logging] [capabilities]` sections (the operator guide has a full annotated example). For
non-mDNS environments, a `peers.toml` lists `[[peers]]` with `id`/`address`/`port`/`role`.

---

## 8.4 Deployment patterns

- **Single host / dev** — just run the binary (optionally `PEAT_NODE_COUNT=20` for the simulator).
- **Multi-node** — one seed node (`--seed`), others join via `PEAT_STATIC_PEERS=seed-ip:4040`.
- **Edge devices** (Jetson, Pi) — cross-compile, `scp`, run with reduced `PEAT_CELL_SIZE=3`.
- **Containers** — multi-stage `Dockerfile` (rust builder → debian-slim runtime); a `docker-compose`
  with a seed + N nodes on one bridge network.
- **Kubernetes** — a `StatefulSet` (stable pod identities) with `PEAT_DISCOVERY_MODE=static` pointed
  at the headless service DNS (`peat-node-0.peat:4040`), a `volumeClaimTemplate` for state, and a
  headless `Service`. (`peat-mesh` also supports native k8s `EndpointSlice` discovery — Module 3.)

### Ports & networking

| Port | Proto | Purpose |
|------|-------|---------|
| 4040 | UDP/TCP | P2P mesh |
| 8080 | TCP | HTTP API |
| 5353 | UDP | mDNS (multicast) |

Open these in `iptables`/`firewalld`. PEAT uses **Iroh for NAT traversal** (optional STUN/relay
config). It's built for constrained links — bandwidth **profiles** range from `minimal` 9.6 kbps
(tactical radio) → `low` 64 kbps (satellite) → `medium` 256 kbps (cellular) → `standard` 1 Mbps
(Wi-Fi) → `high` 10+ Mbps. Set `bandwidth_limit_kbps` + `qos_enabled = true`. Partitions are handled
automatically: heartbeat-timeout detection → exponential-backoff reconnection → CRDT auto-merge on
heal.

---

## 8.5 Security configuration

- **Formation key** — `openssl rand -base64 32`; set via `PEAT_FORMATION_KEY` or
  `[security] formation_key`. This is the cell-admission secret (Module 2·5 §2·5.4).
- **PKI** — optional X.509 device certs (`[security.pki]` with `ca_cert`/`node_cert`/`node_key`,
  `verify_peer = true`).
- **Encryption** — *operator-guide caveat:* the 2025-12-08 guide names **ChaCha20-Poly1305**. That's
  the **pre-FIPS** cipher; the normative security spec and ADR-060 (amended 2026-05-18) moved the
  posture to **AES-256-GCM** with ECDH P-256/P-384 and TLS/QUIC under a FIPS-mode provider
  (`aws-lc-rs`). When the two disagree, the spec/ADR is current (Module 9 §005).
- **Best practices** the guide stresses: always use formation keys in production, enable TLS for the
  HTTP API, rotate credentials (and the formation key on member departure), use PKI in secure
  environments, audit logs, and segment PEAT traffic.

---

## 8.6 Monitoring & observability

PEAT exposes an **HTTP API** and **Prometheus metrics**:

| Endpoint | Purpose |
|----------|---------|
| `GET /health` | liveness (`{"status":"healthy",…}`) |
| `GET /ready` | readiness (`cell_joined`, `peers_connected`) |
| `GET /metrics` | Prometheus metrics |
| `GET /api/v1/status` · `/peers` · `/cell` · `/capabilities` | node/peer/cell introspection |
| `GET /api/v1/network/partitions` | partition status |

Key metrics: `peat_peers_connected`, `peat_cell_size`, `peat_sync_latency_seconds`,
`peat_messages_{sent,received}_total`, `peat_bandwidth_bytes_total`, `peat_leader_elections_total`.
JSON structured logging (`PEAT_LOG_FORMAT=json`) and OpenTelemetry/OTLP tracing are supported.
Alert on: 0 peers for 5 min (critical), 3× failed leader elections (high), sync latency > 10s
(medium), partition detected (high).

---

## 8.7 CoT / TAK integration (operator view)

PEAT bridges to TAK via Cursor-on-Target translation (`[cot]` config: `bind_port` 8087, tcp/udp/
multicast). Capability→CoT type mappings are configurable (`platform_uav = "a-f-A-M-F-Q"`, etc.),
PEAT emits CoT XML with a `<peat:capability …>`/`<__peat>` detail extension, and it can *receive*
waypoints/targets/chat from TAK with a `command_authority` gate (`C2_ONLY` vs `ANY_TAK_USER`). The
*code* for this lives in `peat-transport/src/tak/` + `peat-protocol/src/cot/` (Modules 2 §2.8, 5).

---

## 8.8 Backup, recovery & troubleshooting

- **State persistence** — `[storage] persistence_dir`, periodic snapshots (`snapshot_interval_seconds`,
  `max_snapshots`). Back up by SIGTERM → `tar` the data dir → restart.
- **Recovery** — restore the tarball, *or* wipe the data dir entirely: a clean node **rejoins and
  re-syncs from peers** (CRDT guarantees convergence). Disaster recovery = redeploy with the same
  config + formation key.
- **Troubleshooting greatest hits** (from both guides):
  - *0 peers / `[peers=0]`* → wrong node id or address, or UDP blocked by firewall; mDNS blocked on
    the LAN (use `--peer`).
  - *Leader election timing out* → unsynced clocks (run NTP), high latency (raise the election
    timeout, shrink the cell).
  - *`connected` but no `sync:` for a few seconds* → normal (1–2 round trips after the QUIC
    handshake); investigate only past ~5s.
  - *Reconnect every ~60s* → the connection-recycle workaround; set `PEAT_CONNECTION_RECYCLE_SECS=0`
    (§8.1).
  - *Persistence/backend errors at startup* → check `PEAT_APP_ID`/`PEAT_SECRET_KEY` set, dir
    writable/not full; if corrupted, archive + clear the dir and let it re-sync.

---

## Try it

1. Run Quickstart Scenarios 1–3 on your laptop. Watch for the `sync:` lines — that's convergence.
2. `curl localhost:8080/health` and `…/api/v1/peers` against a running node.
3. Deliberately kill one node mid-run and watch the others' `connection: … lost` then reconnect.

## Checkpoint

- What do the three log prefixes (`discovery:` / `connection:` / `sync:`) each mean, and which one
  proves replication?
- In Scenario 2, how does `charlie` see `bravo` without a direct connection?
- Which three env vars are mandatory, and what does the formation key gate?
- Which cipher does the operator guide name, and why is it stale vs. the spec?
- After wiping a node's data dir, how does it recover — and why is that safe?

---

Next: [Module 9 — The Protocol Specifications »](09-protocol-specs.md)
