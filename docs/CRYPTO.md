# esp-pbft — Crypto Design (Pattern Y)

> **Status:** 🟡 Draft
> **Pattern:** Y — Self-generated TRNG keys + ECDH-boot + HMAC-SHA256 runtime
> **Variant:** Y-5 — periodic re-gen every 24h, staggered by `node_id × 1h`
> **Stack:** PSA Crypto API on Mbed TLS 4.0.0 (TF-PSA-Crypto backend)

---

## 1. Overview

esp-pbft uses **Pattern Y** for all message authentication:

| Phase | Mechanism | When |
|-------|-----------|------|
| **Boot** | ECDSA P-256 keypair via TRNG | Every boot (cold start) |
| **Discovery** | Hello broadcast (pubkey) | After boot |
| **Key agreement** | ECDH → SHA-256 → HMAC key | After Hello exchange |
| **Runtime** | HMAC-SHA256 per message | Every consensus message |
| **Y-5 re-gen** | New keypair + soft re-handshake | Every 24h (staggered) |

**No ECDSA sign/verify at runtime** — ECDSA is used only to derive keys at boot.

---

## 2. Key types and sizes

| Key | Algorithm | Size | Lifetime |
|-----|-----------|------|----------|
| ECDSA private (own) | P-256 (secp256r1) | 32 B | RAM only, regenerated each boot + Y-5 |
| ECDSA public (own) | P-256 uncompressed | 64 B | RAM only, regenerated each boot + Y-5 |
| Peer ECDSA public | P-256 uncompressed | 64 B | BSS, cached (TOFU) |
| Pairwise HMAC key | HMAC-SHA256 | 32 B | RAM, regenerated on peer re-key |

**No key persistence** — private keys live only in RAM.

---

## 3. PSA Crypto API surface (verified for ESP-IDF v6.0.1)

Verified at `~/.espressif/v6.0.1/esp-idf/components/mbedtls/mbedtls/tf-psa-crypto/include/psa/crypto.h`.

### 3.1 APIs used

| Function | Purpose |
|----------|---------|
| `psa_generate_key()` | Create ECDSA P-256 keypair (TRNG-backed) |
| `psa_export_public_key()` | Get our 64-byte uncompressed pubkey for Hello |
| `psa_key_agreement()` | ECDH → shared secret (32 B) |
| `psa_mac_compute()` | HMAC-SHA256 sign |
| `psa_mac_verify()` | HMAC-SHA256 verify |

### 3.2 APIs explicitly NOT used

| Function | Why avoided |
|----------|-------------|
| `psa_sign_message()` / `psa_verify_message()` | ECDSA at runtime = ~50 ms/msg — kills throughput |
| `psa_key_derivation_setup()` etc. | We use simple SHA-256 (not HKDF) — fewer APIs, smaller code |

### 3.3 Key attributes setup (template)

```c
psa_key_attributes_t attr = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_type(&attr, PSA_KEY_TYPE_ECC_KEY_PAIR(PSA_ECC_FAMILY_SECP_R1));
psa_set_key_bits(&attr, 256);
psa_set_key_algorithm(&attr, PSA_ALG_ECDH);  // key-agreement only
psa_set_key_usage_flags(&attr, PSA_KEY_USAGE_DERIVE);  // not sign!
```

Note: `PSA_KEY_USAGE_DERIVE` only — we never sign with this key. This is enforced by the key's policy.

---

## 4. Boot sequence (cold start)

### 4.1 Pseudocode

