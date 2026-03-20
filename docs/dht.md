# Distributed Hash Table — I2PFS-DHT

---

## Design Principles

I2PFS-DHT is a modified Kademlia DHT rebuilt for I2P's high-latency, high-churn environment. Standard Kademlia parameters were derived empirically for 50–200 ms RTTs; applying them unchanged to 500–2000 ms RTTs produces a broken system. Every parameter below has been re-derived for the I2P operating envelope.

### Core Modifications from Standard Kademlia

| Parameter | Standard Kademlia | I2PFS-DHT | Rationale |
|-----------|------------------|-----------|-----------|
| k (bucket size) | 8 | 20 | Higher churn rate requires larger peer sets for consistent routing |
| α (parallelism) | 3 | 1 → 3 (adaptive) | Initial sequential operation avoids tunnel pool exhaustion; increases after first successful hop |
| RPC timeout | 1–3 s | 30 s | Must accommodate tunnel build time + full RTT |
| Lookup timeout | ~10 s | 120 s | Allows 4 sequential hops at 30 s/hop |
| Refresh interval | 1 hour | 30 minutes | Higher churn requires more frequent routing table maintenance |
| Republish interval | 24 hours | 20 hours | Re-announce before TTL expiry with 4-hour safety margin |

---

## Node ID Space and Derivation

The DHT operates in a **256-bit XOR metric space**. Node IDs are derived from an independent `dht_seed` with no relation to any I2P Destination:

```python
# At first launch:
dht_seed = CSPRNG(32 bytes)          # stored encrypted; never transmitted
DHT_ID[0] = BLAKE3(dht_seed || "I2PFS-DHT-epoch" || uint64_le(0))

# Every 48 hours:
DHT_ID[N] = BLAKE3(dht_seed || "I2PFS-DHT-epoch" || uint64_le(N))
```

`DHT_ID[N-1]` remains valid for one epoch after rotation to epoch N. Nodes MUST accept RPCs addressed to either their current or previous DHT_ID.

---

## Routing Table Structure

```
k-bucket structure (k=20, 256 buckets):

Bucket b contains peers at XOR distance [2^b, 2^(b+1))

Each bucket entry (stored as binary record):
{
  dht_id          : 32 bytes     (peer's current DHT identity)
  sam_dest_hash   : 32 bytes     (BLAKE3 of SAM destination for NetDB lookup)
  last_seen       : uint64       (unix seconds)
  rtt_p50         : uint32       (milliseconds, 50th percentile RTT estimate)
  rtt_p95         : uint32       (milliseconds, 95th percentile RTT estimate)
  fail_count      : uint8        (consecutive RPC failures; evict at 3)
  epoch_seen      : uint8        (DHT epoch when this peer was last verified live)
}
```

**Address diversity constraint:** No single I2P address prefix range may occupy more than `k/4 = 5` slots in any bucket. Address prefix diversity is estimated from the first 2 bytes of the SAM destination hash. This mitigates eclipse attacks.

---

## DHT Record Types

### PROVIDER Record

Announces that a node holds a given block. Uses a **blinded provider token** instead of a raw I2P Destination to prevent DHT observers from mapping content to node identities.

```
PROVIDER record (binary, 142 bytes fixed):
├── record_type    : 1 byte   = 0x01
├── dht_key        : 32 bytes (epoch-derived DHT key for this content)
├── provider_token : 64 bytes (blinded; see derivation below)
├── timestamp      : 8 bytes  uint64, unix ms
├── ttl            : 4 bytes  uint32, seconds (max 86,400 = 24 h)
├── epoch          : 1 byte   DHT epoch index (low 8 bits)
└── signature      : 32 bytes Ed25519 signature over all preceding fields
                               (using DHT Signing Key)
```

**Blinded Provider Token derivation:**

```python
provider_token := BLAKE3(
    provider_session_pubkey ||   # 32-byte X25519 public key of Provider SAM Session
    dht_key ||                   # epoch-rotated content DHT key
    "I2PFS-provider-token"
)[:64]
```

A consumer who receives a PROVIDER record and wants to connect:

