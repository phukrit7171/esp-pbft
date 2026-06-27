# esp-pbft — Checkpoint Protocol

> **Status:** 🟡 Draft (v0.1)
> **Scope:** stable-checkpoint detection, watermark advancement, log GC, state proof, lagging-replica catch-up.
> **Out of scope:** view-change → [VIEW-CHANGE.md](./VIEW-CHANGE.md); steady-state 3-phase → [CONSENSUS.md](./CONSENSUS.md).

This document is the **canonical reference for checkpoint and GC in esp-pbft**. It fills the design gap referenced in CONSENSUS.md §8 and ARCHITECTURE.md §8 (both TBD), and addresses audit issue **A6** (high-watermark never advanced → primary stalls when log fills).

---

## 1. Goals

A checkpoint protocol must satisfy three properties:

| Property | Meaning |
|----------|---------|
| **Agreement** | Two honest replicas that both report a stable checkpoint at the same sequence must report the same state digest. |
| **Liveness** | After a checkpoint is stable on all honest replicas, the PBFT log entries below it can be reclaimed; the high watermark advances; new sequences can be assigned. |
| **Catch-up** | A replica that lags behind (just rebooted, was partitioned, etc.) can fetch the missing state from peers using the stable-checkpoint proof as an anchor. |

The protocol is a stripped-down variant of PBFT §4.3 plus a state-transfer subroutine suitable for the static-memory discipline.

---

## 2. Parameters

| Parameter | Default | Kconfig name | Rationale |
|-----------|---------|---------------|-----------|
| `PBFT_CHECKPOINT_INTERVAL` (`K`) | 100 | `CONFIG_PBFT_CHECKPOINT_INTERVAL` | every K sequences, a replica attempts to prove a stable checkpoint |
| `PBFT_CHECKPOINT_TIMEOUT_MS` | 2000 | `CONFIG_PBFT_CHECKPOINT_TIMEOUT_MS` | how long to wait for checkpoint quorum |
| `PBFT_CHECKPOINT_RETRANSMIT` | 3 | `CONFIG_PBFT_CHECKPOINT_RETRANSMIT` | max rebroadcasts of CHECKPOINT |
| `PBFT_LOG_MAX_ENTRIES` | 100 | `CONFIG_PBFT_LOG_MAX_ENTRIES` | log size — must be ≥ K |
| `PBFT_STATE_DIGEST_BYTES` | 32 | (constant) | SHA-256 size |

**Constraint:** `PBFT_LOG_MAX_ENTRIES ≥ PBFT_CHECKPOINT_INTERVAL`. Default both 100 → log exactly holds one checkpoint interval.

---

## 3. Watermarks (fixes audit A6)

### 3.1 Definitions

```c
typedef struct {
    uint64_t low_watermark;     // entries with seq ≤ low are reclaimable
    uint64_t high_watermark;    // entries with seq > high are forbidden (log full)
    uint64_t stable_seq;        // most recent sequence with a stable checkpoint
    uint8_t  stable_digest[32]; // SHA-256 of app state at stable_seq
} pbft_watermark_state_t;
```

### 3.2 Initialisation

```c
void pbft_watermarks_init(void) {
    wm.low_watermark  = 0;
    wm.stable_seq     = 0;
    wm.stable_digest  = {0};   // all zeros — "no app state yet"
    wm.high_watermark = PBFT_LOG_MAX_ENTRIES;   // first usable seq = high + 1
    view_state.next_sequence = wm.high_watermark + 1;
}
```

So at boot, the first Pre-Prepare is at sequence `PBFT_LOG_MAX_ENTRIES + 1 = 101`. This leaves room for the new node to catch up to whatever the cluster has already committed.

### 3.3 Advancement rule

When a checkpoint at sequence `cp_seq` becomes **stable** (≥ 2f+1 matching CHECKPOINT messages — see §5):

```c
void pbft_watermarks_advance(uint64_t cp_seq, const uint8_t cp_digest[32]) {
    if (cp_seq <= wm.stable_seq) return;   // already past
    if (cp_seq % PBFT_CHECKPOINT_INTERVAL != 0) {
        log_warn("Checkpoint seq %llu not aligned to K=%d", cp_seq, PBFT_CHECKPOINT_INTERVAL);
        return;  // ignore misaligned
    }
    wm.stable_seq    = cp_seq;
    memcpy(wm.stable_digest, cp_digest, 32);
    wm.low_watermark = cp_seq;
    wm.high_watermark = cp_seq + PBFT_LOG_MAX_ENTRIES;
}
```