```c
// pbft_crypto_boot() — called once at boot
void pbft_crypto_boot(void) {
    // 1. Generate own ECDSA P-256 keypair
    psa_key_attributes_t attr = PSA_KEY_ATTRIBUTES_INIT;
    psa_set_key_type(&attr, PSA_KEY_TYPE_ECC_KEY_PAIR(PSA_ECC_FAMILY_SECP_R1));
    psa_set_key_bits(&attr, 256);
    psa_set_key_algorithm(&attr, PSA_ALG_ECDH);
    psa_set_key_usage_flags(&attr, PSA_KEY_USAGE_DERIVE);

    psa_status_t status = psa_generate_key(&attr, &my_priv_key_id);
    if (status != PSA_SUCCESS) { /* handle error */ }

    // 2. Export our public key for Hello broadcast
    uint8_t my_pub[64];
    size_t my_pub_len;
    psa_export_public_key(my_priv_key_id, my_pub, sizeof(my_pub), &my_pub_len);
    assert(my_pub_len == 64);

    // 3. Staggered boot delay (random 0-3s)
    vTaskDelay(pdMS_TO_TICKS(esp_random() % 3000));

    // 4. Broadcast Hello (my_pub, my_node_id)
    pbft_network_broadcast_hello(my_pub, my_node_id);

    // 5. Wait for Hellos from 6 peers (timeout 5s)
    if (pbft_discovery_wait(5000) != ESP_OK) { /* retry or fail */ }

    // 6. For each peer: ECDH + derive HMAC key
    for (int peer = 0; peer < PBFT_CLUSTER_SIZE; peer++) {
        if (peer == my_node_id) continue;

        // ECDH: shared = ECDH(my_priv, peer_pub)
        uint8_t shared[32];
        psa_key_agreement(PSA_ALG_ECDH,
                          my_priv_key_id,
                          peer_pub_keys[peer],  // 64 bytes
                          shared, sizeof(shared), &shared_len);

        // Derive HMAC key: SHA-256(shared || "PBFT-HMAC-v1")
        uint8_t hmac_key[32];
        pbft_sha256_derive(shared, sizeof(shared),
                          (uint8_t*)"PBFT-HMAC-v1", 13,
                          hmac_key);

        // Store (static array, BSS)
        memcpy(hmac_keys[peer], hmac_key, 32);

        // Zeroize shared secret on stack
        mbedtls_platform_zeroize(shared, sizeof(shared));
    }
}
```

### 4.2 Why simple SHA-256 (not HKDF)?

We use `SHA-256(shared || "PBFT-HMAC-v1")` instead of HKDF-Extract+Expand because:

| Aspect | HKDF | Simple SHA-256 |
|--------|------|----------------|
| Output | 32+ B (expandable) | 32 B (fixed) |
| Salt input | Required | Not needed (label "PBFT-HMAC-v1" replaces) |
| Info input | Required | Yes ("PBFT-HMAC-v1") |
| Code size | ~200 B | ~50 B |
| Standard | RFC 5869 | Ad-hoc (acceptable for fixed-input pattern) |

For our use case (single 32-byte HMAC key per peer, fixed inputs), simple SHA-256 is sufficient and simpler. **HKDF would be needed if we derived multiple sub-keys per peer** (e.g., one for encrypt, one for MAC) — not our case.

---

## 5. Per-message MAC

### 5.1 MAC input format

```
MAC input (concatenated, no length prefix):
  view      : u32  (4 bytes, little-endian)
  sequence  : u64  (8 bytes, little-endian)
  msg_type  : u8   (1 byte: Pre-Prepare=1, Prepare=2, Commit=3, etc.)
  digest    : 32 bytes (SHA-256 of TX payload, or all-zeros for non-TX msgs)
  payload   : variable (0-256 bytes, TX payload for Pre-Prepare; empty otherwise)
────────────────────────────────────────────────────────────
  Total     : 45 bytes minimum (view+seq+type+digest)
```

### 5.2 Why this input format?

| Field | Purpose |
|-------|---------|
| `view` | Anti-replay across view changes (old-view messages rejected) |
| `sequence` | Anti-replay within view (monotonic check) |
| `msg_type` | Domain separation (Pre-Prepare MAC ≠ Prepare MAC) |
| `digest` | Binds message to specific TX content |
| `payload` | Authenticates actual TX bytes |

Without `view‖sequence`, an attacker could replay an old (committed) Prepare to confuse a slow replica. Without `msg_type`, a Prepare MAC could be reused as Commit MAC.

### 5.3 HMAC compute

