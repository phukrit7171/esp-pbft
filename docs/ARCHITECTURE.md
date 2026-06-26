# esp-pbft — Architecture

> **Status:** 🟡 Draft
> **Scope:** System overview, component layering, sequence diagrams, state machines
> **Out of scope:** Detailed crypto internals → [CRYPTO.md](./CRYPTO.md), wire format → [PROTOCOL.md](./PROTOCOL.md)

---

## 1. System overview

esp-pbft is a **3-layer PBFT consensus core** for a 7-node ESP32-C3 cluster. The library exposes a generic `submit(tx) → callback(tx, sequence)` API and is transport-agnostic (ESP-NOW or Wi-Fi UDP multicast, compile-time select).

### 1.1 Cluster topology

```
              7-node cluster (f = 2, quorum = 5)
   ┌──────────────────────────────────────────────────────┐
   │                                                      │
   │   Node 0 (primary)         ◀──  rotating  ──▶        │
   │   ─── broadcast Pre-Prepare                          │
   │   ─── collect Prepare (≥2f+1 = 5)                    │
   │   ─── collect Commit (≥2f+1 = 5)                     │
   │   ─── fire commit_callback(seq, tx)                  │
   │                                                      │
   │   Node 1..6 (replicas)     ◀──  monitor primary  ──▶ │
   │   ─── receive Pre-Prepare                            │
   │   ─── send Prepare                                   │
   │   ─── send Commit                                    │
   │   ─── apply TX to app on commit                      │
   │                                                      │
   └──────────────────────────────────────────────────────┘
                          │
                          │ transport (compile-time Kconfig)
                          ▼
        ┌──────────────────────────────────────┐
        │  ESP-NOW (default)  |  Wi-Fi UDP     │
        │  peer-to-peer       |  AP + IGMP     │
        │  80 mA, no AP       |  160 mA, AP    │
        └──────────────────────────────────────┘
```

### 1.2 Layered architecture (3 layers)

```
┌─────────────────────────────────────────────────────────┐
│ Layer 3: APPLICATION (user code)                          │
│   ── submit(tx)                                           │
│   ── register commit_callback(tx, sequence)              │
│   ── get state / view / sequence / metrics               │
└────────────────────────┬────────────────────────────────┘
                         │ public API (pbft.h)
┌────────────────────────┴────────────────────────────────┐
│ Layer 2: CONSENSUS CORE (public + internal modules)      │
│                                                         │
│  pbft_consensus.c   ←── 3-phase state machine            │
│  pbft_viewchange.c  ←── primary rotation                 │
│  pbft_checkpoint.c  ←── stable checkpoint + GC           │
│  pbft_log.c         ←── in-memory log (last 100 TXs)     │
└────────────────────────┬────────────────────────────────┘
                         │ internal API
┌────────────────────────┴────────────────────────────────┐
│ Layer 1: SERVICES                                         │
│                                                         │
│  pbft_crypto.c       ←── Pattern Y: TRNG + ECDH + HMAC   │
│  pbft_network.c      ←── transport abstraction            │
│    ├── pbft_network_espnow.c                             │
│    └── pbft_network_wifi_udp.c                           │
│  pbft_storage.c      ←── NVS (config, no keys)           │
│  pbft_membership.c   ←── 7-node table + peer cache       │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Boot sequence

### 2.1 First-time boot (cold start)

```
   ┌──────────┐         ┌──────────┐         ┌──────────┐
   │ Node X   │         │ Node Y   │         │ Node Z   │
   └────┬─────┘         └────┬─────┘         └────┬─────┘
        │                    │                    │
   t=0  │ TRNG keygen        │                    │
        │ (5 ms)             │                    │
        ▼                    │                    │
   ECDSA P-256 keypair       │                    │
   (my_priv, my_pub)         │                    │
        │                    │                    │
        │ random delay       │                    │
        │ 0-3s (staggered)   │                    │
        │                    │                    │
   t≈1  │ broadcast Hello    │                    │
        │ (my_pub, node_id)  │                    │
        ├───────────────────▶│ broadcast Hello    │
        │                    ├───────────────────▶│
        │                    │                    │
        │ cache pubkey       │ cache pubkey       │
        │ (TOFU: first wins) │ (TOFU: first wins) │
        │                    │                    │
        │ ECDH(my_priv,      │ ECDH(my_priv,      │
        │   peer_pub_Y)      │   peer_pub_X)      │
        │   = shared_XY      │   = shared_YX      │
        │                    │                    │
        │ HMAC-SHA256(       │ HMAC-SHA256(       │
        │   shared_XY,       │   shared_YX,       │
        │   "PBFT-HMAC-v1")  │   "PBFT-HMAC-v1")  │
        │   = hmac_XY        │   = hmac_YX        │
        │                    │                    │
        ▼                    ▼                    ▼
   Store 6 HMAC keys     Store 6 HMAC keys    Store 6 HMAC keys
   in RAM (192 B)        in RAM (192 B)       in RAM (192 B)
        │                    │                    │
   t≈3  │ PBFT ready          │ PBFT ready         │ PBFT ready
        │                    │                    │
