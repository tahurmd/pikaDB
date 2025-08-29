# PikaDB — Masterless Distributed Key–Value Store for Raspberry Pi Pico 2 W (and PC)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](#)
[![Language: C](https://img.shields.io/badge/Language-C-informational)](#)
[![Targets: Pico 2 W + PC](https://img.shields.io/badge/Targets-Pico%202%20W%20%2B%20PC-blue)](#)
[![Status: Design Spec](https://img.shields.io/badge/Status-Design_Spec-green)](#)

A lean, **masterless** distributed **key–value** store that runs on **Raspberry Pi Pico 2 W** and on **PC** for development. Any node can **read, write, delete, and query**. Data is sharded on a **32‑bit consistent‑hash ring** and **replicated**, with lightweight **gossip** for membership and **HLC (Hybrid Logical Clocks)** + **LWW (last‑writer‑wins)** for convergence. Everything is written in **C** with a **single‑threaded event loop** and **bounded** buffers/queues so it fits tiny hardware.

> This README is the **source of truth** for architecture, protocols, operations, and build steps.

---

## 0) Quick Glance

- **Cluster size:** 4–10 peers (Pico 2 W and/or PC)
- **Partitioning:** **32‑bit ring**, **vnodes** per node (Pico: 8, PC: 32–64)
- **Replication:** default **RF=2** (primary + secondary); option to run **RF=3** later
- **Consistency:** eventual; **HLC + LWW**; **read‑repair** for convergence
- **Membership:** **gossip‑lite** (SWIM‑style PING/PONG/SUSPECT/DOWN)
- **Storage:** **pico-kvstore** on‑device (append‑only, batched flush)
- **Networking:** tiny **binary TCP** protocol, 1 in‑flight frame/connection
- **Execution:** single‑threaded, **bounded** sockets/queues; predictable timing

> **Pico 2 W constraints we design for:** \~520 KB SRAM (target node ≤\~300 KB), 4 MB external QSPI flash, 2.4 GHz Wi‑Fi. Prefer infrastructure Wi‑Fi for clusters; SoftAP only for bring‑up. Keep pages/frames small; schedule heavy work under stable power.

---

## 1) Why these choices?

| Area          | Choice                                   | Why                                                             |
| ------------- | ---------------------------------------- | --------------------------------------------------------------- |
| Topology      | Masterless peers                         | No single bottleneck; any node can serve                        |
| Ring          | **32‑bit** consistent hash + vnodes      | Smooth balance even at 5–10 nodes; avoids 16‑bit crowding       |
| Replication   | RF=2 (option RF=3)                       | Good resilience vs. flash/CPU overhead on microcontrollers      |
| Consistency   | HLC + LWW                                | Causality with O(1) metadata; simple conflict resolution        |
| Membership    | Gossip‑lite (SWIM‑style)                 | Tiny message cost; quick failure detection; simple to implement |
| Storage       | pico-kvstore + batched flush             | Flash‑friendly, crash‑safe, tiny footprint                      |
| Protocol      | Binary TCP, 1 frame in‑flight            | Backpressure is trivial; small, deterministic buffers           |
| Eventing      | Single‑threaded                          | Fewer edge cases; suits Pico 2 W; easy to reason about          |
| Observability | READY / STATS / RING / NODES / QUERYNODE | Make behavior visible for learning & debugging                  |

---

## 2) Repo Layout (planned)

```
pikadb/
├─ Makefile
├─ README.md              # this doc
├─ include/               # public headers
├─ src/
│   ├─ main.c             # event loop + dispatch
│   ├─ ring.c/.h          # 32-bit ring, vnodes, owners
│   ├─ hlc.c/.h           # Hybrid Logical Clock
│   ├─ net.c/.h           # TCP accept/send/recv (bounded)
│   ├─ proto.c/.h         # frame encode/decode
│   ├─ store.c/.h         # pico-kvstore adapter + metadata
│   ├─ peers.c/.h         # seeds, gossip-lite, node table
│   ├─ repl.c/.h          # replication, hinted handoff
│   ├─ repair.c/.h        # read-repair (rate-limited)
│   ├─ migrate.c/.h       # ring rebalancing (paged, throttled)
│   ├─ metrics.c/.h       # counters + READY/STATS JSON
│   └─ util.c/.h          # crc32c, fmix32, time, rand, helpers
├─ tools/
│   └─ pikactl.py         # CLI: put/get/del/scan/nodes/ring/querynode/stats/ready
├─ vendor/
│   └─ pico-kvstore/       # storage engine
├─ tests/
│   ├─ unit/              # ring, hlc, proto
│   └─ cluster/           # ping/kv/scan/rebalance/chaos
└─ scripts/
    ├─ cluster.sh         # spawn/kill N local nodes (PC)
    └─ chaos.sh           # partitions/failures (PC)
```

---

## 3) Architecture Diagrams

### 3.1 Cluster (Masterless)

```
                   ┌───────────────────────────────────┐
                   │            Access Point           │
                   │         (infrastructure Wi‑Fi)    │
                   └───────────────────────────────────┘
          ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
Client ──▶│  Node A      │──▶│  Node B      │──▶│  Node C      │◀── Client
          │ (Pico/PC)    │   │ (Pico/PC)    │   │ (Pico/PC)    │
          │──────────────│   │──────────────│   │──────────────│
          │ Event Loop   │   │ Event Loop   │   │ Event Loop   │
          │ Protocol     │   │ Protocol     │   │ Protocol     │
          │ Ring/Vnodes  │   │ Ring/Vnodes  │   │ Ring/Vnodes  │
          │ Replication  │   │ Replication  │   │ Replication  │
          │ Gossip       │   │ Gossip       │   │ Gossip       │
          │ pico-kvstore  │   │ pico-kvstore  │   │ pico-kvstore  │
          └──────────────┘   └──────────────┘   └──────────────┘
```

### 3.2 Single‑Node Stack (Pico 2 W / PC)

```
┌─────────────────────────────────────────────────────────────┐
│                         PikaDB Process                      │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │ Event Loop   │───▶│  Dispatcher  │───▶│  Handlers    │   │
│  └──────────────┘    └──────────────┘    └─────┬────────┘   │
│                                                │            │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────▼─────────┐  │
│  │  net.c       │  │ proto.c      │  │  ring.c           │  │
│  │ sockets, I/O │  │ frame parse   │  │ owners(key)       │  │
│  └──────────────┘  └──────────────┘  └─────────┬─────────┘  │
│                                                │            │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────▼─────────┐  │
│  │ repl.c       │  │ repair.c     │  │ store.c (pikaKV)  │  │
│  │ replicate/hint│ │ read‑repair   │  │ WAL + metadata    │  │
│  └──────────────┘  └──────────────┘  └─────────┬─────────┘  │
│                                                │            │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────▼─────────┐  │
│  │ peers.c      │  │ hlc.c        │  │ metrics.c         │  │
│  │ seeds/gossip │  │ clocks/LWW    │  │ READY/STATS JSON  │  │
│  └──────────────┘  └──────────────┘  └────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 4) Cluster Identity & Bootstrap

- **cluster_id (128‑bit)** and **cluster_salt (32‑bit)** define the ring identity.
- **Genesis (recommended):** start exactly **one node** with `--bootstrap` to mint and persist `cluster_id` and `cluster_salt` in `_sys/cluster.meta`.
- **Join:** all other nodes use `--seed ip:port` to fetch the cluster meta and adopt it. A **join token** (shared string) avoids accidental cross‑joins on the same LAN. **Nodes refuse JOIN if `join_token` mismatches.**
- **Immutability:** treat `cluster_salt` as **immutable**. Changing it = a new cluster.

**Bootstrap example**

```
pikadb --node-id 1 --port 9101 --bootstrap --join-token PIKA2025
```

**Join example**

```
pikadb --node-id 2 --port 9102 --seed 192.168.1.10:9101 --join-token PIKA2025
```

**JOIN_REQ → JOIN_ACK fields** (conceptual)

- `JOIN_REQ { node_id, build_profile, features, join_token }`
- `JOIN_ACK { accept, cluster_id, cluster_salt, rf, vnodes_per_node, ring_version, ring_snapshot[], nodes[] }`

**Node lifecycle**

```
STANDBY → JOINING → ALIVE
  ^           │         │
  └───────────┴─────────┘ (persist cluster.meta / gossip)
```

**Seeds are discovery only:** proxying and routing always use the **peers table (ID→addr\:port)** learned from seeds/gossip; ports are never inferred.

## 5) Data Model & Limits

- **Key:** up to **64 B** (binary‑safe)
- **Value:** up to **512 B** on Pico builds (PC builds may allow larger later)
- **Per‑entry metadata:** HLC (phys_ms, logical, node_id), flags (active/tombstone), size, CRC32C
- **Reserved prefixes:** `_sys:`, `_ring:`, `_hint:`, `_gossip:` (not for user data)

---

## 6) 32‑bit Ring, Vnodes, Tokens

- Hash space: **0..2^32‑1**
- Each node has **vnodes** (Pico: 8; PC: 32–64). For vnode index `i`:

  - `entropy = crc32c({node_id,i})`, `salt = (node_id<<8)|i`
  - `token = fmix32(entropy ^ fmix32(salt) ^ cluster_salt)`

- Persist vnode tokens on first boot and reuse on restart.
- Sort all vnodes by token; owners for key `k` are chosen by walking clockwise until **RF** distinct nodes are selected.
- Guardrail: compute ownership % and **CV (stddev/mean)**; warn if `CV > 0.15`.

**Commands**

- **RING** → `{version, vnodes_per_node, items:[{node_id, ranges:[[start,end],...], percent}]}`
- **QUERYNODE <key>** → `{hash, owners:[{id,addr,port}], primary, replicas}`

---

## 7) Time & Conflicts — HLC + LWW

- HLC = `(phys_ms, logical, node_id)`
- **FUTURE_MAX_MS = 2000**: if a received `phys_ms > now + FUTURE_MAX_MS`, **clamp** to `now`, **bump logical**, and increment `clock_skew_events` metric.
- **Send:** if `now_ms > last.phys` → `logical=0`, else `logical++`; `phys = max(last.phys, now_ms)`
- **Receive:** never move local `phys` backwards; if remote ahead (within window) → adopt and set `logical = remote.logical + 1`; else bump local logical
- **Compare:** lexicographic `(phys, logical, node_id)`; higher wins (**LWW**)
- **Policy:** PUT vs PUT → newer wins; PUT vs DEL → newer (tombstone) wins

---

## 8) Storage — pico-kvstore

- Append‑only **WAL** with per‑record CRC32C
- **Batched flush** (e.g., every 25–100 ms or N bytes) to protect flash
- **Snapshot/compaction**: periodic; only under stable power (“green window”)
- **GC tombstones**: horizon ≥ **30–60 min** to prevent “zombie” resurrect across partitions
- **Stats**: `wal_bytes_pending`, `flush_count`, `flush_p95_ms`, `compaction_seconds_today`

---

## 9) Protocol — Binary, Length‑Prefixed

**Frame:** `[ LEN(2) | TYPE(1) | FLAGS(1) | BODY … | CRC32C(4) ]`

- `LEN` counts from `TYPE` through `CRC32C`
- Max frame \~600 B (fits tiny TCP windows)
- **One in‑flight** frame per connection (natural backpressure)
- **Versioning:** first server reply **echoes `proto_ver`** so clients can verify supported features.

**Client‑visible commands**

| Command     | Body                           | Reply                                                                                                |
| ----------- | ------------------------------ | ---------------------------------------------------------------------------------------------------- |
| `PUT`       | `[klen:1][key][vlen:2][value]` | `OK` or `RETRY`                                                                                      |
| `GET`       | `[klen:1][key]`                | `VALUE [vlen:2][value]` or `NOTFOUND`                                                                |
| `DEL`       | `[klen:1][key]`                | `OK`                                                                                                 |
| `SCAN`      | `[plen:1][prefix][limit:2]`    | JSON page; use `SCAN_CONT` for next page                                                             |
| `NODES`     | —                              | JSON `{id,addr,port,state,incarnation,last_seen_ms}`                                                 |
| `QUERYNODE` | `[klen:1][key]`                | JSON `{hash, owners:[{id,addr,port}], primary, replicas}`                                            |
| `RING`      | —                              | JSON `{version,vnodes_per_node,items:[{node_id,ranges,percent}]}`                                    |
| `ERR`       | `{code:string,msg:string}`     | Signals client‑actionable errors: `VALUE_TOO_LARGE`, `BAD_KEY`, `OWNER_UNREACHABLE`, `BUSY`, `RETRY` |
| `READY`     | —                              | JSON `{ready,can_read,can_write,profile,rf,wl,rl,vnodes,ring_version,gc_horizon_s}`                  |
| `STATS`     | —                              | JSON counters/latency summaries                                                                      |

**Error surface:** server returns `ERR{code,msg}` for client‑actionable failures (size, bad key, owner unreachable, busy); **do not use `NOTFOUND` for transport errors.**

(Internals also use `REPL`, `HANDOFF` (hint), `REPAIR`, and gossip frames.) `REPL`, `HANDOFF` (hint), `REPAIR`, and gossip frames.)

---

## 10) Request Paths

### 10.1 Write (PUT)

```
Client → Any node (Coordinator)
1) Compute owners for key (RF=2 default).
2) Stamp HLC; persist locally by **WAL append (not necessarily flash flush)**.
3) Async replicate to the other owner:
   - If reachable → apply & ack internally.
   - If down → enqueue HINT (bounded cap + TTL).
4) ACK client after local WAL append (W=1 default).
   (Optional mode: --wl QUORUM → ACK after both owners persist; recommended on PC.)
```

Client → Any node (Coordinator)

1. Compute owners for key (RF=2 default).
2. Stamp HLC; persist locally (WAL append).
3. Async replicate to the other owner:

   - If reachable → apply & ack internally.
   - If down → enqueue HINT (bounded cap + TTL).

4. ACK client after local persist (W=1 default).
   (Optional mode: WL=QUORUM → ACK after both owners persist.)

```

### 10.2 Read (GET)
```

Client → Any node

1. If local fresh copy → return VALUE.
2. On miss, proxy using peers table (ID→addr\:port learned from seeds/gossip):
   a) Try PRIMARY owner.
   b) If PRIMARY unreachable, try SECONDARY (one hop failover).
   c) Only if both unreachable → return NOTFOUND (with ERR\:OWNER_UNREACHABLE optional).
3. If multiple replies arrive, compare HLCs → return newest (LWW).
4. If divergence observed → enqueue READ‑REPAIR (rate‑limited).

```

### 10.3 Delete (DEL)
```

Client → Any node

1. Write tombstone with fresh HLC; persist locally.
2. Async replicate (or hint if down).
3. ACK client; GC removes data+tombstone after horizon.

````

### 10.4 Prefix scan (SCAN)
- Paged/streaming: coordinator requests small pages from owners; yields between pages to keep loop responsive.
- De‑dup by key at coordinator (keep only newest by HLC).
- **Pico profile caps:** enforce `limit ≤ 50` and total response `≤ 4096 B`; **requests exceeding caps are truncated and the reply includes `"partial": true`.**
- **PC profile:** may return a `next_start` token for resuming large scans.

---

## 11) Replication, Hinted Handoff, Read‑Repair

- **Replication:** coordinator sends to each non‑local owner (bounded concurrency)
- **Hints:** per‑target queue, cap by items and bytes (e.g., 256 items / 64 KB), TTL ~10–15 min; drain on recovery at limited rate (e.g., ≤2 in‑flight, ≤200 items/s)
- **Read‑repair:** token bucket (e.g., 10 repairs/s) **per key‑range**; suppress repeated repairs on same key for ≥60 s
- **Idempotency:** bounded LRU (e.g., 256 entries) over `(origin_id, seq)` to drop replayed PUT/DEL during reconnects.

---

## 12) Membership — Gossip‑Lite

- **States:** `ALIVE → SUSPECT → DOWN`
- **Timers (Pico default):** period = 1000 ms; suspect = 5000 ms; down = 15000 ms
- **Preset `wifi`:** more tolerant links → suspect = 8000 ms; down = 30000 ms (documented preset)
- **Incarnation** increments on restart; higher incarnation wins conflicts
- **Seeds:** at least one `--seed ip:port` to join; **join token** (string) prevents accidental cross‑joins
- **Protocol version handshake:** initial `HELLO{proto_ver,features}` (or proto_ver in first reply) so frames can evolve safely.
- **NODES** returns the current view

---

## 13) Rebalancing (Add/Remove Nodes)

- Ring version bumps; **migrate** only the affected ranges
- **Owner‑pull** migration: new owner pulls from old owner in **small pages**
- Throttles: ≤4 keys in flight; cap KB/s; always yield to foreground ops
- `STATS` exposes migration progress (moved/remaining/ETA)

---

## 14) Backpressure & Flow Control

- Single in‑flight frame per connection
- If send would block: retry next tick once; then reply `RETRY` (and count it)
- Per‑tick budgets: limit work per loop (max frames handled, max store ops, max hints sent, max repairs) to avoid starvation

---

## 15) Observability (ship these surfaces)

### 15.1 READY (snapshot)
```json
{
  "ready": true,
  "can_read": true,
  "can_write": true,
  "profile": "pico|pc",
  "cluster_id": "b839…",
  "rf": 2,
  "wl": "W1|QUORUM",
  "rl": "R1|QUORUM",
  "vnodes": 8,
  "ring_version": 7,
  "ring_cv": 0.08,
  "gc_horizon_s": 3600,
  "features": ["join_token","paged_scan"],
  "cluster_salt": "0xA5A5A5A5"
}
````

### 15.2 STATS (rolling counters)

- `puts/gets/dels`, `put_p50/p95_ms`, `get_p50/p95_ms`
- `repl_sent/recv/failed`, `repl_lag_p95_ms`
- `hints_enqueued/delivered/expired/dropped`, `max_hint_age_s`
- `read_repair_applied`, `repair_rate_limited`
- `net_send_drop_full`, `proto_retry`
- `wal_bytes_pending`, `wal_age_ms`, `flush_count`, `flush_p95_ms`, `compaction_seconds_today`
- `ring_cv`, `ring_version`, `reachable_replicas`

### 15.3 NODES / RING / QUERYNODE

- **NODES**: `[ {id, addr, port, state, incarnation, last_seen_ms} ]`
- **RING**: `{version, vnodes_per_node, items:[{node_id, ranges, percent}]}`
- **QUERYNODE <key>**: `{hash, owners:[{id,addr,port}], primary, replicas}`

---

## 16) Defaults & Flags

```
# Cluster & ring
--node-id <0..255>
--port <tcp>
--bootstrap                 # only for the first node (genesis)
--seed <ip:port>            # repeatable for joining nodes
--join-token <string>
--rf 2                      # 2 or 3
--vnodes 8|32               # pico|pc
--cluster-salt 0xA5A5A5A5   # optional if preconfig; else learned from seed

# Consistency
--wl W1|QUORUM              # default W1
--rl R1|QUORUM              # default R1

# Store
--flush-interval-ms 25
--flush-bytes 4096
--gc-horizon-s 3600
--snapshot-interval-s 300

# Gossip
--gossip-period-ms 1000
--gossip-suspect-ms 5000
--gossip-down-ms 15000

# Hints & repair
--hint-cap-items 256
--hint-cap-bytes 65536
--hint-ttl-s 900
--hint-drain-rps 200
--hint-inflight 2
--repair-rate-per-range 10  # ops/s
--repair-suppress-ms 60000

# Limits
--key-max 64
--value-max 512             # pico build
--clients-max 8             # pico; 32 on pc
```

---

## 17) Resource Budget (Pico target ≤ \~300 KB)

|                             Region |      Budget |
| ---------------------------------: | ----------: |
|                 App stack + locals |     \~80 KB |
|         Network buffers (≤8 conns) |     \~60 KB |
| Metadata cache (\~400 keys × 32 B) |     \~13 KB |
|                  Ring/vnode tables |      \~2 KB |
|                  Gossip node table |      \~1 KB |
|                        Hints queue |      \~8 KB |
|                      Metrics/state |      \~2 KB |
|                 HLC/protocol state |      \~1 KB |
|                  **Safety margin** | **≥120 KB** |

---

## 18) Ops Runbook

**Create (bootstrap) a cluster**

```
pikadb --node-id 1 --port 9101 --bootstrap --join-token PIKA2025
```

**Join a cluster**

```
pikadb --node-id 2 --port 9102 --seed 192.168.1.10:9101 --join-token PIKA2025
```

**Observe**

```
pikactl host:port nodes
pikactl host:port ring
pikactl host:port querynode mykey
pikactl host:port stats
pikactl host:port ready
```

**CRUD**

```
pikactl host:port put k1 v1
pikactl host:port get k1
pikactl host:port del k1
pikactl host:port scan sensor: 50
```

---

## 19) Testing & Acceptance

**Functional**

- CRUD from any node; GET returns VALUE if **≥1 owner reachable via proxy**
- Node down: writes still ACK; hints grow then drain on return
- Add/remove: ring version increments; migration bounded; ops continue

**Performance (indicative)**

- Local GET p95 ≤ few ms (cache hit); cross‑node ≤ tens of ms on Pico Wi‑Fi
- PUT ACK p95 ≤ tens of ms in batched mode; WAL pending bounded

**Health**

- Fail fast at startup if `ring_cv > 0.20` unless `--allow-unbalanced`
- No unbounded queues (hints, repairs, sockets)
- `ring_cv ≤ 0.15` on typical 3–5 nodes with vnodes defaults
- GC horizon respected; no zombie keys after partitions

---

## 20) Roadmap (opt‑in later)

- **WL=QUORUM default** profile for stricter durability
- **Chunked values** (PC only) for >512 B values
- **Light anti‑entropy sweep** (idle only)
- **TLS/PSK** between nodes (PC first)
- **Visualizer** for ring/membership on PC

---

## 21) Known Trade‑offs (documented)

- **W=1 ACK** can lose a write if the coordinator dies before replicate/hint → toggle **WL=QUORUM** when you need stronger durability.
- **Eventual consistency** means a brief window of divergence until read‑repair/hinting converge replicas.
- **Small value cap (512 B on Pico)** is intentional for predictable RAM/MTU; lift on PC builds later.
