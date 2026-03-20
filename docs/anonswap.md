# AnonSwap Block Exchange Protocol

---

## Overview

AnonSwap is I2PFS's block exchange protocol. It is built on three core principles:

1. **Fixed binary frames only.** Every field is at a fixed offset with a known size, enabling zero-copy deserialization and eliminating parser attack surface. No CBOR, no JSON, no variable-length schema-implicit encoding.
2. **No persistent peer state.** AnonSwap maintains no per-peer history beyond the current session's Noise handshake keys and ratchet state. No credit scores, no want-list histories, no connection scores.
3. **Sessions are ephemeral by design.** A session is created for a specific content retrieval and torn down when complete. Session IDs are never reused.

---

## Session Establishment — Noise_IKpsk2

AnonSwap uses **Noise_IKpsk2** for session establishment. This pattern was chosen for three reasons:

1. **1 RTT establishment:** The initiator sends its ephemeral key and encrypted payload in message 1. The responder replies in message 2. Application data flows immediately after.
2. **Responder static key privacy:** The responder's static key is never revealed to passive observers. An eavesdropper cannot determine which node is acting as provider.
3. **PSK capability check:** The pre-shared key slot allows use of a content-derived key as an additional authentication factor — only a consumer who correctly performed a DHT lookup for the specific content can complete the handshake.

### Handshake Pattern

```
Prologue: "I2PFS-AnonSwap-v1"   (both parties must match)

  Initiator (Consumer)             Responder (Provider)
  ─────────────────────────────────────────────────────
  → e, es, s, ss, psk              ← message 1 (encrypted)
                                    e, ee, se ← message 2
  ─────────────────────────────────────────────────────
  [Both parties derive: send_key, recv_key, nonce_counters]
```

### PSK Derivation

```python
psk := HKDF-SHA256(
    ikm  = dht_key_for_content,    # 32-byte epoch-derived DHT key (public)
    salt = "I2PFS-session-psk",
    info = session_id,             # 16-byte ephemeral random session ID
    len  = 32
)
```

The PSK derived from the content's DHT key means that only a consumer who correctly performed the DHT lookup can complete the handshake. This is a lightweight capability check requiring no per-content access list on the provider.

### Handshake Message Sizes

| Message | Direction | Size |
|---------|-----------|------|
| Message 1 | Consumer → Provider | ~128 bytes |
| Message 2 | Provider → Consumer | ~64 bytes |

---

## Frame Format

All AnonSwap messages use a fixed binary frame:

```
AnonSwap Frame (binary, little-endian)
├── frame_type  : 1 byte   (message type ID; see table below)
├── session_id  : 16 bytes (ephemeral, per-retrieval; random; never reused)
├── sequence    : 8 bytes  uint64, per-session counter; monotonically increasing
├── payload_len : 4 bytes  uint32
└── payload     : payload_len bytes (type-specific)

Total overhead: 29 bytes per frame
```

### Frame Types