After advancement, log entries with `seq ≤ cp_seq` may be reclaimed. Log entries with `seq > cp_seq + PBFT_LOG_MAX_ENTRIES` would overflow the log; the primary must halt sequence assignment (return `PBFT_ERR_LOG_FULL`) until the next checkpoint stabilises.

### 3.4 Log-full detection (corrected from CONSENSUS.md §8.4)

```c
bool pbft_log_full(void) {
    return view_state.next_sequence > wm.high_watermark;
}
```

(not `> high_watermark + PBFT_LOG_MAX_ENTRIES` — that overcounts by one interval)

When `pbft_log_full()` is true:
- **Primary** stops accepting new `pbft_submit()` calls (returns `PBFT_ERR_LOG_FULL`).
- **Replicas** continue processing existing in-flight TXs.
- Checkpoint stabilisation (which advances the high watermark) unblocks the primary.

---

## 4. Per-entry state additions

To support checkpoint integration, add to `pbft_log_entry_t` (extends CONSENSUS.md §3.1):

```c
typedef struct {
    // ... existing fields ...
    uint64_t   state_entered_ms;   // NEW — for timeout tracking (audit B6)
    uint8_t    prepare_bitmask;    // CHANGED — single byte, not uint8_t[7] (audit B5)
    uint8_t    commit_bitmask;     // CHANGED — single byte
    uint8_t    exec_bitmask;       // NEW — replicas that have executed this TX
    uint8_t    is_checkpoint;      // NEW — true if seq is a multiple of K
    uint8_t    payload[PBFT_TX_PAYLOAD_MAX];   // CHANGED — inline, not pointer (audit A5)
} pbft_log_entry_t;
```

**Inline payload** raises per-entry size from ~96 B to ~360 B. With 100 entries, the log occupies **36 KB** instead of the previously-claimed 12 KB. Update HANDOVER §3.7, ARCHITECTURE §11, CONSENSUS §13 accordingly (audit A5).

Static bitmasks reduce struct size by 6 B per entry vs the array form, partially offsetting the inline-payload cost.

---

## 5. Checkpoint message flow

### 5.1 Trigger

A replica attempts to prove a checkpoint when **both**:

- It has locally executed all TXs with `seq ≤ cp_seq` (i.e., its `last_executed_seq ≥ cp_seq`), AND
- `cp_seq` is a multiple of `PBFT_CHECKPOINT_INTERVAL`, AND
- It has not already broadcast a CHECKPOINT for `cp_seq` in this interval.

The interval value is local — different replicas may broadcast at slightly different times as their execution progresses.

### 5.2 Message

```
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |  common header (msg_type=6, sender_id, reserved)               |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                                                               |
  |                   sequence (uint64)                           |
  |                                                               |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                                                               |
  |              app_state_digest (32 bytes)                      |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                                                               |
  |                   mac (32 bytes)                              |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

  Total: 4 + 8 + 32 + 32 = 76 bytes
```

The MAC input is `view ‖ sequence ‖ msg_type=6 ‖ app_state_digest` (per CRYPTO.md §5.1).

### 5.3 Quorum

A checkpoint at sequence `s` with digest `d` is **stable** when ≥ 2f+1 = 5 distinct replicas broadcast CHECKPOINT(s, d) with matching digests. A replica tracks incoming CHECKPOINTs in a bitmask:

```c
typedef struct {
    uint64_t   sequence;
    uint8_t    digest[32];
    uint8_t    sender_bitmask;   // bit i = replica i sent a matching CHECKPOINT
    uint8_t    count;
    bool       is_stable;
} pbft_checkpoint_proof_t;
```

The local replica has an implicit "1 vote" (it sent its own CHECKPOINT) → count starts at 1.

### 5.4 Pseudocode: handle CHECKPOINT