```

### 2.2 Boot mitigation

- **Staggered boot** — random 0–3s delay before first Hello broadcast
- **Why:** 7 nodes booting simultaneously → 7 simultaneous broadcasts → ~21 redundant Hellos per receiver → risk of ESP-NOW broadcast storm
- **Trade-off:** cluster ready in ≤3s + handshake (worst case ≈ 5s)

### 2.3 Soft re-handshake (Pattern Y-5, every 24h)

```
   t=0h      Node 0 boots normally
             ...
   t=24h     Node 0 timer fires (its offset = 0)
             ─────────────────────────────────────
   t=24h+ε   Node 0 generates new ECDSA keypair
             (TRNG, 5 ms, RAM only)
                  │
                  ▼
             broadcast Hello (new_pub, node_id=0)
             + re-handshake marker in payload
                  │
                  ├──────────▶  Node 1: cache new pubkey
                  ├──────────▶  Node 2: cache new pubkey
                  │            ...
                  └──────────▶  Node 6: cache new pubkey
                                  │
                                  ▼
                            Each peer:
                            ECDH(my_priv, new_pub_0) = new_shared
                            HMAC-SHA256(new_shared, "PBFT-HMAC-v1")
                              = new_hmac[0]
                                  │
                                  ▼
                            Update hmac_keys[0] in RAM
                                  │
                                  ▼
                            Re-handshake complete (~50 ms)
```

**Compromise window:** bounded by re-gen period (24h), not by node uptime.

**Note:** Self re-gen and peer re-gen are independent. Node 0's re-gen triggers peers to re-ECDH with node 0; it does NOT trigger peers to re-ECDH with each other.

---

## 3. Consensus sequence (steady state)

### 3.1 Normal request flow (primary honest)

```
   Client    Primary (p)    Replicas (1..6)    App
     │            │                │           │
     │ submit(tx) │                │           │
     ├───────────▶│                │           │
     │            │ Pre-Prepare    │           │
     │            │ (v, n, d, tx)  │           │
     │            ├───────────────▶│           │
     │            │                │ Verify HMAC
     │            │                │ Accept    │
     │            │                │           │
     │            │ Prepare        │           │
     │            │ (v, n, d)      │           │
     │            │◀───────────────┤           │
     │            ├───────────────▶│           │
     │            │  ... (all nodes broadcast)  │
     │            │                │           │
     │            │ collect Prepare (≥2f+1=5)  │
     │            │                │           │
     │            │ state → PREPARED (own)     │
     │            │                │           │
     │            │ Commit         │           │
     │            │ (v, n, d)      │           │
     │            │◀───────────────┤           │
     │            ├───────────────▶│           │
     │            │  ... (all nodes broadcast)  │
     │            │                │           │
     │            │ collect Commit (≥2f+1=5)   │
     │            │                │           │
     │            │ state → COMMITTED (own)    │
     │            │                │ state → COMMITTED
     │            │                │ commit_callback(tx, n)
     │            │                ├──────────▶│
     │            │                │           │ apply TX
     │            │                │           │
