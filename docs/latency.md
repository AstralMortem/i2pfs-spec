# Latency Optimization

---

## Overview

I2P's latency characteristics differ fundamentally from clearnet: tunnel build time (5–30 s) is the dominant source of latency, not data transfer time. The latency optimization layer is designed to make tunnel build time invisible to operations by ensuring pre-built tunnels are always available when needed.

---

## Pre-Warmed Tunnel Pools

Tunnel build time (5–30 s) is the largest single source of latency in I2PFS operations. The solution is to ensure that by the time any operation needs a tunnel, pre-built tunnels are already available.

| Session | Min pool size | Rebuild trigger |
|---------|--------------|-----------------|
| DHT SAM Session | 3 outbound + 3 inbound | 2 minutes before expiry |
| Provider SAM Session | 2 outbound + 2 inbound | 2 minutes before expiry |
| Client SAM Session | 1 outbound + 2 inbound | Per-retrieval (new session) |

Pool maintenance runs continuously in the background with P4 (lowest) bandwidth priority. If a pool falls below minimum during active operations, tunnel builds are escalated to P0 (critical) priority.

**Tunnel expiry jitter:** Tunnels are rebuilt with a random ±30 second jitter on the rebuild trigger time. This prevents synchronized rebuild storms where all pool tunnels expire simultaneously under high load.

---

## Request Pipelining

Consumers MUST pipeline chunk requests rather than sequentially waiting for each chunk:

```
Sequential retrieval (naive, 10 chunks at 800 ms RTT):
  WANT chunk_0 → 800 ms → receive → WANT chunk_1 → 800 ms → ...
  Total: ~8 seconds + transfer time

Pipelined retrieval (I2PFS, pipeline depth = WINDOW_SIZE):
  WANT chunk_0 ┐
  WANT chunk_1 ├── sent simultaneously
  WANT chunk_2 ┘
  ...
  receive chunk_0 → ACK → WANT chunk_3   (window slides)
  receive chunk_1 → ACK → WANT chunk_4
  ...
  Total: ~800 ms + (10 × transfer_time_per_chunk)
```

### Adaptive Window Size

```python
WINDOW_SIZE = clamp(
    value = floor(RTT_p50_ms / chunk_transfer_time_ms) + 2,
    min   = 2,
    max   = 6
)

# chunk_transfer_time_ms = (32768 * 8) / (effective_bandwidth_kbps * 1000)
```

**Example:** At 800 ms RTT and 16 KB/s effective bandwidth (2,000 ms per chunk):
`WINDOW_SIZE = floor(800 / 2000) + 2 = 2`

Maximum window size is **capped at 6**. A window of 8 at typical I2P bandwidth rates puts 256 KB in flight simultaneously (~8–32 seconds of tunnel capacity), risking tunnel saturation and self-inflicted congestion.

---

## Speculative Prefetch

During sequential file reading, I2PFS prefetches ahead of the current consumption position:

```python
PREFETCH_DEPTH = min(
    ceil(RTT_p50_ms / chunk_transfer_time_ms) + 1,
    WINDOW_SIZE
)
```

Prefetched chunks are held in a local read-ahead cache with a **10-minute TTL**. Prefetch requests use a separate sub-session to avoid head-of-line blocking on the primary retrieval session. Prefetched chunks that are never consumed are evicted after TTL but not removed from provider store if pinned.

---

## Adaptive Timeout Chain

Single-timeout designs fail catastrophically on I2P: a single unresponsive tunnel stalls the entire retrieval. I2PFS uses a cascaded timeout chain:

| Phase | Time window | Action |
|-------|-------------|--------|
| Phase 1 | 0 – 30 s | Wait for `SEND_BLOCK` from primary provider (P1) |
| Phase 2 | 30 – 60 s | Send `WANT_BLOCK` to secondary provider (P2) in parallel; continue waiting for P1 |
| Phase 3 | 60 – 90 s | Send `WANT_BLOCK` to tertiary provider (P3) in parallel; first response from P1/P2/P3 wins |
| Phase 4 | 90 – 120 s | Re-run full `FIND_PROVIDERS` (provider list may be stale due to epoch change); select 3 new providers |
| Phase 5 | 120 s+ | Return `RETRIEVAL_TIMEOUT` to caller; persist block bitmap checkpoint for resume |

This cascade ensures:

- A single slow provider causes at most 30 s additional latency (not failure)
- A stale provider list causes at most 90 s additional latency before refresh
- Complete retrieval failure only occurs after exhausting all 6+ providers and a fresh DHT lookup

---

## Garlic Message Batching

Messages destined for the same I2P Destination within a **50 ms window** are batched into a single garlic message:

```
Example batch (same destination, 50 ms window):
├── DHT FIND_PROVIDERS RPC                    ~80 bytes
├── AnonSwap WANT_BLOCK for chunk 0           ~71 bytes
├── AnonSwap WANT_BLOCK for chunk 1           ~71 bytes
├── AnonSwap WANT_BLOCK for chunk 2           ~71 bytes
└── AnonSwap ACK for previously received      ~71 bytes

Total: ~364 bytes in one tunnel traversal
vs. 5 separate traversals without batching
```

The 50 ms batching window balances responsiveness against efficiency: waiting longer produces better batching but adds latency to time-sensitive requests.

Maximum batch size: **48 KB** (leaving 12 KB headroom within 60 KB streaming MTU).

---

## Leaf Chunk DHT Lookup Optimization

Leaf chunk ACIs derived from the same root will typically be stored on the same set of providers as the root block. After locating root block providers in Phase 1, I2PFS SHOULD preemptively query those same providers for all leaf chunks before running separate DHT lookups per leaf ACI.

This collapses N DHT lookups (one per chunk) into 0 extra lookups in the common case, saving potentially 20+ minutes of DHT lookup time for a large file.

---

## RTT Estimation

Each routing table entry maintains rolling RTT statistics (Jacobson/Karels algorithm). These estimates drive:

- `WINDOW_SIZE` adaptation
- Provider selection (nodes with `rtt_p95 > 1,500 ms` deprioritized for interactive retrieval)
- Timeout chain phase durations
- Congestion detection (RTT increase > 1.5× baseline for > 60 s triggers flow control)

See [Distributed Hash Table](dht.md#rtt-estimation) for the full RTT estimation algorithm.