```c
void pbft_handle_checkpoint(const pbft_checkpoint_t* cp, uint8_t sender) {
    if (sender >= PBFT_CLUSTER_SIZE) return;
    if (cp->sequence % PBFT_CHECKPOINT_INTERVAL != 0) {
        log_warn("Misaligned checkpoint seq %llu from %d", cp->sequence, sender);
        return;
    }
    if (pbft_verify_mac(sender, view_state.view, cp->sequence,
                         PBFT_MSG_CHECKPOINT, cp->app_state_digest,
                         NULL, 0, cp->mac) != ESP_OK) {
        log_warn("Checkpoint MAC invalid from %d", sender);
        return;
    }

    // Find or create proof entry
    pbft_checkpoint_proof_t* proof = pbft_checkpoint_proof_find(cp->sequence);
    if (!proof) {
        proof = pbft_checkpoint_proof_create(cp->sequence);
        if (!proof) { log_warn("Checkpoint proof table full"); return; }
    }

    // Match digest — disagreement means the cluster is partitioned or Byzantine
    if (proof->count > 0 && memcmp(proof->digest, cp->app_state_digest, 32) != 0) {
        log_error("Conflicting checkpoint digest at seq %llu from %d", cp->sequence, sender);
        // Do not increment count; do not mark stable
        // Possible trigger for view-change (primary equivocation)
        pbft_alarm_raise(PBFT_ALARM_CHECKPOINT_CONFLICT);
        return;
    }

    memcpy(proof->digest, cp->app_state_digest, 32);
    if (!(proof->sender_bitmask & (1u << sender))) {
        proof->sender_bitmask |= (1u << sender);
        proof->count++;
    }

    // Stable?
    if (proof->count >= PBFT_QUORUM && !proof->is_stable) {
        proof->is_stable = true;
        pbft_watermarks_advance(cp->sequence, cp->app_state_digest);
        pbft_gc_up_to(cp->sequence);
        log_info("Stable checkpoint at seq %llu", cp->sequence);
    }
}
```

### 5.5 Pseudocode: send CHECKPOINT

```c
void pbft_maybe_send_checkpoint(void) {
    // Find the next checkpoint boundary we have executed
    uint64_t target = (wm.stable_seq / PBFT_CHECKPOINT_INTERVAL + 1)
                      * PBFT_CHECKPOINT_INTERVAL;

    if (view_state.last_executed_seq < target) return;   // not ready
    if (target == wm.stable_seq) return;                 // already done
    if (pbft_checkpoint_proof_exists(target)) return;    // already broadcast

    // Compute app state digest at this sequence
    // (callback supplied by application; if no callback, digest = SHA-256 of zero string)
    uint8_t digest[32];
    pbft_app_state_digest_at(target, digest);  // application-provided

    pbft_checkpoint_t cp = {
        .sequence = target,
    };
    memcpy(cp.app_state_digest, digest, 32);

    pbft_compute_mac(my_node_id, view_state.view, cp.sequence,
                     PBFT_MSG_CHECKPOINT, cp.app_state_digest,
                     NULL, 0, cp.mac);

    default_transport->broadcast(&cp, sizeof(cp));

    // Local vote
    pbft_handle_checkpoint(&cp, my_node_id);
}
```

This is invoked from the consensus state machine every time a TX transitions to EXECUTED.

---

## 6. State digest

The application supplies the digest (the consensus layer is application-agnostic). The contract:

```c
typedef void (*pbft_state_digest_fn)(uint64_t seq, uint8_t out_digest[32]);

void pbft_register_state_digest_fn(pbft_state_digest_fn fn);
```

If no callback is registered, esp-pbft uses an all-zero digest for every checkpoint — this is correct (all replicas compute the same digest) but useless for change detection.

For the example app (counter), the digest is `SHA-256(counter_value || monotonic_clock)`. The state-transfer protocol uses the digest only as an equality check, not as a content hash (so even a degenerate all-zero digest works as long as all replicas agree).

---

## 7. Garbage collection

### 7.1 Trigger

When `pbft_watermarks_advance(seq, digest)` is called (i.e., a new stable checkpoint), invoke `pbft_gc_up_to(seq)`.

### 7.2 GC algorithm

```c
void pbft_gc_up_to(uint64_t stable_seq) {
    int freed = 0;
    for (int i = 0; i < PBFT_LOG_MAX_ENTRIES; i++) {
        pbft_log_entry_t* e = &pbft_log[i];
        if (e->state == PBFT_STATE_FREE) continue;
        if (e->sequence > stable_seq) continue;
        if (e->state != PBFT_STATE_EXECUTED) {
            log_warn("GC: seq %llu in state %d — should be EXECUTED by now", e->sequence, e->state);
            continue;   // safety: only GC executed entries
        }
        // Zero out payload (defence-in-depth — may contain app secrets)
        mbedtls_platform_zeroize(e->payload, e->payload_len);
        memset(e, 0, sizeof(*e));
        e->state = PBFT_STATE_FREE;
        freed++;
    }
    log_info("GC reclaimed %d entries below seq %llu", freed, stable_seq);
}
```

**Why the EXECUTED-state gate:** a TX in PREPARED but not yet COMMITTED must remain in the log so the new primary can include it in the O-set during view-change. GC of an uncommitted TX would violate PBFT safety.

### 7.3 Per-entry `is_checkpoint` flag