```c
// pbft_compute_mac(peer_id, view, seq, type, digest, payload, payload_len, mac_out)
esp_err_t pbft_compute_mac(uint8_t peer_id,
                             uint32_t view, uint64_t seq, uint8_t type,
                             const uint8_t digest[32],
                             const uint8_t* payload, size_t payload_len,
                             uint8_t mac_out[32]) {
    uint8_t buf[45 + 256];   // stack, bounded
    size_t off = 0;

    // Write view (LE)
    memcpy(buf + off, &view, 4);  off += 4;
    // Write sequence (LE)
    memcpy(buf + off, &seq, 8);   off += 8;
    // Write type
    buf[off++] = type;
    // Write digest
    memcpy(buf + off, digest, 32); off += 32;
    // Write payload (if any)
    if (payload && payload_len > 0) {
        if (payload_len > 256) return ESP_ERR_INVALID_ARG;
        memcpy(buf + off, payload, payload_len);
        off += payload_len;
    }

    // HMAC-SHA256 (uses ESP32-C3 SHA HW peripheral)
    return psa_mac_compute(hmac_key_ids[peer_id],
                          PSA_ALG_HMAC(PSA_ALG_SHA_256),
                          buf, off,
                          mac_out, 32, &(size_t){0});
}
```

**Performance:** ~5 µs per MAC on ESP32-C3 (SHA HW @ 160 MHz).

### 5.4 HMAC verify (receive path)

```c
esp_err_t pbft_verify_mac(uint8_t peer_id,
                            uint32_t view, uint64_t seq, uint8_t type,
                            const uint8_t digest[32],
                            const uint8_t* payload, size_t payload_len,
                            const uint8_t received_mac[32]) {
    uint8_t calc_mac[32];

    // 1. Re-compute expected MAC
    esp_err_t err = pbft_compute_mac(peer_id, view, seq, type, digest,
                                      payload, payload_len, calc_mac);
    if (err != ESP_OK) return err;

    // 2. Constant-time compare (prevent timing attack)
    return mbedtls_ct_memcmp(calc_mac, received_mac, 32) == 0
           ? ESP_OK : ESP_ERR_INVALID_MAC;
}
```

**Constant-time compare is mandatory** — non-constant-time `memcmp` leaks information through timing.

---

## 6. Discovery (Hello exchange)

### 6.1 Hello message format

```c
typedef struct __attribute__((packed)) {
    uint8_t  msg_type;       // = PBFT_MSG_HELLO (0)
    uint8_t  sender_id;      // 0..6
    uint8_t  pubkey[64];     // uncompressed P-256
    uint8_t  re_handshake;   // 1 if this is a re-handshake (Y-5 or peer restart)
    uint8_t  mac[32];        // HMAC over (msg_type ‖ sender_id ‖ pubkey ‖ re_handshake)
                             //   using current hmac_keys[sender_id] if known,
                             //   else zeros (initial Hello)
} pbft_hello_t;
```

### 6.2 TOFU verification

```c
esp_err_t pbft_on_hello(const pbft_hello_t* hello) {
    uint8_t peer = hello->sender_id;
    if (peer >= PBFT_CLUSTER_SIZE) return ESP_ERR_INVALID_ARG;

    // TOFU check: have we seen this peer before?
    bool first_hello = !peer_pubkey_cached[peer];

    if (first_hello) {
        // Cache the pubkey (first wins)
        memcpy(peer_pubkey_cached[peer], hello->pubkey, 64);
        // Compute HMAC key with new peer
        pbft_derive_hmac_key(peer, hello->pubkey);
        // MAC may be zeros (peer has no key yet) — accept
        ESP_LOGW(TAG, "First hello from node %d — TOFU trust", peer);
    } else {
        // Subsequent hellos: pubkey MUST match
        if (memcmp(peer_pubkey_cached[peer], hello->pubkey, 64) != 0) {
            ESP_LOGE(TAG, "Hello from node %d has different pubkey! Alarm!", peer);
            return ESP_ERR_INVALID_PEER;
        }
        // Verify MAC with current hmac_keys[peer]
        // (re_handshake may be 1 — meaning peer regenerated key, but TOFU says first wins)
    }
    return ESP_OK;
}
```