```

### 3.2 Message sequence numbers

| Field | Size | Purpose |
|-------|------|---------|
| `view` | u32 | Which primary is current leader |
| `sequence` | u64 | Monotonic TX order (assigned by primary) |
| `digest` | 32 B | SHA-256 of TX payload (request id) |
| `mac` | 32 B | HMAC-SHA256 over `(view ‖ seq ‖ type ‖ digest ‖ payload)` |

### 3.3 State machine (per request, per replica)

```
                         receive Pre-Prepare
                                 │
                                 ▼
              ┌──────────────────────────────────┐
              │  IDLE                            │
              └──────────────┬───────────────────┘
                             │ verify HMAC + accept Pre-Prepare
                             │ → send Prepare
                             ▼
              ┌──────────────────────────────────┐
              │  PRE_PREPARED                    │
              └──────────────┬───────────────────┘
                             │ collect ≥2f+1 matching Prepare
                             ▼
              ┌──────────────────────────────────┐
              │  PREPARED                        │
              └──────────────┬───────────────────┘
                             │ → send Commit
                             │ (already in PREPARED locally,
                             │  just send Commit broadcast)
                             ▼
              ┌──────────────────────────────────┐
              │  PREPARED + sent Commit          │
              └──────────────┬───────────────────┘
                             │ collect ≥2f+1 matching Commit
                             ▼
              ┌──────────────────────────────────┐
              │  COMMITTED                       │
              │  → fire commit_callback(tx, n)   │
              └──────────────┬───────────────────┘
                             │ apply to app + log entry
                             ▼
              ┌──────────────────────────────────┐
              │  EXECUTED                        │
              │  → mark for checkpoint cleanup   │
              └──────────────────────────────────┘

   Transition on view-change (any state):
     PRE_PREPARED / PREPARED → reset to IDLE (new view)
     COMMITTED / EXECUTED → KEEP (carries over to new view)
```

### 3.4 Primary's state machine (simpler)

```
   receive submit(tx) from app
                │
                ▼
   assign sequence = next_seq()
                │
                ▼
   broadcast Pre-Prepare(v, sequence, digest, tx)
                │
                ▼
   collect ≥2f+1 Prepare (own + replicas)
                │
                ▼
   broadcast Commit(v, sequence, digest)
                │
                ▼
   collect ≥2f+1 Commit
                │
                ▼
   fire commit_callback(tx, sequence)
                │
                ▼
   next_seq++
```

---

## 4. Component dependencies

```
pbft.c (public API)
  ├── pbft_consensus.c
  │     ├── pbft_log.c
  │     ├── pbft_crypto.c ── ESP-IDF mbedtls
  │     ├── pbft_network.c ─┐
  │     │                    ├── pbft_network_espnow.c (if CONFIG_PBFT_TRANSPORT_ESP_NOW)
  │     │                    └── pbft_network_wifi_udp.c (if CONFIG_PBFT_TRANSPORT_WIFI_UDP)
  │     ├── pbft_membership.c
  │     └── pbft_storage.c ── ESP-IDF NVS
  ├── pbft_viewchange.c
  │     └── pbft_consensus.c (read state)
  ├── pbft_checkpoint.c
  │     └── pbft_log.c (cleanup)
  └── pbft_log.c
```

**Dependency rule:** no cycles. Public API depends on consensus, consensus depends on services (crypto, network, storage).

---

## 5. Data flow (steady state, single request)

```
                        ┌────────────────────────────┐
                        │        pbft.c               │
                        │   pbft_submit(tx) called    │
                        └─────────────┬──────────────┘
                                      │ enqueue TX
                                      ▼
                        ┌────────────────────────────┐
                        │    pbft_consensus.c         │
                        │  if primary:               │
                        │   assign seq, build PrePrep │
                        │  build signed packet       │
                        └─────────────┬──────────────┘
                                      │
                                      ▼
                        ┌────────────────────────────┐
                        │    pbft_crypto.c            │
                        │   psa_mac_compute(         │
                        │     key=hmac_keys[peer],   │
                        │     input=view||seq||...   │
                        │   ) → 32-byte MAC          │
                        └─────────────┬──────────────┘
                                      │
                                      ▼
                        ┌────────────────────────────┐
                        │    pbft_network.c           │
                        │   send(broadcast, packet)   │
                        │   → ESP-NOW | Wi-Fi UDP    │
                        └─────────────┬──────────────┘
                                      │
                                      │ UDP packet (broadcast)
                                      ▼
                                [7 ESP32-C3]
                                      │
                                      ▼
                        ┌────────────────────────────┐
                        │    pbft_network.c (rx)      │
                        │   on packet received        │
                        │   dispatch to consensus     │
                        └─────────────┬──────────────┘
                                      │
                                      ▼
                        ┌────────────────────────────┐
                        │    pbft_crypto.c            │
                        │   psa_mac_verify(...)       │
                        │   accept or reject         │
                        └─────────────┬──────────────┘
                                      │
                                      ▼
                        ┌────────────────────────────┐
                        │    pbft_consensus.c         │
                        │   state machine:           │
                        │   Pre-Prepare → Prepare    │
                        │   → Commit → Callback      │
                        └─────────────┬──────────────┘
                                      │
                                      ▼
                        ┌────────────────────────────┐
                        │    Application              │
                        │   commit_callback(tx, seq) │
                        │   → apply to app state     │
                        └────────────────────────────┘