When GC runs, mark every freed entry's slot as available. Future Pre-Prepares with `seq ≤ stable_seq` are rejected at receive time (consensus handler checks `pp->sequence > wm.low_watermark`).

---

## 8. Lagging-replica catch-up (state transfer)

### 8.1 When needed

A replica is "lagging" when:
- Its `last_executed_seq < wm.stable_seq`, OR
- It receives a Pre-Prepare or New-View with `seq > last_executed_seq + 1` (gap).

### 8.2 Protocol

The lagging replica requests state from peers using a `STATE-REQUEST` / `STATE-RESPONSE` pair. Both messages are NEW message types (IDs 7 and 8 in the enum).

```
   STATE-REQUEST (msg_type=7)
   ┌──────────────────────────────────────────────┐
   │ common header (type=7, sender_id, reserved) │
   │ requested_seq (uint64)                       │
   │ known_digest (32 bytes) — digest I have      │
   │ mac (32 bytes)                               │
   └──────────────────────────────────────────────┘
   Size: 4 + 8 + 32 + 32 = 76 bytes

   STATE-RESPONSE (msg_type=8)
   ┌──────────────────────────────────────────────┐
   │ common header (type=8, sender_id, reserved) │
   │ requested_seq (uint64)                       │
   │ num_entries (uint8)                          │
   │ { Pre-Prepare × num_entries }                │
   │ mac (32 bytes)                               │
   └──────────────────────────────────────────────┘
   Size: 4 + 8 + 1 + num × (82 + payload_len) + 32
        ≈ 45 + num × 338 bytes max
```

For n=7, num_entries can be 1-100 (capped at log size). Single STATE-RESPONSE can be up to 33.8 KB → UDP only.

### 8.3 Catch-up flow

```
   Lagging replica L                         Peer replica P
        │                                          │
        │  (notices seq gap or post-boot)           │
        │                                          │
        │──── STATE-REQUEST(s, d) ────────────────▶│
        │                                          │
        │                                          │ verify MAC
        │                                          │ find entries with seq ≥ s
        │                                          │ build Pre-Prepare messages
        │                                          │ (each with its own MAC)
        │                                          │
        │◀──── STATE-RESPONSE(s, n, {Pre-Prepare}) │
        │                                          │
        │  verify MAC                              │
        │  for each Pre-Prepare:                   │
        │    run pbft_handle_pre_prepare           │
        │    → catch up via normal 3-phase         │
```