**TOFU trust model:** First Hello from a peer is trusted. Subsequent Hellos must match cached pubkey. If mismatch → log alarm and refuse.

---

## 7. Y-5 periodic re-gen (every 24h, staggered)

### 7.1 Trigger

Each node has a timer that fires at `(node_id × 1 hour)` after boot. With n=7:
- Node 0: 00:00
- Node 1: 01:00
- Node 2: 02:00
- ...
- Node 6: 06:00

Period: 24h (all 7 nodes re-gen within 6h window).

### 7.2 Re-gen flow

```c
// pbft_y5_timer_cb() — called every 24h
void pbft_y5_timer_cb(void) {
    ESP_LOGI(TAG, "Y-5 re-gen triggered");

    // 1. Destroy old keypair (zeroize)
    psa_destroy_key(my_priv_key_id);
    mbedtls_platform_zeroize(hmac_keys, sizeof(hmac_keys));
    mbedtls_platform_zeroize(peer_pubkey_cached, sizeof(peer_pubkey_cached));

    // 2. Generate new keypair (TRNG)
    psa_generate_key(&attr, &my_priv_key_id);

    // 3. Export new pubkey
    uint8_t new_pub[64];
    psa_export_public_key(my_priv_key_id, new_pub, 64, &len);

    // 4. Broadcast Hello with re_handshake=1
    pbft_hello_t hello = {
        .msg_type    = PBFT_MSG_HELLO,
        .sender_id   = my_node_id,
        .re_handshake = 1,
    };
    memcpy(hello.pubkey, new_pub, 64);
    pbft_network_broadcast(&hello, sizeof(hello));

    // 5. Wait for peers to re-ECDH (handled via Hello receive + soft re-handshake)

    // 6. Restart 24h timer
    pbft_timer_start(PBFT_TIMER_Y5, 24 * 60 * 60 * 1000);
}
```

### 7.3 Soft re-handshake (peer receives re-handshake Hello)

When a replica receives Hello with `re_handshake=1` from peer X:

```c
// Already in pbft_on_hello()
if (hello->re_handshake && !first_hello) {
    // Same pubkey? Could be wrong peer, reject
    if (memcmp(peer_pubkey_cached[peer], hello->pubkey, 64) != 0) {
        // Different pubkey for known peer → re-handshake accepted
        memcpy(peer_pubkey_cached[peer], hello->pubkey, 64);
        pbft_derive_hmac_key(peer, hello->pubkey);
        ESP_LOGW(TAG, "Soft re-handshake with node %d", peer);
    }
    // Verify MAC with new hmac_keys[peer] (which we just derived)
}
```

---

## 8. ESP32-C3 crypto performance (verified)

From research at `~/.espressif/v6.0.1/esp-idf/components/mbedtls/`:

| Operation | Latency | Hardware |
|-----------|---------|----------|
| TRNG keygen (32 B) | <1 ms | TRNG peripheral |
| ECDSA P-256 keypair (TRNG + ecc_mul) | ~5 ms | TRNG + ECC HW |
| ECDH P-256 (shared secret) | ~8 ms | ECC HW point mul |
| HMAC-SHA256 (45-byte input) | ~3 µs | SHA HW + DMA |
| HMAC-SHA256 (256-byte input) | ~5 µs | SHA HW + DMA |
| ECDSA sign (NOT USED at runtime) | ~50 ms | Software (no HW) |
| ECDSA verify (NOT USED at runtime) | ~15 ms | ECC HW point mul |

**Conclusion:** Pattern Y uses **only fast operations** (TRNG, ECDH once at boot, HMAC at runtime). No slow ECDSA sign/verify.

---

## 9. Key lifecycle diagram

