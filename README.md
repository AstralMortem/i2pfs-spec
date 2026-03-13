# I2PFS — Anonymous Distributed File System for I2P
### Technical Specification v1.0 · Draft · March 2026

---

> **Status:** Draft for public review  
> **Network target:** I2P (Invisible Internet Project)  
> **Inspired by:** IPFS, but redesigned from the ground up for anonymity-first, high-latency operation

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Motivation and Goals](#2-motivation-and-goals)
3. [I2P Network Constraints](#3-i2p-network-constraints)
4. [System Architecture](#4-system-architecture)
5. [Cryptographic Foundations](#5-cryptographic-foundations)
6. [Anonymous Content Identifier (ACI)](#6-anonymous-content-identifier-aci)
7. [Data Model and Chunking](#7-data-model-and-chunking)
8. [Distributed Hash Table — I2P-DHT](#8-distributed-hash-table--i2p-dht)
9. [Block Exchange Protocol — AnonBitSwap](#9-block-exchange-protocol--anonbitswap)
10. [Replication and Fault Tolerance](#10-replication-and-fault-tolerance)
11. [Latency Optimization Layer](#11-latency-optimization-layer)
12. [Node Identity and Anonymity Model](#12-node-identity-and-anonymity-model)
13. [End-to-End Encryption Protocol](#13-end-to-end-encryption-protocol)
14. [Content Publication and Retrieval Flows](#14-content-publication-and-retrieval-flows)
15. [Mutable Content and Naming](#15-mutable-content-and-naming)
16. [Bandwidth and QoS Management](#16-bandwidth-and-qos-management)
17. [Security Threat Model](#17-security-threat-model)
18. [Interoperability and Migration](#18-interoperability-and-migration)
19. [Implementation Recommendations](#19-implementation-recommendations)
20. [Appendices](#20-appendices)

---

## 1. Abstract

**I2PFS** (Invisible Internet Project File System) is a decentralized, content-addressed distributed file system designed exclusively for the I2P anonymous overlay network. It provides complete sender and recipient anonymity, end-to-end data encryption, fault-tolerant decentralized storage, and content distribution optimized for I2P's unique high-latency, bandwidth-constrained tunnel architecture.

Unlike naive ports of IPFS to I2P, I2PFS is redesigned from protocol primitives upward to treat anonymity as a first-class invariant and I2P's latency profile as a primary engineering constraint — not an afterthought.

**Core guarantees provided by this specification:**

- Neither publishers nor consumers can be identified by any network-level observer
- Content is addressed by cryptographic hash, not by location or publisher identity
- All data is encrypted before it leaves the originating node
- No central servers, authorities, or trust anchors exist in the system
- Content survives significant node churn through adaptive replication
- All protocol operations are designed to succeed under 200–2000 ms round-trip times

---

## 2. Motivation and Goals

### 2.1 Why Not IPFS Over I2P?

IPFS assumes a low-latency network. Its DHT (based on Kademlia), block exchange protocol (BitSwap), and connection management are all calibrated for sub-100ms round trips. Running IPFS over I2P results in:

- Frequent DHT lookup timeouts before content is discovered
- BitSwap's want-list protocol collapsing under tunnel rebuild latency
- IPFS peer IDs leaking long-term correlation information
- Content IDs (CIDs) being globally observable and linkable to publisher behavior
- Connection churn defeating IPFS's connection reuse assumptions

### 2.2 Design Goals

| Priority | Goal | Description |
|----------|------|-------------|
| **Critical** | Complete Anonymity | Publisher and consumer identities are computationally unlinkable |
| **Critical** | Content Addressing | Data identified by cryptographic hash of content, not location |
| **Critical** | End-to-End Encryption | Data encrypted before leaving origin; no plaintext traverses the network |
| **High** | Decentralization | No central servers, registries, or certificate authorities |
| **High** | Fault Tolerance | Content survives node departure through proactive replication |
| **High** | Latency Optimization | All operations designed for 200–2000 ms RTT without failure |
| **Medium** | Bandwidth Efficiency | Minimize retransmissions and overhead over constrained tunnels |
| **Medium** | Forward Secrecy | Compromise of long-term keys does not expose past transmissions |

### 2.3 Non-Goals

- Real-time streaming (latency is too variable)
- Sub-second file retrieval (incompatible with I2P tunnel architecture)
- Clearnet accessibility (I2PFS is strictly an I2P-internal protocol)
- Compatibility with IPFS wire protocol

---

## 3. I2P Network Constraints

All I2PFS design decisions are constrained by I2P's operational characteristics. Implementors **MUST** internalize these figures; ignoring them is the primary failure mode of I2P protocol implementations.

### 3.1 I2P Network Parameters

| Parameter | Typical Value | Worst Case | Protocol Impact |
|-----------|--------------|------------|-----------------|
| Round-Trip Time | 500 – 800 ms | 2000+ ms | All timeouts ≥ 30 s; async-first design mandatory |
| Tunnel Lifetime | 10 minutes | N/A | Connections are ephemeral; stateless request design required |
| Tunnel Build Time | 5 – 30 s | 60 s | Connections established lazily; pools pre-warmed |
| Raw MTU (I2NP message) | ~1 KB | 64 KB | Application frames assembled across multiple I2NP messages |
| Streaming MTU | ~60 KB | ~64 KB | Large frames must be segmented at application layer |
| Effective bandwidth | 8 – 32 KB/s | 2 KB/s | Conservative flow control; no burst assumptions |
| Peer reachability | ~60–70% | — | Replication factor must compensate for availability |

### 3.2 Fundamental Asymmetries

I2P tunnels are **unidirectional**. A client builds:

- **Outbound tunnels** — for sending data
- **Inbound tunnels** — for receiving data (published in the NetDB as a Lease Set)

This means every "connection" is actually two independent tunnel paths. I2PFS must handle scenarios where the outbound path succeeds but the response path fails, requiring idempotent, retry-safe request design.

### 3.3 Garlic Routing Properties

I2P uses **garlic routing** — multiple encrypted messages (cloves) bundled into a single garlic message. I2PFS exploits this by:

- Bundling small DHT responses with block data in the same garlic message
- Batching multiple block requests into a single outbound garlic message
- Piggybacking ACK messages onto data responses

---

## 4. System Architecture

### 4.1 Layered Protocol Stack

```
┌─────────────────────────────────────────────┐
│  L6: Content API                            │  Put / Get / Pin / Ls / Resolve
├─────────────────────────────────────────────┤
│  L5: AnonBitSwap (Block Exchange)           │  Latency-aware block trading
├─────────────────────────────────────────────┤
│  L4: I2P-DHT (Routing & Discovery)          │  Modified Kademlia for high-latency
├─────────────────────────────────────────────┤
│  L3: ACI Resolution (Anonymity Layer)       │  Anonymous Content Identifier mapping
├─────────────────────────────────────────────┤
│  L2: I2P Streaming (Transport)              │  Reliable delivery via SAM bridge
├─────────────────────────────────────────────┤
│  L1: I2P Garlic Routing (Network)           │  [Out of scope — provided by I2P]
└─────────────────────────────────────────────┘
```

### 4.2 Node Architecture

Every I2PFS node is **fully symmetric** — all nodes can fulfill all roles simultaneously. Specialization is a performance optimization, not a protocol requirement.

```
┌──────────────────────────────────────────────────────────────┐
│                        I2PFS Node                            │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────┐  │
│  │ Block Store │  │  DHT Store  │  │   Routing Table      │  │
│  │ (encrypted) │  │  (k-bucket) │  │   (I2P Dests)        │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬───────────┘  │
│         │                │                    │              │
│  ┌──────▼─────────────────────────────────────▼────────────┐ │
│  │              I2PFS Protocol Engine                      │ │
│  │   AnonBitSwap │ I2P-DHT │ ACI Resolver │ Replicator     │ │
│  └──────────────────────────────┬──────────────────────────┘ │
│                                 │                            │
│  ┌──────────────────────────────▼───────────────────────────┐│
│  │              I2P SAM Bridge Interface                    ││
│  └──────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

**Node roles:**

- **Publisher** — encrypts content, computes ACIs, inserts into DHT
- **Provider** — stores replicated blocks, responds to AnonBitSwap requests
- **Router** — participates in DHT routing without necessarily storing content
- **Consumer** — retrieves and reassembles content from providers

---

## 5. Cryptographic Foundations

### 5.1 Primitive Selection Rationale

| Primitive | Algorithm | Rationale |
|-----------|-----------|-----------|
| Symmetric encryption | ChaCha20-Poly1305 (AEAD) | Constant-time; high performance on low-power devices; no AES-NI required |
| Key exchange | X25519 (ECDH) | Efficient; constant-time; native I2P crypto compatibility |
| Signatures | Ed25519 | 64-byte signatures; fast batch verification; native I2P support |
| Content hashing | BLAKE3 | Faster than SHA-256; built-in tree-hashing enables streaming verification |
| Key derivation | HKDF-SHA256 | Standard; composable; domain-separated |
| Passphrase KDF | Argon2id | Memory-hard; resists GPU/ASIC attacks on passphrase-protected keys |
| Random generation | OS CSPRNG (getrandom/CryptGenRandom) | Platform-native; no custom entropy pools |

### 5.2 Key Hierarchy

```
Master Identity Key (Ed25519)
│
├── DHT Signing Subkey    [rotated every 48h]
├── Encryption Subkey     [rotated every 24h]
│   └── Per-session ephemeral X25519 key
└── Content Signing Key   [per-publish, never reused]
```

All subkeys are derived via HKDF with domain separation labels. The Master Identity Key **NEVER** appears on the wire.

### 5.3 Forward Secrecy

All I2PFS sessions use ephemeral X25519 keys negotiated via a Noise_XX handshake (see Section 13). Long-term identity keys are used only for authentication, not encryption. Compromise of a long-term key does not expose any past session content.

---

## 6. Anonymous Content Identifier (ACI)

The ACI replaces IPFS's CID. It is designed to be **unlinkable** to the publisher and to prevent observers from correlating retrieval requests with specific content outside the I2P network.

### 6.1 ACI Binary Structure

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├───────┬───────────────┬───────────────────────────────────────────┤
│ Ver   │ Codec (2B)    │ Hash Algo (1B) │ ...                       │
├───────┴───────────────┴───────────────────────────────────────────┤
│                  Blind Factor (32 bytes)                           │
├───────────────────────────────────────────────────────────────────┤
│                  Content Hash (32 bytes)                           │
└───────────────────────────────────────────────────────────────────┘
Total: 68 bytes
```

| Field | Size | Description |
|-------|------|-------------|
| `version` | 1 byte | Always `0x01` for this specification |
| `codec` | 2 bytes | Content type (`0x0000`=raw, `0x0070`=dag-pb, `0x0129`=dag-cbor) |
| `hash_algo` | 1 byte | `0x1E` = BLAKE3 |
| `blind_factor` | 32 bytes | Random bytes generated per-publish, kept secret by publisher |
| `content_hash` | 32 bytes | `BLAKE3(plaintext_content ‖ blind_factor)` |

### 6.2 ACI Properties

**Binding:** The `content_hash` cryptographically binds the ACI to the specific content. Any modification to content produces a different ACI.

**Blindness:** Without knowing `blind_factor`, an adversary who possesses the content cannot compute the ACI, preventing targeted DHT lookups to track content popularity.

**DHT Key Derivation:** Nodes store content under a DHT key derived from but distinct from the ACI:

```
DHT_key := HKDF-SHA256(
    ikm  = ACI,
    salt = "I2PFS-DHT-v1",
    info = network_epoch_bytes,   // changes every 6 hours
    len  = 32
)
```

The epoch rotation means that long-term DHT observation cannot build a persistent map of ACI→location.

### 6.3 Human-Readable Encoding

ACIs are encoded in **Base32 with I2PFS prefix** for human display:

```
i2pfs:bafk2bzace...
```

Base32 is chosen over Base58 because it is case-insensitive and avoids visually ambiguous characters (0/O, 1/l).

---

## 7. Data Model and Chunking

### 7.1 Chunking Strategy

I2PFS splits content into fixed-size **blocks** optimized for I2P tunnel characteristics:

```
┌───────────────────────────────────────────────────────────┐
│                    File / Data Object                     │
│                                                           │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│   │ Block 0  │  │ Block 1  │  │ Block 2  │  │ Block N  │  │
│   │ 32 KB    │  │ 32 KB    │  │ 32 KB    │  │ ≤ 32 KB  │  │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
│                         │                                 │
│              ┌──────────▼──────────┐                      │
│              │    Root ACI Block   │                      │
│              │  (Merkle DAG root)  │                      │
│              └─────────────────────┘                      │
└───────────────────────────────────────────────────────────┘
```

**Default block size: 32 KB**

Rationale:
- Fits comfortably within I2P streaming MTU (~60 KB) with header overhead
- Large enough to amortize per-block DHT lookup cost (expensive at high latency)
- Small enough that a single block loss does not waste excessive bandwidth on retry
- 2× I2P's raw MTU, limiting retransmissions to at most 2 I2NP messages per block

For content < 4 KB, no chunking occurs — the entire content is stored as a single inline block.

### 7.2 Merkle DAG Structure

Blocks are organized into a **Merkle DAG** (Directed Acyclic Graph):

```
Root Block
{
  "type": "file",
  "size": 98304,
  "chunk_size": 32768,
  "encryption_header": { ... },   // see Section 13
  "links": [
    { "aci": "<ACI of block 0>", "size": 32768 },
    { "aci": "<ACI of block 1>", "size": 32768 },
    { "aci": "<ACI of block 2>", "size": 32768 }
  ]
}
```

The root block ACI is the **canonical content address** shared with consumers. Consumers retrieve the root block, parse the link list, then fetch leaf blocks in parallel.

### 7.3 Block Envelope Format

Every block on the wire is wrapped in an authenticated envelope:

```
Block Envelope (binary, little-endian)
├── magic        : 4 bytes  = 0x49 0x32 0x50 0x46  ("I2PF")
├── version      : 1 byte   = 0x01
├── flags        : 1 byte   (bit 0 = compressed, bit 1 = is_root)
├── payload_len  : 4 bytes  (uint32, encrypted payload length)
├── nonce        : 12 bytes (ChaCha20-Poly1305 nonce, random)
├── encrypted_payload : payload_len bytes
└── poly1305_tag : 16 bytes
```

The `encrypted_payload` contains either raw block data (leaf) or serialized DAG node (root).

### 7.4 Block Deduplication

BLAKE3 content hashing provides automatic deduplication. Identical content at any granularity (full file, individual chunk) shares the same ACI and is stored only once per node. This holds across different publishers — two users storing the same file produce identical encrypted blocks only if they use the same encryption key (which they do not by default; see Section 13 for multi-recipient encryption).

---

## 8. Distributed Hash Table — I2P-DHT

### 8.1 Design Principles

I2PFS uses a modified Kademlia DHT adapted for I2P's high-latency, high-churn environment. The key modifications are:

1. **Asynchronous-first:** All DHT operations are non-blocking with explicit timeout chains
2. **Larger k-buckets:** k=20 (vs. Kademlia's typical k=8) to tolerate higher churn
3. **Longer timeouts:** Base timeout 30 s (vs. Kademlia's ~1–3 s)
4. **Parallel factor α=1 initially:** Strict sequential lookups to avoid tunnel congestion; increased to α=3 after first response confirms peer reachability
5. **Signed records:** All DHT records are signed with the publisher's Ed25519 key

### 8.2 Node ID Space

The DHT operates in a 256-bit XOR metric space. Each node's DHT ID is:

```
DHT_ID := BLAKE3(i2p_destination_bytes ‖ "I2PFS-DHT-ID-v1" ‖ epoch_bytes)
```

DHT IDs are rotated every 48 hours (epoch change) to prevent long-term routing table poisoning and traffic analysis. Old IDs remain valid for one additional epoch to allow graceful migration.

### 8.3 Routing Table Structure

```
k-bucket structure (k=20):
├── Bucket 0:  peers at XOR distance [2^255, 2^256)
├── Bucket 1:  peers at XOR distance [2^254, 2^255)
├── ...
└── Bucket 255: peers at XOR distance [1, 2)

Each bucket entry stores:
{
  "dht_id":        32 bytes,
  "i2p_dest":      516 bytes,
  "last_seen":     unix timestamp,
  "rtt_estimate":  milliseconds,
  "reachable":     bool
}
```

### 8.4 DHT Record Types

#### PROVIDER record
Announces that a node holds a given block:

```
PROVIDER {
  "dht_key":    32 bytes,         // DHT_key derived from ACI
  "provider":   I2P Destination,  // who holds the block
  "timestamp":  uint64,           // unix ms
  "ttl":        uint32,           // seconds, max 86400 (24h)
  "signature":  64 bytes          // Ed25519 over all above fields
}
```

#### VALUE record
Stores small values directly in the DHT (used for mutable pointers):

```
VALUE {
  "dht_key":    32 bytes,
  "seq":        uint64,           // monotonic sequence number
  "value":      ≤ 1000 bytes,
  "signature":  64 bytes
}
```

### 8.5 Lookup Protocol (Iterative)

```
FIND_PROVIDERS(target_dht_key):

1. Query local routing table for the k=20 closest known peers to target
2. Send FIND_PROVIDERS RPC to the 3 closest peers (α=3 if >1 response received before)
3. For each response:
   a. If PROVIDER records returned → add to result set, continue in parallel
   b. If closer peers returned → add to candidate set
4. From candidate set, select 3 closest not yet queried → repeat from step 2
5. Terminate when:
   - result set has ≥ MIN_PROVIDERS (default: 3) records, OR
   - all candidates within k-closest have been queried, OR
   - total elapsed time > LOOKUP_TIMEOUT (default: 120 s)
6. Return result set (may be empty on failure)
```

**Lookup timing parameters:**

| Parameter | Default | Rationale |
|-----------|---------|-----------|
| `RPC_TIMEOUT` | 30 s | Accounts for tunnel build + RTT |
| `LOOKUP_TIMEOUT` | 120 s | Allows ~4 sequential hops |
| `MAX_HOPS` | 20 | Standard Kademlia convergence |
| `MIN_PROVIDERS` | 3 | Ensures redundancy before stopping |
| `ALPHA` | 1→3 | Adaptive parallelism |

### 8.6 DHT Security

- All records are signed; unsigned records **MUST** be rejected
- Nodes maintain a reputation table; peers returning invalid signatures are banned for 1 epoch
- Eclipse attack mitigation: routing table peers are selected from diverse I2P address prefixes
- Sybil mitigation: DHT IDs are tied to I2P Destinations, which require computational proof-of-work to generate in bulk

---

## 9. Block Exchange Protocol — AnonBitSwap

### 9.1 Overview

AnonBitSwap is I2PFS's block exchange protocol, replacing IPFS BitSwap. It is redesigned for anonymity (no persistent peer identities in want-lists) and high latency (no assumption of sub-second response).

### 9.2 Key Differences from IPFS BitSwap

| Aspect | IPFS BitSwap | AnonBitSwap |
|--------|-------------|-------------|
| Want-list gossip | Broadcast to all connected peers | Unicast to DHT-discovered providers only |
| Session identity | Persistent peer ID | Per-session ephemeral ID |
| Credit/debt tracking | Yes, per peer | No (eliminates correlation) |
| Parallel requests | Aggressive | Conservative (bandwidth-aware) |
| Timeout | ~1–5 s | 30 s base, 90 s max |
| Retransmit strategy | Immediate | Exponential backoff |

### 9.3 Message Types

```
AnonBitSwap messages (encoded as CBOR):

WANT_BLOCK {
  "session_id":   16 bytes (ephemeral, per-retrieval session),
  "aci":          68 bytes,
  "dht_key":      32 bytes,
  "max_size":     uint32,
  "timestamp":    uint64
}

HAVE_BLOCK {
  "session_id":   16 bytes,
  "aci":          68 bytes,
  "size":         uint32,
  "price":        0       // I2PFS does not implement payment; reserved
}

SEND_BLOCK {
  "session_id":   16 bytes,
  "aci":          68 bytes,
  "block_data":   bytes   // encrypted block envelope (Section 7.3)
}

DONT_HAVE {
  "session_id":   16 bytes,
  "aci":          68 bytes
}
```

### 9.4 Retrieval Flow

```
Consumer                    Provider (DHT-discovered)
   │                                │
   │── WANT_BLOCK(ACI) ────────────►│
   │                                │  (verify ACI, locate block)
   │◄─ HAVE_BLOCK(ACI, size) ───────│
   │                                │
   │── WANT_BLOCK(ACI) [confirm] ──►│  (consumer confirms intent)
   │                                │
   │◄─ SEND_BLOCK(ACI, data) ───────│
   │                                │
   │  [verify BLAKE3(data) == ACI.content_hash]
   │  [decrypt block]
```

### 9.5 Session Anonymity

Each content retrieval uses a fresh `session_id`. Session IDs are **never reused** across different ACI retrievals. This prevents a provider from correlating multiple block requests to the same consumer (even if the provider is malicious).

Consumers route WANT_BLOCK messages through I2P tunnels whose inbound Lease Set is **rotated per retrieval session**, further breaking correlation.

### 9.6 Optimistic Sending

For small blocks (< 4 KB), providers **MAY** send `SEND_BLOCK` immediately after `WANT_BLOCK` without waiting for confirmation, to save one round-trip (~500–800 ms). Consumers **MUST** accept unsolicited `SEND_BLOCK` messages for ACIs they have an active session for.

---

## 10. Replication and Fault Tolerance

### 10.1 Replication Model

I2PFS uses **proactive push replication** rather than IPFS's passive provider announcement model. Content is actively pushed to multiple independent nodes at publish time.

**Target replication factor: R = 5**

This accounts for I2P's ~60–70% peer reachability and typical node churn rates. The probability that at least one of R=5 replicas remains reachable given 70% availability is:

```
P(at least 1 available) = 1 - (1 - 0.70)^5 = 1 - 0.30^5 ≈ 99.76%
```

### 10.2 Replica Selection Algorithm

Replicas are selected to maximize:

1. **DHT distance diversity** — replicas spread across XOR metric space
2. **I2P network diversity** — replicas on different I2P router families (estimated by Destination prefix analysis)
3. **Uptime history** — prefer nodes with high `last_seen` freshness

```
SELECT_REPLICAS(aci, R=5):
1. Perform DHT FIND_PROVIDERS for ACI's DHT_key
2. Collect k=20 nearest responsible nodes from DHT
3. Score each candidate:
   score = w1 * xor_distance_diversity
         + w2 * estimated_uptime_score
         + w3 * inverse_load_estimate
4. Select top R candidates by score
5. Send encrypted block + PROVIDER announcement to each
```

### 10.3 Replication Maintenance

Nodes that store blocks **MUST** re-announce their PROVIDER record before TTL expiry (TTL default: 24 hours, re-announce at 20 hours). If a node detects that fewer than `REPLICATION_MIN` (default: 3) providers exist for a block it stores, it **SHOULD** trigger additional replication to restore the target factor.

### 10.4 Repair Protocol

```
REPAIR_TRIGGER conditions:
- Provider count < REPLICATION_MIN during FIND_PROVIDERS
- PROVIDER record TTL < 4 hours (preemptive)
- Node departure detected (Lease Set no longer in NetDB)

REPAIR_PROCEDURE:
1. Fetch block locally (already stored)
2. Run SELECT_REPLICAS to find new target nodes
3. Offer block via HAVE_BLOCK to new replicas
4. New replicas fetch via SEND_BLOCK
5. New replicas publish PROVIDER records in DHT
```

### 10.5 Pinning

Publishers and consumers can **pin** blocks locally, preventing garbage collection:

```
Pin types:
- RECURSIVE: Pin root ACI and all referenced blocks
- DIRECT:    Pin only the specified ACI block
- INDIRECT:  Block is pinned because it's referenced by a pinned root
```

Pinned blocks are always included in PROVIDER announcements and never garbage collected regardless of access frequency.

---

## 11. Latency Optimization Layer

### 11.1 Pre-Warming Tunnel Pool

Tunnel build time (5–30 s) is the largest source of latency in I2PFS operations. To hide this cost:

- Nodes maintain a **pre-built outbound tunnel pool** of at least 3 ready tunnels
- Tunnels are rebuilt 2 minutes before their 10-minute expiry
- Idle tunnels are recycled but not torn down (I2P allows idle tunnel keepalive)

### 11.2 Request Pipelining

Consumers SHOULD pipeline block requests rather than waiting for each block before requesting the next:

```
Sequential (naive):
  GET block 0 → wait 800ms → GET block 1 → wait 800ms → ...
  Total for 10 blocks: ~8 seconds

Pipelined (I2PFS):
  GET block 0
  GET block 1     ← sent immediately, don't wait
  GET block 2
  ...
  → Responses arrive in ~800ms + data transfer time
  Total for 10 blocks: ~1.5 seconds
```

Pipeline depth limit: **8 concurrent requests** per provider. Exceeding this risks tunnel saturation.

### 11.3 Speculative Prefetch

When a consumer begins reading a sequential file, I2PFS **speculatively prefetches** the next N blocks before they are requested:

```
PREFETCH_WINDOW = ceil(RTT_estimate / block_transfer_time) + 2
```

At 800 ms RTT and ~500 ms per 32 KB block at 64 KB/s: `PREFETCH_WINDOW = ceil(800/500) + 2 = 4 blocks`

Prefetched blocks are cached locally. If not consumed within 5 minutes, they are evicted from cache (but not from provider store if pinned).

### 11.4 Adaptive Timeout Chain

Rather than a single hard timeout, I2PFS uses a **timeout chain** with escalating fallback:

```
Phase 1 [0 – 30s]:   Wait for primary provider response
Phase 2 [30 – 60s]:  Send WANT_BLOCK to secondary provider in parallel
Phase 3 [60 – 90s]:  Send WANT_BLOCK to tertiary provider in parallel
Phase 4 [90 – 120s]: Re-run DHT FIND_PROVIDERS (provider list may be stale)
Phase 5 [120s+]:     Report retrieval failure to caller
```

This cascade ensures that a single slow or unresponsive tunnel does not cause retrieval failure; it just delays the eventual successful response.

### 11.5 Garlic Batching

Multiple small protocol messages destined for the same I2P Destination are batched into a single garlic message to reduce tunnel overhead:

```
Batched garlic cloves:
├── DHT FIND_PROVIDERS RPC
├── AnonBitSwap WANT_BLOCK for block 0
├── AnonBitSwap WANT_BLOCK for block 1
└── HAVE_BLOCK acknowledgment for previously received block
```

Maximum batch size: 48 KB (leaves headroom within 60 KB streaming MTU for I2P framing overhead).

### 11.6 RTT Estimation and Adaptive Behavior

Each peer entry in the routing table maintains a smoothed RTT estimate:

```
RTT_smooth = 0.875 * RTT_smooth + 0.125 * RTT_sample   // EWMA, α=1/8
RTO = max(30s, RTT_smooth * 3 + 4 * RTT_variance)
```

Nodes with high RTT estimates are deprioritized for time-sensitive requests but still used for background replication where latency is not critical.

---

## 12. Node Identity and Anonymity Model

### 12.1 Identity Layers

I2PFS explicitly separates identity concerns into three independent layers:

```
Layer 1: I2P Destination     — used for I2P tunnel routing only
Layer 2: DHT Identity        — used for DHT participation; rotated every 48h
Layer 3: Content Identity    — per-publish Ed25519 key; never reused
```

No layer should be linkable to another by an external observer. Specifically:

- I2P Destinations are **never transmitted in DHT messages** (I2P handles routing transparently)
- DHT IDs are **never published in content headers**
- Content signing keys are **single-use** and derived from a master key that never appears on-wire

### 12.2 Threat Model for Identity

| Attack | Mitigation |
|--------|-----------|
| Passive observation of DHT traffic | All DHT messages encrypted via I2P garlic routing |
| Correlating lookups to a single node | DHT ID rotation every 48h |
| Correlating downloads to identity | Per-retrieval ephemeral session IDs |
| Publisher deanonymization via CID | ACI blind factor prevents CID preimage attacks |
| Long-term traffic analysis | Tunnel rotation; DHT ID rotation |
| Sybil nodes returning false routing data | Signed DHT records; reputation tracking |

### 12.3 Lease Set Privacy

I2P Lease Sets (which advertise a node's inbound tunnels) are published to the I2P NetDB and are theoretically observable. I2PFS mitigates this by:

- Using **Encrypted Lease Sets** (I2P feature) so only authorized nodes can decode them
- Rotating Lease Sets more aggressively than I2P defaults (every 5 minutes instead of 10)
- Using **Blinded Lease Set keys** so that the same I2P Destination has a different NetDB entry across rotations

---

## 13. End-to-End Encryption Protocol

### 13.1 Session Establishment (Noise_XX)

I2PFS uses the **Noise_XX** handshake pattern for session-level encryption between peers:

```
Initiator                           Responder
─────────────────────────────────────────────
→ e                                          // ephemeral pubkey
                         ← e, ee, s, es      // responder ephemeral + static
→ s, se                                      // initiator static
─────────────────────────────────────────────
[Both parties derive session keys]
→ [encrypted application data]
```

The static keys used in Noise_XX are the node's **DHT Identity keys** (not the master keys). Since DHT keys rotate every 48 hours, each new epoch establishes new long-term session keys, providing a form of forward secrecy at the session layer in addition to the per-message ephemeral key exchange.

### 13.2 Content Encryption

Content is encrypted **before chunking**, not per-block. This allows the encryption header to be stored only in the root block:

```
ENCRYPT_CONTENT(plaintext, recipients):

1. Generate content_key (32 bytes, CSPRNG)
2. Generate content_nonce (12 bytes, CSPRNG)
3. ciphertext := ChaCha20-Poly1305(
       key   = content_key,
       nonce = content_nonce,
       aad   = ACI_bytes,
       data  = plaintext
   )
4. For each recipient R:
   a. Generate ephemeral X25519 keypair (ek_pub, ek_priv)
   b. shared_secret := X25519(ek_priv, R.encryption_pubkey)
   c. wrap_key := HKDF-SHA256(shared_secret, salt=ek_pub, info="I2PFS-wrap-v1")
   d. wrapped_key := ChaCha20-Poly1305(wrap_key, nonce=random12, data=content_key)
   e. Store (ek_pub, wrapped_key_nonce, wrapped_key) in encryption header
5. Chunk ciphertext into 32 KB blocks
6. Build Merkle DAG with encryption_header in root block
```

**Encryption header in root block:**

```json
{
  "version": 1,
  "cipher": "chacha20-poly1305",
  "kdf": "hkdf-sha256",
  "content_nonce": "<base64, 12 bytes>",
  "recipients": [
    {
      "ephemeral_pubkey": "<base64, 32 bytes>",
      "wrapped_key_nonce": "<base64, 12 bytes>",
      "wrapped_key": "<base64, 48 bytes>"  // 32-byte key + 16-byte tag
    }
  ]
}
```

### 13.3 Decryption Flow

```
DECRYPT_CONTENT(root_block, recipient_privkey):

1. Parse encryption_header from root_block
2. Find matching recipient entry:
   For each entry in header.recipients:
     shared_secret := X25519(recipient_privkey, entry.ephemeral_pubkey)
     wrap_key := HKDF-SHA256(shared_secret, ...)
     Try: content_key := ChaCha20-Poly1305-Decrypt(wrap_key, entry.wrapped_key)
     If tag valid: found content_key
3. Retrieve all leaf blocks (parallel, see Section 9)
4. Reassemble ciphertext from ordered leaf block data
5. plaintext := ChaCha20-Poly1305-Decrypt(content_key, content_nonce, ciphertext)
6. Verify: BLAKE3(plaintext ‖ blind_factor) == ACI.content_hash
```

### 13.4 Anonymous Multi-Party Sharing

A publisher can encrypt content for multiple recipients simultaneously (step 4 above generates one `wrapped_key` entry per recipient). Recipients cannot determine how many other recipients exist or who they are — each entry in the header is indistinguishable to non-holders.

---

## 14. Content Publication and Retrieval Flows

### 14.1 Publication Flow

```
PUBLISH(file_path, recipients=[]):

1.  Read file content
2.  Generate blind_factor (32 random bytes)
3.  Compute content_hash = BLAKE3(content ‖ blind_factor)
4.  Construct ACI (Section 6.1)
5.  Encrypt content → ciphertext (Section 13.2)
6.  Split ciphertext into 32 KB chunks
7.  For each chunk:
    a. Wrap in Block Envelope (Section 7.3)
    b. Compute leaf ACI
8.  Build Merkle DAG root block (Section 7.2)
9.  Wrap root in Block Envelope
10. Derive DHT_key from root ACI
11. Select R=5 replica nodes (Section 10.2)
12. For each replica node:
    a. Open I2P streaming connection
    b. Perform Noise_XX handshake
    c. Send root block + all leaf blocks via AnonBitSwap
    d. Await HAVE_BLOCK acknowledgments
13. Publish PROVIDER records in DHT for all ACIs
14. Return root ACI to caller (share this + blind_factor with authorized consumers)
```

**Expected publication time for a 1 MB file:**

```
Tunnel pre-warming:         0 s (pool pre-warmed)
Encryption + chunking:      ~50 ms (local)
DHT replica selection:      ~60 s (3 DHT hops × 20 s/hop)
Block transfer (×5 nodes):  ~20 s (pipelined, parallel to 5 nodes)
DHT announcement:           ~30 s (overlaps with transfer)
──────────────────────────────────────────────────────
Total:                      ~60–90 s
```

### 14.2 Retrieval Flow

```
RETRIEVE(root_aci, blind_factor, recipient_privkey):

1.  Derive DHT_key from root_aci
2.  Run DHT FIND_PROVIDERS(DHT_key) → provider_list (Section 8.5)
3.  Send WANT_BLOCK(root_aci) to top-3 providers (adaptive timeout chain)
4.  Receive and verify root block:
    a. Verify Poly1305 tag on Block Envelope
    b. Verify BLAKE3(payload) matches ACI content_hash
5.  Parse encryption_header and DAG links from root block
6.  Decrypt content_key using recipient_privkey (Section 13.3)
7.  For each leaf ACI in DAG links (parallel, pipeline depth 8):
    a. Run DHT FIND_PROVIDERS(leaf_dht_key) [may be cached from publication]
    b. Send WANT_BLOCK to providers
    c. Verify and collect encrypted leaf blocks
8.  Reassemble ordered ciphertext from leaf blocks
9.  Decrypt: plaintext = ChaCha20-Poly1305-Decrypt(content_key, nonce, ciphertext)
10. Final integrity check: BLAKE3(plaintext ‖ blind_factor) == root_aci.content_hash
11. Return plaintext to caller
```

**Expected retrieval time for a 1 MB file:**

```
DHT lookup (root):          ~60 s worst case, ~20 s typical
Root block transfer:        ~1 s
DHT lookups (leaves):       ~20 s (parallelized across 32 leaves)
Leaf block transfers:       ~15 s (pipelined, 8 parallel)
Decryption:                 ~50 ms (local)
──────────────────────────────────────────────────────
Total:                      ~35–80 s
```

---

## 15. Mutable Content and Naming

### 15.1 Problem

ACIs are immutable — they identify a specific version of content. For updating content (e.g., a website, a document that evolves), I2PFS provides **Mutable Pointers**.

### 15.2 Mutable Pointer (MP) Structure

A Mutable Pointer is a signed DHT VALUE record mapping a stable public key to a current root ACI:

```
MP_DHT_key := BLAKE3(mp_pubkey ‖ "I2PFS-MP-v1")

MP_VALUE {
  "version":    1,
  "seq":        uint64,          // monotonically increasing; higher seq wins
  "aci":        68 bytes,        // current root ACI
  "timestamp":  uint64,          // unix ms
  "ttl":        uint32,          // seconds
  "pubkey":     32 bytes,        // Ed25519 public key (stable identity)
  "signature":  64 bytes         // Ed25519 signature over all above fields
}
```

### 15.3 MP Resolution

```
RESOLVE_MP(mp_pubkey):
1.  Compute MP_DHT_key
2.  Run DHT GET_VALUE(MP_DHT_key)
3.  Receive MP_VALUE records from multiple nodes
4.  Verify signature on each record
5.  Return ACI from record with highest valid `seq`
```

### 15.4 MP Update (Publisher)

```
UPDATE_MP(mp_privkey, new_root_aci):
1.  Fetch current MP_VALUE to get latest seq
2.  Construct new MP_VALUE with seq += 1
3.  Sign with mp_privkey
4.  Run DHT PUT_VALUE(MP_DHT_key, MP_VALUE)
5.  DHT nodes accept update only if new seq > stored seq
```

### 15.5 MP Anonymity Properties

The `mp_pubkey` is a stable identifier that can be shared as a "persistent address." However:

- The MP_DHT_key is a hash of the pubkey, not the pubkey itself
- Publishers update MPs via anonymous I2P tunnels
- Rotating the underlying content (ACI) on each update prevents content correlation
- MP records do not contain any publisher identity information

---

## 16. Bandwidth and QoS Management

### 16.1 Flow Control

I2PFS implements window-based flow control at the application layer (independent of I2P's own flow control):

```
SEND_WINDOW = min(8, floor(RTT_smooth / block_transfer_time) + 2)

Sender sends up to SEND_WINDOW blocks without waiting for ACK.
Each received SEND_BLOCK triggers window slide by 1.
On timeout: halve SEND_WINDOW, retransmit timed-out block.
```

### 16.2 Bandwidth Quotas

Nodes **SHOULD** implement configurable bandwidth quotas to prevent I2PFS from monopolizing shared I2P tunnel capacity:

```
Recommended defaults:
- MAX_UPLOAD_RATE:    16 KB/s  (50% of typical tunnel capacity)
- MAX_DOWNLOAD_RATE:  24 KB/s  (75% — downloads prioritized)
- MAX_REPLICATION_RATE: 8 KB/s (background, lower priority)
```

### 16.3 Priority Queues

Outbound message queue is divided into priority classes:

| Priority | Queue | Message Types |
|----------|-------|---------------|
| P0 (Highest) | Real-time | Session handshakes, tunnel keepalives |
| P1 (High) | Interactive | WANT_BLOCK, HAVE_BLOCK for active retrievals |
| P2 (Normal) | Background | SEND_BLOCK for provider requests |
| P3 (Low) | Maintenance | DHT lookups, replication, PROVIDER announcements |

### 16.4 Congestion Detection

I2P provides no explicit congestion signals. I2PFS infers congestion from:

- Rising RTT estimates (> 1.5× baseline → reduce pipeline depth)
- Increasing timeout rate (> 20% RPCs timing out → halve active connections)
- Tunnel rebuild rate (> 2 rebuilds/minute → pause background replication)

---

## 17. Security Threat Model

### 17.1 Adversary Model

I2PFS assumes an adversary with the following capabilities:

- **Global passive adversary:** Can observe all I2P network traffic metadata (but not content, which is encrypted by I2P)
- **Active network adversary:** Can inject, delay, or drop I2P messages
- **Sybil adversary:** Can operate up to 10% of DHT nodes
- **Malicious provider:** A content provider attempts to fingerprint or deanonymize consumers
- **Malicious consumer:** A consumer attempts to deanonymize publishers

I2PFS does **not** protect against:

- An adversary who controls the I2P software on the user's machine
- Statistical disclosure attacks requiring global traffic analysis over months
- Physical access to nodes

### 17.2 Attack Mitigations

#### Publisher Deanonymization

| Vector | Mitigation |
|--------|-----------|
| ACI preimage attack | Blind factor prevents computing ACI from content alone |
| DHT insert timing correlation | Publication occurs via pre-warmed tunnels with jitter delay |
| Replica selection fingerprinting | Randomized replica selection with ±20% score noise |
| Long-term DHT observation | DHT ID rotation every 48h breaks long-term routing correlation |

#### Consumer Deanonymization

| Vector | Mitigation |
|--------|-----------|
| Want-list correlation | Per-retrieval session IDs; no persistent want-list broadcast |
| Tunnel fingerprinting | Lease Set rotation every 5 minutes |
| Provider logging | Session ephemeral keys; provider cannot link two retrievals |
| Timing correlation of lookup + download | Random jitter (0–5 s) between DHT lookup and block request |

#### Content Integrity Attacks

| Vector | Mitigation |
|--------|-----------|
| Block substitution | BLAKE3 verification of every block before acceptance |
| DHT poisoning (false PROVIDER records) | Records are signed; invalid signatures rejected |
| Rollback attack on mutable pointers | Monotonic sequence numbers; highest valid seq wins |
| Chosen-ciphertext attack | ChaCha20-Poly1305 AEAD rejects modified ciphertext |

### 17.3 Cryptographic Agility and Upgrades

Algorithm identifiers are versioned in all wire formats. A future specification version can introduce new primitives by:

1. Incrementing the `version` field in ACI, Block Envelope, and session headers
2. Nodes MUST support all versions within the current major version
3. Old-version blocks remain retrievable as long as any provider stores them

---

## 18. Interoperability and Migration

### 18.1 IPFS Content Bridge

I2PFS can operate a **gateway mode** that fetches IPFS content (via Tor or a trusted clearnet gateway) and re-publishes it to I2PFS, creating a mirror with a new ACI. This allows users to access clearnet IPFS content anonymously.

Note: This breaks IPFS CID compatibility by design — the ACI for bridged content will differ from the original IPFS CID. Content is verified against the original IPFS CID at fetch time, then re-wrapped with I2PFS encryption.

### 18.2 SAM Bridge Requirements

I2PFS requires I2P's SAM (Simple Anonymous Messaging) bridge v3.3 or later:

```
Required SAM capabilities:
- STREAM CONNECT / ACCEPT
- DATAGRAM SEND / RECV
- DEST GENERATE (Ed25519 + X25519)
- LOOKUP (NetDB resolution)
- SESSION CREATE STYLE=STREAM
```

### 18.3 Configuration Reference

```toml
[node]
dht_id_rotation_hours = 48
lease_set_rotation_minutes = 5

[dht]
k_bucket_size = 20
alpha = 3
rpc_timeout_s = 30
lookup_timeout_s = 120
min_providers = 3

[storage]
block_size_bytes = 32768
replication_factor = 5
replication_min = 3
provider_ttl_hours = 24
provider_reannounce_hours = 20
gc_unpinned_after_days = 7

[bandwidth]
max_upload_kbps = 16
max_download_kbps = 24
max_replication_kbps = 8
pipeline_depth = 8

[crypto]
noise_pattern = "XX"
aead = "chacha20-poly1305"
hash = "blake3"
kdf = "hkdf-sha256"

[tunnels]
outbound_pool_size = 3
rebuild_before_expiry_minutes = 2
```

---

## 19. Implementation Recommendations

### 19.1 Language and Runtime

Recommended implementation languages:

- **Rust** — Preferred. Zero-cost abstractions; memory safety; excellent async runtime (tokio); existing I2P SAM bridge libraries.
- **Go** — Acceptable. Strong concurrency; good crypto libraries; existing I2CP/SAM bindings.
- **Java/Kotlin** — Possible. Native I2P is JVM-based, enabling tighter integration.

### 19.2 Storage Backend

| Backend | Recommendation | Notes |
|---------|---------------|-------|
| RocksDB | **Recommended** | High write throughput; compaction handles block churn |
| SQLite | Acceptable for small nodes | Simpler; lower performance under heavy load |
| Filesystem (content-addressed) | Acceptable | Simple; OS-level caching; poor for large block counts |

Block data and DHT routing table **MUST** be stored separately to allow independent compaction and backup.

### 19.3 Mandatory Test Vectors

All implementations **MUST** pass the following test vectors before considering themselves compliant:

1. **ACI Computation:** Given known `content`, `blind_factor`, verify `ACI` matches reference
2. **Block Envelope:** Given known `plaintext`, `key`, `nonce`, verify `envelope` matches reference
3. **DHT Key Derivation:** Given known `ACI`, `epoch`, verify `DHT_key` matches reference
4. **Noise_XX Handshake:** Interoperation test with reference implementation
5. **Content Encryption/Decryption:** Round-trip test with 2-recipient header
6. **Mutable Pointer:** Update, conflict resolution (highest seq wins), and replay rejection

Test vectors are published separately as `I2PFS-testvectors-v1.json`.

### 19.4 Security Audit Requirements

Before any production deployment, implementations MUST undergo:

- External cryptographic review of handshake and encryption code
- Fuzzing of all deserialization paths (Block Envelope, DHT records, AnonBitSwap messages)
- Audit of CSPRNG initialization and entropy sources
- Review of side-channel exposure in crypto primitive usage

---

## 20. Appendices

### Appendix A: Comparison with IPFS

| Aspect | IPFS | I2PFS |
|--------|------|-------|
| Network | Clearnet (+ Tor/I2P overlay) | I2P-native |
| Content ID | CID (multihash, public) | ACI (blinded, unlinkable) |
| Anonymity | Optional, limited | Mandatory, protocol-level |
| Encryption | Optional (application layer) | Mandatory, built-in |
| DHT timeout | 1–5 s | 30 s base |
| Block size | Variable (256 KB default) | Fixed 32 KB |
| Replication | Passive (announce-based) | Proactive push |
| Mutable content | IPNS (public key based) | Mutable Pointers (anonymous) |
| Peer discovery | mDNS + DHT + bootstrap | DHT only (no mDNS) |
| Transport | libp2p (multi-protocol) | I2P Streaming (SAM) |

### Appendix B: Glossary

| Term | Definition |
|------|-----------|
| **ACI** | Anonymous Content Identifier — I2PFS's blinded content address |
| **AnonBitSwap** | I2PFS's block exchange protocol, adapted from IPFS BitSwap |
| **Blind Factor** | 32-byte random value combined with content in ACI hash to prevent preimage attacks |
| **DHT** | Distributed Hash Table — decentralized key-value store for content discovery |
| **Garlic Routing** | I2P's encryption scheme that bundles multiple messages into a single encrypted bundle |
| **I2P-DHT** | I2PFS's modified Kademlia DHT adapted for high-latency operation |
| **Lease Set** | I2P structure advertising a node's inbound tunnel endpoints |
| **MP** | Mutable Pointer — DHT record mapping a stable key to a current ACI |
| **Noise_XX** | Cryptographic handshake pattern providing mutual authentication and forward secrecy |
| **Provider** | Node that stores and serves blocks for a given ACI |
| **SAM** | Simple Anonymous Messaging — I2P's application-layer bridge protocol |
| **Tunnel** | I2P's unidirectional encrypted relay path through multiple routers |

### Appendix C: Wire Format Summary

```
ACI:              68 bytes (1+2+1+32+32)
Block Envelope:   10 + payload + 16 bytes header+tag
DHT PROVIDER:     ~600 bytes (dest + signature + metadata)
Noise_XX msg 1:   32 bytes (ephemeral pubkey)
Noise_XX msg 2:   ~600 bytes (dest + ephemeral + ciphertext)
Noise_XX msg 3:   ~80 bytes
AnonBitSwap WANT: ~100 bytes (CBOR encoded)
AnonBitSwap SEND: 10 + 32768 + 16 bytes (typical leaf block)
```

### Appendix D: Reference Implementation Repository

The reference implementation is maintained at:

```
I2P eepSite: [to be announced on I2P forums]
I2P Git:     http://git.idk.i2p/i2pfs/i2pfs-core.git
```

The reference implementation is written in Rust and licensed under MIT.

---

*End of I2PFS Technical Specification v1.0*