L broadcasts STATE-REQUEST to all peers (or to a random subset if load is a concern). The first peer to respond "wins" (others' responses are discarded after L verifies digests).

**Self-healing property:** once L receives Pre-Prepares via STATE-RESPONSE, it processes them through the normal `pbft_handle_pre_prepare` handler — no special-case catch-up code path is needed. The normal Prepare/Commit quorum completes the catch-up.

### 8.4 State application (app layer)

For TXs that have already been EXECUTED on peers (i.e., `seq ≤ wm.stable_seq`), the L replica must also re-apply them to its app state. This requires:

1. App-supplied state snapshot fetch — out of scope for esp-pbft core (the app must provide a callback to fetch its current state).
2. Verification via stable-checkpoint digest.

For v1, the lagging replica must either:
- (a) Restart from app's initial state and re-execute TXs from `app_initial_seq + 1` to `stable_seq`. Works if the app is replay-deterministic.
- (b) Fetch state snapshot from a peer (requires additional app-supplied callback).

The esp-pbft core delivers (a) automatically (Pre-Prepares from the catch-up response feed the normal 3-phase → commit callback fires → app applies TX). Option (b) is for apps that can't replay.

### 8.5 Catch-up timeout and retry

If no STATE-RESPONSE arrives within `PBFT_CHECKPOINT_TIMEOUT_MS`, retransmit up to `PBFT_CHECKPOINT_RETRANSMIT = 3` times. After that, declare partition and trigger view-change.

---

## 9. Interaction with view-change

### 9.1 Checkpoint during view-change

If a replica is in `PBFT_VIEW_PHASE_CHANGING` or `PBFT_VIEW_PHASE_WAIT_NEW`:
- It still accepts CHECKPOINT messages.
- Its own CHECKPOINT broadcast is suppressed (waiting for new view is safer).
- When New-View arrives and view resumes, catch up via the O-set first, then resume normal checkpoint behaviour.

### 9.2 O-set construction and checkpoints

The new primary's O-set (§6.1 of VIEW-CHANGE.md) starts at `low_watermark + 1` (where `low_watermark = max(checkpoint_seq across VCs)`). This means O-set never re-proposes checkpoint-anchored TXs — they remain in the stable state.

### 9.3 New-View and high_watermark

After successful view-change, the new primary's `high_watermark` is recomputed from the current `low_watermark`. No special handling needed.

---

## 10. Memory cost

| Item | Static size |
|------|-------------|
| `pbft_checkpoint_proof_t` table (8 in-flight proofs) | 8 × ~50 B = 400 B |
| Updated `pbft_log_entry_t` | ~360 B per entry |
| `PBFT_LOG_MAX_ENTRIES = 100` | 36 KB total (vs originally-claimed 12 KB) |
| STATE-REQUEST/RESPONSE scratch | 4 KB UDP RX buffer (already counted) |
| **Total checkpoint-related** | **~37 KB** |

The PBFT log is now the single largest static allocation in esp-pbft. Static_assert needed:

```c
_Static_assert(sizeof(pbft_log_entry_t) * PBFT_LOG_MAX_ENTRIES <= PBFT_LOG_MAX_BYTES,
               "PBFT log exceeded budget");
```

where `PBFT_LOG_MAX_BYTES = 40 * 1024` (40 KB cap).

---

## 11. Static_assert catalog (checkpoint-related)

Add to ARCHITECTURE.md §11.3:

```c
// MEMORY.md maintains the full catalog. Checkpoint-critical assertions:

_Static_assert(PBFT_LOG_MAX_ENTRIES >= PBFT_CHECKPOINT_INTERVAL,
               "LOG_MAX must be >= CHECKPOINT_INTERVAL");

_Static_assert(PBFT_CHECKPOINT_INTERVAL > 0,
               "CHECKPOINT_INTERVAL must be positive");

_Static_assert(PBFT_QUORUM == 5,
               "PBFT_QUORUM must be 5 for n=7, f=2");

_Static_assert(sizeof(pbft_log_entry_t) * PBFT_LOG_MAX_ENTRIES <= 40 * 1024,
               "PBFT log exceeded 40 KB budget");
```

---

## 12. Configuration (Kconfig additions)

```kconfig
CONFIG_PBFT_CHECKPOINT_INTERVAL=100
CONFIG_PBFT_CHECKPOINT_TIMEOUT_MS=2000
CONFIG_PBFT_CHECKPOINT_RETRANSMIT=3
CONFIG_PBFT_LOG_MAX_ENTRIES=100
CONFIG_PBFT_LOG_MAX_BYTES=40960
CONFIG_PBFT_TX_PAYLOAD_MAX=256
# CONFIG_PBFT_STATE_TRANSFER enables STATE-REQUEST/RESPONSE catch-up
CONFIG_PBFT_STATE_TRANSFER=y
```

---

## 13. Open questions (checkpoint level)

| # | Question | Recommendation |
|---|----------|----------------|
| CP1 | What if a replica's app state digest differs from the stable digest because it has a stale view of an app-supplied state? | Trust the digest. If 2f+1 agree on digest d, then ≥ f+1 honest replicas have app state d; any replica with a different digest must discard its divergent state and re-apply via catch-up. |
| CP2 | How often to attempt checkpoint broadcast? | Only when `last_executed_seq` crosses a K-boundary. Natural rate-limiting. |
| CP3 | Should the app state digest be signed by the app (e.g., HMAC keyed with app secret)? | Optional v2. The PBFT HMAC on the CHECKPOINT message is sufficient for v1 (prevents external forgery). |
| CP4 | What if state transfer is needed but no peer responds (full partition)? | Declare partition mode (see VIEW-CHANGE §9.3); resume on healing. |

---

## 14. References

- [HANDOVER.md](./HANDOVER.md) — memory budget (now updated)
- [ARCHITECTURE.md](./ARCHITECTURE.md) §8 — preview (this doc replaces)
- [CONSENSUS.md](./CONSENSUS.md) — log state machine, state-machine for entries
- [VIEW-CHANGE.md](./VIEW-CHANGE.md) — view-change interactions (§9 above)
- [CRYPTO.md](./CRYPTO.md) §5 — MAC computation for CHECKPOINT
- [PROTOCOL.md](./PROTOCOL.md) — wire format (§6.7)
- [DESIGN-AUDIT.md](./DESIGN-AUDIT.md) — A5 (inline payload), A6 (high-watermark advance), B5 (bitmask), B6 (state_entered_ms)
- Castro & Liskov, "Practical Byzantine Fault Tolerance" (OSDI 1999), §4.3 — checkpoint protocol

---

**End of CHECKPOINT.md (v0.1 draft — addresses audit issues A5, A6, B5, B6)**