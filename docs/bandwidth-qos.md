# Bandwidth & QoS Management

---

## Flow Control

I2PFS implements window-based flow control at the application layer, independent of and in addition to I2P's own streaming flow control. The purpose is to prevent a fast publisher from overwhelming a slow provider's tunnel capacity.

```python
# Per-session flow control state:
SEND_WINDOW = clamp(
    value = floor(RTT_p50_ms / chunk_transfer_time_ms) + 2,
    min   = 2,
    max   = 6
)

in_flight = 0

# On SEND_BLOCK sent:    in_flight++
# On ACK received:       in_flight--; send next pending chunk if in_flight < SEND_WINDOW
# On timeout (30 s):     in_flight--; SEND_WINDOW = max(1, SEND_WINDOW // 2); retransmit
# On DONT_HAVE:          in_flight--; route WANT_BLOCK to next provider
```

---

## Bandwidth Quotas

Nodes SHOULD implement configurable bandwidth quotas to prevent I2PFS traffic from monopolizing the shared I2P router's tunnel capacity.

### Default Quotas

| Traffic Class | Default Rate | Notes |
|---|---|---|
| `MAX_UPLOAD_RATE` | 16 KB/s | 50% of typical tunnel capacity |
| `MAX_DOWNLOAD_RATE` | 24 KB/s | 75% — downloads are user-facing, prioritized |
| `MAX_REPLICATION_RATE` | 6 KB/s | Background only |
| `MAX_DHT_RATE` | 2 KB/s | DHT traffic is small but high-frequency |

Bandwidth quotas are enforced using a **token bucket** per traffic class. Bucket refill rate equals the configured quota; bucket depth is 3× the per-message size to allow small bursts.

---

## Priority Queues

The outbound message scheduler uses a strict priority queue with five priority classes:

| Priority | Class | Message Types | Notes |
|----------|-------|--------------|-------|
| P0 — Critical | Handshake | Noise handshake messages, `SESSION_END`, `RATCHET` | Never queued; sent immediately |
| P1 — Interactive | Retrieval | `WANT_BLOCK`, `SEND_BLOCK`, `ACK` for active consumer sessions | Low-latency user-facing |
| P2 — Normal | Serving | `SEND_BLOCK` responses to provider requests from other nodes | Best-effort serving |
| P3 — Background | DHT | DHT RPCs, `FIND_PROVIDERS`, `PROVIDER` announcements | Non-urgent |
| P4 — Bulk | Replication | Background `SEND_BLOCK` for repair operations | Lowest priority |

P0 messages bypass all quotas and are sent immediately. P1–P4 messages are subject to per-class token buckets. P4 (replication) is paused entirely when tunnel rebuild rate exceeds 2/minute.

---

## Congestion Detection

I2P provides no explicit congestion signals. I2PFS infers congestion from observable proxy metrics:

| Signal | Threshold | Response |
|--------|-----------|----------|
| RTT increase | >1.5× baseline for >60 s | Reduce `WINDOW_SIZE` by 1; reduce `MAX_REPLICATION_RATE` by 25% |
| RPC timeout rate | >20% of RPCs timing out | Halve active outbound connections; pause P4 traffic |
| Tunnel rebuild rate | >2 rebuilds/minute | Pause P4 entirely; reduce P3 by 50% |
| Sequential `DONT_HAVE` | >3 from same provider | Deprioritize provider; escalate to next in timeout chain |

Congestion responses are reversed (relaxed) after the triggering metric returns to normal for **120 seconds** continuously.

---

## Configuration

All bandwidth and QoS parameters are configurable in `config.toml`:

```toml
[bandwidth]
max_upload_kbps         = 16
max_download_kbps       = 24
max_replication_kbps    = 6
max_dht_kbps            = 2
pipeline_window_max     = 6
garlic_batch_window_ms  = 50
garlic_batch_max_bytes  = 49152   # 48 KB
```

See [Types, Constants & Wire Formats](types-constants.md#configuration-defaults) for the full configuration reference.