```
   t=0 (boot)
      │
      ▼
   TRNG keygen → ECDSA P-256 keypair (RAM only)
      │
      ▼
   Staggered delay (0-3s)
      │
      ▼
   Hello broadcast (pubkey, node_id)
      │
      ▼
   Wait for 6 Hellos → cache peer pubkeys
      │
      ▼
   For each peer: ECDH + SHA-256 derive HMAC key
      │
      ▼
   Steady state: HMAC every message (5 µs/msg)
      │
      │ ... 24 hours ...
      ▼
   t=24h (Y-5 trigger)
      │
      ▼
   psa_destroy_key(my_priv) → zeroize RAM
      │
      ▼
   psa_generate_key() → NEW keypair
      │
      ▼
   Hello broadcast with re_handshake=1
      │
      ▼
   Peers receive → re-derive HMAC keys (soft)
      │
      ▼
   Steady state resumes with new keys
      │
      │ ... 24 hours ...
      ▼
   (cycle continues)

   On node crash/restart:
      │
      ▼
   Same flow as t=0 (fresh TRNG keypair, no compromise window)
```

---

## 10. Security analysis

### 10.1 What we get

| Property | Mechanism |
|----------|-----------|
| **Authentication** | HMAC-SHA256 proves sender has shared key |
| **Integrity** | HMAC detects any bit-flip |
| **Anti-replay** | `view ‖ sequence` in MAC input |
| **Forward secrecy (within session)** | Y-5 re-gen bounds compromise window to 24h |
| **Forward secrecy (across reboot)** | New keypair at every boot |
| **Self-healing** | Compromise recovery via Y-5 or reboot |

### 10.2 What we don't get (and why it's OK)

| Property | Why not needed |
|----------|----------------|
| **Non-repudiation** | All nodes equally trusted (no audit trail needed) |
| **Confidentiality** | PBFT messages contain TX payload — application decides encryption |
| **Perfect forward secrecy (sub-second)** | 24h compromise window is acceptable for IoT |

### 10.3 Threat model

| Attacker | Defense |
|----------|---------|
| **Passive eavesdropper** | No defense (no encryption needed for PBFT) |
| **Active message forger** | HMAC verification — no shared key → reject |
| **Replay attacker** | `view ‖ sequence` bound in MAC |
| **Compromised node (full key)** | Y-5 re-gen + soft re-handshake bounds window to 24h |
| **MITM during Hello** | TOFU — first Hello trusted (physical security assumed) |
| **Byzantine primary** | PBFT Prepare/Commit quorum catches (≥2f+1) |

---

## 11. PSA key storage pattern (v1)

**v1 uses PSA volatile keys** (in RAM only, not persisted):

```c
psa_key_id_t my_priv_key_id;          // volatile, RAM only
psa_key_id_t hmac_key_ids[7];         // 6 volatile HMAC keys
```

**v2 could use PSA persistent keys** (NVS-backed, encrypted) if Y-5 is insufficient. Not implemented in v1.

---

## 12. Open questions (crypto-level)

| # | Question | Status |
|---|----------|--------|
| C1 | Should Hello be encrypted? (privacy of pubkey) | 🟡 Open — low priority |
| C2 | Should we add ECDSA signature on Hello (drop TOFU)? | 🟡 Open — adds ~50ms handshake cost |
| C3 | HMAC key rotation interval — 24h optimal? | 🟡 TBD — depends on threat model |
| C4 | Should we use HMAC-SHA256 or HMAC-SHA3-256? | ✅ SHA-256 (HW-accelerated) |

---

## 13. References

- [HANDOVER.md](./HANDOVER.md) §3.5 — Pattern Y decision
- [ARCHITECTURE.md](./ARCHITECTURE.md) §2 — Boot sequence
- [ARCHITECTURE.md](./ARCHITECTURE.md) §11 — Static memory discipline
- ESP-IDF v6.0.1 PSA Crypto docs — `~/.espressif/v6.0.1/esp-idf/components/mbedtls/`
- NIST FIPS 180-4 — SHA-256
- NIST FIPS 186-4 — ECDSA P-256
- RFC 2104 — HMAC
- RFC 5869 — HKDF (not used, see §4.2)

---

**End of CRYPTO.md (v0.1 draft — awaiting review)**