```

---

## 6. Concurrency model

- **Single-threaded** for v1 (FreeRTOS task = `pbft_task`, priority 5)
- **Event loop** driven by ESP-IDF `esp_event`:
  - Network RX events → dispatch to consensus
  - Timer events (view-change timeout, Y-5 re-gen) → dispatch to relevant module
  - App `submit()` calls → enqueue to consensus TX queue
- **No locks** (single-threaded = no shared-state races)
- **Trade-off:** simpler code, no priority inversion risk; max throughput limited by single-core CPU

See [API-REFERENCE.md](./API-REFERENCE.md) §thread-safety for exact contract.

---

## 7. View-change (preview)

When a replica suspects the primary is faulty (Prepare timeout, or `2f+1` ViewChange messages received):

```
   Replica R detects primary fault
                │
                ▼
   broadcast VIEW-CHANGE(v+1, R, last_stable_checkpoint_proof)
                │
                ▼
   collect ≥2f+1 VIEW-CHANGE messages for v+1
                │
                ▼
   New primary p_new = (v+1) mod n
                │
                ▼
   p_new builds NEW-VIEW(v+1, V-set, O-set)
   - V-set = set of received VIEW-CHANGE messages
   - O-set = Pre-Prepare messages to re-propose (from V-set's checkpoints)
                │
                ▼
   broadcast NEW-VIEW(v+1)
                │
                ▼
   Replicas verify NEW-VIEW (≥2f+1 VIEW-CHANGE proofs)
                │
                ▼
   State: view = v+1, p_new becomes primary
   Resume consensus from O-set
```

Full details in [VIEW-CHANGE.md](./VIEW-CHANGE.md) (TBD).

---

## 8. Checkpoint (preview)

Every `K` sequences (e.g., K=100), replicas broadcast CHECKPOINT messages with state digest:

```
   seq reaches K * n
                │
                ▼
   broadcast CHECKPOINT(seq, state_digest, app_state_hash)
                │
                ▼
   collect ≥2f+1 matching CHECKPOINT
                │
                ▼
   stable checkpoint = seq
   → garbage-collect log entries below seq
   → discard Pre-Prepare/Prepare below seq
```

Full details in [CHECKPOINT.md](./CHECKPOINT.md) (TBD).

---

## 9. Open design questions (architectural level)

These are higher-level than the HANDOVER.md open questions:

| # | Question | Why it matters | Status |
|---|----------|----------------|--------|
| A1 | Should primary's Prepare phase be skipped? (optimisation) | Reduces 1 round-trip in normal case | 🟡 Open |
| A2 | Should we batch multiple submit()s into one Pre-Prepare? | Throughput vs latency trade-off | 🟡 Open |
| A3 | How do we handle client retry on timeout? (PBFT doesn't define this) | Liveness under packet loss | 🟡 Open |
| A4 | Should view-change include app state proof? | Catch-up cost vs safety | 🟡 Open |

---

## 10. References

- [HANDOVER.md](./HANDOVER.md) — decisions summary (transport, Pattern Y, n=7)
- [CRYPTO.md](./CRYPTO.md) — Pattern Y handshake + HMAC details
- [PROTOCOL.md](./PROTOCOL.md) — wire format + transport abstraction
- [CONSENSUS.md](./CONSENSUS.md) — detailed 3-phase spec
- [VIEW-CHANGE.md](./VIEW-CHANGE.md) — primary rotation
- [CHECKPOINT.md](./CHECKPOINT.md) — garbage collection
- Castro & Liskov, "Practical Byzantine Fault Tolerance" (OSDI 1999)

---

**End of ARCHITECTURE.md (v0.1 draft — awaiting review)**