| `frame_type` | Name | Direction | Payload |
|---|---|---|---|
| `0x01` | `WANT_BLOCK` | Consumer → Provider | `aci[38] ‖ max_size[4]` = 42 bytes |
| `0x02` | `HAVE_BLOCK` | Provider → Consumer | `aci[38] ‖ size[4] ‖ chunk_index[4]` = 46 bytes |
| `0x03` | `SEND_BLOCK` | Provider → Consumer | `aci[38] ‖ chunk_index[4] ‖ block_data[var]` |
| `0x04` | `DONT_HAVE` | Provider → Consumer | `aci[38]` = 38 bytes |
| `0x05` | `ACK` | Consumer → Provider | `aci[38] ‖ chunk_index[4]` = 42 bytes |
| `0x06` | `RESUME_REQ` | Consumer → Provider | `aci[38] ‖ bitmap_len[4] ‖ bitmap[var]` |
| `0x07` | `RESUME_ACK` | Provider → Consumer | `aci[38] ‖ avail_bitmap_len[4] ‖ avail_bitmap[var]` |
| `0x08` | `RATCHET` | Either → Either | `new_ratchet_key[32]` |
| `0x09` | `SESSION_END` | Either → Either | `reason[1]` (see [error codes](types-constants.md#error-codes)) |

---

## Retrieval Session Lifecycle

```
Consumer                               Provider
   │                                       │
   │   [Derive PSK from content DHT_key]   │
   │                                       │
   │──── Noise_IKpsk2 Message 1 ──────────►│
   │                                       │  [Verify PSK; establish session]
   │◄─── Noise_IKpsk2 Message 2 ───────────│
   │                                       │
   │  [Optional: resume if checkpoint]     │
   │──── RESUME_REQ(aci, bitmap) ─────────►│
   │◄─── RESUME_ACK(aci, avail_bitmap) ────│
   │                                       │
   │──── WANT_BLOCK(aci, chunk_i) ────────►│  ┐
   │──── WANT_BLOCK(aci, chunk_j) ────────►│  │ pipelined,
   │──── WANT_BLOCK(aci, chunk_k) ────────►│  │ window ≤ 6
   │                                       │  ┘
   │◄─── SEND_BLOCK(aci, chunk_i, data) ───│  (optimistic: no HAVE_BLOCK round trip)
   │◄─── SEND_BLOCK(aci, chunk_j, data) ───│
   │◄─── SEND_BLOCK(aci, chunk_k, data) ───│
   │                                       │
   │──── ACK(aci, chunk_i) ───────────────►│  (window slides)
   │──── ACK(aci, chunk_j) ───────────────►│
   │                                       │
   │  [Verify: BLAKE3(data‖tag) == ACI]    │
   │  [Decrypt chunk; update bitmap]       │
   │                                       │
   │──── SESSION_END(0x00) ───────────────►│
```

### Optimistic Sending

Providers send `SEND_BLOCK` **immediately** after receiving `WANT_BLOCK` without waiting for any confirmation. This eliminates the `HAVE_BLOCK → WANT_BLOCK[confirm]` round trip, saving 500–800 ms per block at typical I2P RTTs. The consumer's `ACK` serves only as a flow control signal (sliding window), not as a delivery confirmation.

Providers that cannot serve a block MUST send `DONT_HAVE` within **5 seconds** of receiving `WANT_BLOCK`. Silence is not an acceptable response.

---

## Session Anonymity

**Ephemeral session IDs:** Each session uses a 16-byte random `session_id` generated fresh. Session IDs are never reused. A provider receiving two sessions with different `session_id` values cannot determine whether they originate from the same consumer.

**Per-retrieval Client SAM session:** The consumer's sessions originate from a Client SAM Session that is rotated for each top-level content retrieval (one root ACI = one SAM session rotation). The I2P Destination seen by the provider changes with each new retrieval.

**No persistent want-list:** There is no concept of a broadcast want-list. `WANT_BLOCK` messages are sent unicast only to providers discovered via DHT lookup.

---

## Symmetric Ratchet

After every **1,000 AnonSwap messages** within a session (tracked via the `sequence` counter), both parties advance the symmetric ratchet:

```python
# Party whose sequence hits the 1,000-message mark:
ratchet_key_new = CSPRNG(32 bytes)
send RATCHET frame with payload = ratchet_key_new
send_key_new = BLAKE3(current_send_key || ratchet_key_new || "I2PFS-ratchet-send")[:32]
send_key = send_key_new
sequence = 0

# Receiving party, on RATCHET frame:
recv_key_new = BLAKE3(current_recv_key || ratchet_key_new || "I2PFS-ratchet-recv")[:32]
recv_key = recv_key_new
```

**Post-compromise security guarantee:** If an attacker obtains current session keys and then loses access, the ratchet step — incorporating fresh random material from `ratchet_key_new` — ensures the attacker cannot decrypt future messages. The maximum exposure window for any compromise is bounded to ≤ 1,000 messages (~32 MB at 32 KB per message).

---

## Flow Control

Per-session window-based flow control prevents a fast publisher from overwhelming a slow provider's tunnel:

```python
SEND_WINDOW = clamp(
    value = floor(RTT_p50_ms / block_transfer_time_ms) + 2,
    min   = 2,
    max   = 6
)

# block_transfer_time_ms = (32768 * 8) / (effective_bandwidth_kbps * 1000)

in_flight = 0
# On SEND_BLOCK sent:      in_flight++
# On ACK received:         in_flight--; send next pending block if in_flight < SEND_WINDOW
# On timeout (30 s):       in_flight--; halve SEND_WINDOW (min 1); retransmit
# On DONT_HAVE:            in_flight--; route WANT_BLOCK to next provider
```

!!! note "Why max window is 6, not 8"
    A window of 8 at typical I2P bandwidth puts 256 KB in flight simultaneously (~8–32 seconds of tunnel capacity). This risks tunnel saturation on the provider side and produces self-inflicted congestion. A cap of 6 (192 KB in flight) is calibrated for the I2P bandwidth envelope.