1. Resolves the provider's encrypted Lease Set (addressed by `BLAKE3(provider_token)[:32]` as a pseudonymous NetDB address)
2. Decrypts the Lease Set using the AnonSwap session PSK (Section 9.3 of [AnonSwap Protocol](anonswap.md))
3. Establishes an AnonSwap session to the provider

### VALUE Record

Used for Mutable Pointer storage:

```
VALUE record (binary, variable length):
├── record_type   : 1 byte  = 0x02
├── dht_key       : 32 bytes
├── seq           : 8 bytes uint64, monotonically increasing
├── value_len     : 2 bytes uint16
├── value         : value_len bytes (≤ 1,000 bytes)
└── signature     : 64 bytes Ed25519 over (dht_key || seq || value_len || value)
```

---

## Lookup Protocol

```python
FIND_PROVIDERS(target_dht_key):

Preconditions: routing table populated with at least k/2 = 10 live peers.

1.  Select initial_peers = 3 closest peers to target_dht_key from local routing table.
    If fewer than 3 available, set α=1 and use all available.

2.  Initialize:
      queried    = {}
      candidates = initial_peers (sorted by XOR distance to target)
      results    = {}  (PROVIDER records)
      α          = 1

3.  Round loop:
    a. Select up to α peers from candidates not in queried, closest to target.
    b. For each selected peer p:
         send FIND_PROVIDERS_RPC(target_dht_key) via DHT SAM Session.
         Set RPC timer = 30 s.
    c. On response from peer p within timeout:
         Mark p as queried. Update p's RTT estimate.
         For each returned PROVIDER record:
           Verify signature. If valid, add to results.
         For each returned peer reference:
           If not in queried and closer than farthest queried: add to candidates.
         If α == 1 and at least one response received: set α = 3.
    d. On timeout from peer p:
         Mark p as queried (failed). Increment p's fail_count.
    e. If α == 1: wait for current RPC before sending next.
       If α == 3: send next batch when any response arrives.

4.  Termination (check after each round):
    a. |results| >= MIN_PROVIDERS (3): success — return results.
    b. All candidates within k-closest have been queried: exhausted — return results.
    c. Total elapsed > LOOKUP_TIMEOUT (120 s): timeout — return results (may be empty).

5.  Return results sorted by (signature_valid, timestamp DESC).
```

!!! note "Why α starts at 1"
    On first invocation, the tunnel pool health is unknown. Sending α=3 simultaneous RPCs on a degraded pool wastes 3 tunnel slots and produces 3 timeouts instead of 1. α increases to 3 only after confirming at least one peer is reachable.

---

## DHT Security

**Record authentication:** All DHT records are signed. Unsigned or invalid-signature records MUST be silently rejected and the sending peer's `fail_count` incremented. Three consecutive failures cause eviction from the routing table.

**Replay protection:** PROVIDER records include a `timestamp` field. Records with timestamps more than 30 minutes in the past, or timestamps in the future, MUST be rejected.

**Sybil resistance:** DHT IDs require computational work proportional to the cost of creating I2P Destinations. Bulk Sybil attacks require bulk I2P Destination creation, which involves a proof-of-work challenge at the I2P NetDB level. A `dht_seed` is committed to at the time the node's SAM sessions are established.

**Eclipse attack mitigation:** Routing table bucket population ensures that no single I2P address prefix range occupies more than `k/4 = 5` slots in any bucket.

---

## RTT Estimation

Each routing table entry maintains rolling RTT statistics using the Jacobson/Karels algorithm:

```python
# On each RPC completion:
rtt_err  = sample_ms - rtt_p50
rtt_p50 += 0.125 * rtt_err
rtt_var += 0.25 * (abs(rtt_err) - rtt_var)
rto      = max(30_000, rtt_p50 + 4 * rtt_var)   # milliseconds

# P95 via reservoir sampling (last 100 samples):
rtt_samples.append(sample_ms)
if len(rtt_samples) > 100:
    rtt_samples.remove_oldest()
rtt_p95 = percentile_95(rtt_samples)
```

Nodes with `rtt_p95 > 1,500 ms` are deprioritized for interactive block retrieval but still used for background replication tasks.
