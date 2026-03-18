# I2PFS — Anonymous Distributed File System for I2P
### Technical Specification v1.0 · March 2026

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
6. [Anonymous Content Identifier (ACI) and Capability Model](#6-anonymous-content-identifier-aci-and-capability-model)
7. [Data Model and Chunking](#7-data-model-and-chunking)
8. [Distributed Hash Table — I2P-DHT](#8-distributed-hash-table--i2p-dht)
9. [Block Exchange Protocol — AnonSwap](#9-block-exchange-protocol--anonswap)
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

**I2PFS** (Invisible Internet Project File System) is a decentralized, content-addressed distributed file system designed exclusively for the I2P anonymous overlay network. It provides computationally unlinkable sender and recipient anonymity, per-chunk end-to-end encryption with forward secrecy, fault-tolerant distributed storage, and a block exchange protocol optimized for I2P's high-latency, bandwidth-constrained unidirectional tunnel architecture.

I2PFS is designed from cryptographic and network primitives upward with anonymity and I2P's operational envelope as first-class constraints. Every protocol design decision is evaluated against I2P's 500–2000 ms round-trip times, 10-minute tunnel lifetimes, 8–32 KB/s effective bandwidth, and the fundamental requirement that no operation reveals publisher or consumer identity to any observer, including other I2PFS nodes.

**Core guarantees provided by this specification:**

- Neither publishers nor consumers can be identified by any network-level or protocol-level observer
- Content is addressed by cryptographic hash of content combined with a secret capability token — possession of an ACI alone does not grant retrieval capability without the corresponding read capability
- All data is encrypted per-chunk before leaving the originating node; no plaintext ever traverses the network
- No central servers, authorities, or trust anchors exist in the system
- Content survives significant node churn through proactive adaptive replication
- All protocol operations succeed under 200–2000 ms round-trip times using asynchronous, pipelined designs
- Forward secrecy: compromise of long-term keys does not expose past data
- Post-compromise security: a lightweight ratchet limits the blast radius of session key compromise

---

## 2. Motivation and Goals

### 2.1 Why Not IPFS Over I2P?

IPFS was engineered assuming low-latency clearnet operation. Its subsystems fail predictably on I2P:

- The Kademlia DHT uses 1–3 s timeouts calibrated for sub-100 ms RTTs. Under I2P's 500–2000 ms RTTs, lookups time out before the first hop completes.
- BitSwap broadcasts want-lists to all connected peers. I2P tunnels are expensive to establish and short-lived (10 min); maintaining many connections simultaneously collapses the tunnel pool.
- IPFS peer IDs are stable long-term identifiers. On a network where anonymity is the goal, stable peer IDs enable long-term traffic correlation.
- IPFS CIDs are globally public and unblinded. Any observer who knows a CID can query the DHT to find who is storing or requesting specific content.
- IPFS assumes TCP-style persistent connections. I2P's tunnel-based routing provides only ephemeral, unidirectional paths with no guarantee of session continuity beyond 10 minutes.

### 2.2 Design Goals

| Priority | Goal | Description |
|----------|------|-------------|
| **Critical** | Complete Anonymity | Publisher and consumer identities are computationally unlinkable at protocol, DHT, and transport layers |
| **Critical** | Capability-Based Access | Retrieving content requires both the ACI and a read capability token; neither alone is sufficient |
| **Critical** | End-to-End Encryption | Per-chunk encryption with derived nonces; no plaintext traverses the network |
| **Critical** | I2P-Native Operation | All timeouts, message sizes, flow control, and concurrency limits calibrated for I2P constraints |
| **High** | Forward Secrecy | Ephemeral session keys; compromise of long-term keys does not expose past content |
| **High** | Post-Compromise Security | Symmetric ratchet limits exposure window after session key compromise |
| **High** | Fault Tolerance | Content survives node departure through proactive replication with diversity constraints |
| **High** | Resumable Transfers | Block bitmap checkpointing enables interrupted downloads to resume from last checkpoint |
| **Medium** | Bandwidth Efficiency | Fixed binary frames; garlic batching; per-tunnel flow control |
| **Medium** | Streaming Large Files | Per-chunk encryption enables incremental decryption and verification during download |

### 2.3 Non-Goals

- Real-time media streaming (I2P latency variance makes this infeasible; this protocol targets reliable bulk file transfer)
- Sub-10-second file retrieval (incompatible with I2P tunnel architecture; expect 30–120 s for typical files)
- Clearnet accessibility (I2PFS is strictly an I2P-internal protocol)
- Compatibility with IPFS wire protocol (incompatible by design)
- Payment channels or incentive mechanisms (out of scope; nodes participate voluntarily)

---

## 3. I2P Network Constraints

All I2PFS design decisions are constrained by I2P's operational characteristics. Implementors **MUST** internalize these parameters before implementing any subsystem. The single most common failure mode in I2P protocol implementations is ignoring these constraints and designing for clearnet assumptions.

### 3.1 I2P Network Parameters

| Parameter | Typical | Worst Case | Design Implication |
|-----------|---------|------------|-------------------|
| Round-Trip Time | 500–800 ms | 2000+ ms | Minimum RPC timeout: 30 s. All I/O must be non-blocking. |
| Tunnel Lifetime | 10 minutes | 10 minutes (hard) | No connection state may outlive a tunnel. All requests must be idempotent. |
| Tunnel Build Time | 5–30 s | 60 s | Tunnel pools must be pre-warmed. Tunnel build latency must not appear in operation latency. |
| Raw I2NP MTU | ~1 KB | 64 KB | Protocol messages larger than 1 KB require fragmentation at the I2NP layer; I2PFS frames should fit in ≤ 2 I2NP messages. |
| Streaming MTU | ~60 KB | ~64 KB | Application-layer frames must not exceed 48 KB to leave headroom for I2P framing overhead. |
| Effective Bandwidth | 8–32 KB/s | 2 KB/s | Conservative flow control. Pipeline depth must be limited to avoid tunnel saturation. No burst assumptions. |
| Peer Reachability | 60–70% | ~40% | Replication factor of 5 is the minimum for acceptable availability. |
| NetDB Lookup | 5–20 s | 45 s | I2P destination resolution is slow; destinations must be cached aggressively. |

### 3.2 Unidirectional Tunnel Asymmetry

I2P tunnels are strictly unidirectional. Every "connection" between two nodes consists of two independent tunnel paths:

- **Outbound tunnel** — built by the local node, used for sending
- **Inbound tunnel** — built by the local node, published as a Lease Set, used for receiving

The critical implication is that a message can be successfully sent via an outbound tunnel while the response fails because the sender's inbound tunnel has expired or failed. I2PFS handles this by:

1. Designing all requests to be **idempotent** — retransmitting a request always produces the same result
2. Separating outbound and inbound tunnel failure handling — an outbound send failure means retransmit; an inbound response timeout means retry on a different provider or rebuild the inbound tunnel pool
3. Never assuming that a successful send implies a successful receive

### 3.3 Garlic Routing and Message Bundling

I2P uses **garlic routing** — multiple individually encrypted messages (called "cloves") are bundled into a single encrypted garlic message delivered in one tunnel traversal. This is a key optimization opportunity: the marginal cost of adding a small clove to an existing garlic message is zero in terms of additional tunnel hops.

I2PFS exploits garlic routing by:

- Bundling DHT responses with block data in the same garlic message when both are ready simultaneously
- Batching multiple WANT_BLOCK requests for different blocks into a single outbound garlic message to the same provider
- Piggybacking ACK messages and PROVIDER announcements onto data response messages
- Attaching a Lease Set delivery clove to the first message in a new session, eliminating a separate NetDB lookup round trip

Maximum garlic batch size: **48 KB** (leaving 12 KB headroom within the 60 KB streaming MTU for I2P framing and per-clove overhead).

### 3.4 SAM Bridge Constraints

I2PFS communicates with I2P exclusively via the SAM (Simple Anonymous Messaging) bridge v3.3+. SAM imposes its own constraints:

- SAM streaming sessions multiplex over a single TCP connection to the SAM bridge; high message rates can saturate this connection independently of tunnel bandwidth.
- Each SAM session has a single I2P Destination. Multiple independent Destinations require multiple SAM sessions, which is the correct approach for I2PFS's identity separation model.
- Datagram sessions via SAM have a 31 KB payload limit per datagram.
- SAM session creation itself takes 2–10 s; sessions must be created in advance and pooled.

---

## 4. System Architecture

### 4.1 Layered Protocol Stack

```
┌─────────────────────────────────────────────────────────────┐
│  L7: Content API                                            │
│      Put(file) → RootACI+ReadCap                            │
│      Get(RootACI, ReadCap) → file                           │
│      Pin / Unpin / List / Resolve                           │
├─────────────────────────────────────────────────────────────┤
│  L6: Mutable Pointer Layer                                  │
│      MP resolution: stable_key → current RootACI           │
│      Blinded update protocol                                │
├─────────────────────────────────────────────────────────────┤
│  L5: AnonSwap (Block Exchange Protocol)                     │
│      Fixed-binary-frame block request/response              │
│      Ephemeral session IDs; no persistent peer identity     │
│      Resumable download checkpointing                       │
├─────────────────────────────────────────────────────────────┤
│  L4: I2P-DHT (Content Routing)                              │
│      Modified Kademlia; 30 s RPC timeout; k=20              │
│      Blinded provider tokens; signed records                │
├─────────────────────────────────────────────────────────────┤
│  L3: Session Encryption (Noise_IKpsk2 + Ratchet)           │
│      Mutual authentication; forward secrecy                 │
│      Symmetric ratchet for post-compromise security         │
├─────────────────────────────────────────────────────────────┤
│  L2: I2P Streaming / Datagram Transport (SAM v3.3)          │
│      Reliable delivery; unidirectional tunnel abstraction   │
├─────────────────────────────────────────────────────────────┤
│  L1: I2P Garlic Routing (provided by I2P router)            │
│      [Out of scope — operated by local I2P router process]  │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Node Architecture

Every I2PFS node is **fully symmetric** — all nodes are protocol peers capable of fulfilling all roles simultaneously. Role specialization is a runtime performance optimization, never a protocol requirement. A node with 2 GB of storage and a residential I2P connection can provide all four functions.

```
┌─────────────────────────────────────────────────────────────────────┐
│                           I2PFS Node                                │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐   │
│  │  Block Store │  │  DHT Store   │  │     Routing Table        │   │
│  │  (per-chunk  │  │  (k-buckets) │  │  (DHT IDs + SAM Dests)   │   │
│  │   encrypted) │  │              │  │                          │   │
│  └──────┬───────┘  └──────┬───────┘  └───────────┬─────────────┘   │
│         │                 │                      │                  │
│  ┌──────▼─────────────────▼──────────────────────▼───────────────┐  │
│  │                   I2PFS Protocol Engine                       │  │
│  │                                                               │  │
│  │  ┌───────────┐  ┌──────────┐  ┌────────────┐  ┌───────────┐  │  │
│  │  │ AnonSwap  │  │ I2P-DHT  │  │ACI Resolver│  │Replicator │  │  │
│  │  └───────────┘  └──────────┘  └────────────┘  └───────────┘  │  │
│  │                                                               │  │
│  │  ┌───────────────────────────────────────────────────────┐    │  │
│  │  │          Session Manager (Noise + Ratchet)            │    │  │
│  │  └───────────────────────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                  │                                  │
│  ┌───────────────────────────────▼─────────────────────────────┐    │
│  │         I2P SAM Bridge Interface (multiple sessions)        │    │
│  │   DHT SAM Session │ Provider SAM Session │ Client SAM Sess  │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

**SAM session separation is mandatory.** Three independent SAM sessions with independent I2P Destinations are required:

- **DHT Session** — used exclusively for DHT RPC traffic. Keeps DHT identity independent of content serving identity.
- **Provider Session** — used for serving blocks to consumers. Rotated every 24 hours or 1000 connections, whichever comes first.
- **Client Session** — used for fetching content. Rotated per retrieval session (see Section 9.5).

This prevents any single compromised or observed SAM session from linking DHT routing participation to content serving to content retrieval.

**Node roles:**

- **Publisher** — encrypts content per-chunk, computes root ACI, selects replicas, initiates push replication
- **Provider** — stores replicated encrypted blocks, responds to AnonSwap block requests, maintains PROVIDER records in DHT
- **Router** — participates in DHT routing, forwards RPCs, does not necessarily store content blocks
- **Consumer** — resolves ACIs via DHT, fetches blocks from providers, verifies and decrypts

---

## 5. Cryptographic Foundations

### 5.1 Primitive Selection

| Primitive | Algorithm | Rationale |
|-----------|-----------|-----------|
| AEAD encryption | ChaCha20-Poly1305 | Constant-time on all hardware; no AES-NI requirement; 256-bit security; IETF standard (RFC 8439) |
| Key agreement | X25519 (ECDH) | Constant-time; 32-byte keys and outputs; native I2P cryptographic compatibility; high performance |
| Signatures | Ed25519 | 64-byte signatures; fast batch verification; native I2P support; 128-bit security |
| Content hashing | BLAKE3 | ~3× faster than SHA-256; built-in parallel tree hashing enables per-chunk streaming verification without reconstructing the full Merkle tree in memory |
| Key derivation | HKDF-SHA256 | RFC 5869; composable; domain-separated; widely audited |
| Passphrase KDF | Argon2id | Memory-hard (RFC 9106); resists GPU and ASIC brute-force of user-chosen passphrases; t=3, m=65536, p=4 as minimum parameters |
| Session handshake | Noise_IKpsk2 | Pre-shared key variant enables fast re-connection without full DH; responder static key remains hidden from passive observers; 1.5 RTT establishment |
| Symmetric ratchet | BLAKE3-based KDF chain | Lightweight; no per-message DH; provides break-in recovery without Double Ratchet complexity overhead impractical for bulk file transfer |
| Random generation | OS CSPRNG (`getrandom` / `CryptGenRandom`) | Platform-native; no custom entropy pools; mandatory for all key material generation |

### 5.2 Key Hierarchy

The master identity key is a node's long-term identity and **MUST NEVER appear on the wire** in any form, directly or derivably.

```
Master Identity Key (Ed25519, stored encrypted on disk)
│
├── DHT Signing Key          [derived; rotated every 48 h; epoch-indexed]
│   └── Signs DHT records; NEVER used for encryption
│
├── Provider Static Key      [derived; rotated every 24 h or 1000 sessions]
│   └── Noise_IKpsk2 responder static key for AnonSwap sessions
│   └── Advertised in Blinded Provider Token (see Section 8.4)
│
├── Mutable Pointer Key      [derived; rotated per named content channel]
│   └── Ed25519 key for signing MP_VALUE records
│
└── Content Read Key         [derived per-publish from random material]
    └── Used as root of per-chunk key derivation; stored in ReadCap (Section 6)
```

**Subkey derivation:**

All subkeys are derived via HKDF-SHA256 with mandatory domain separation:

```
SubKey := HKDF-SHA256(
    ikm  = master_secret_bytes,
    salt = "I2PFS-v2-" || subkey_label,
    info = epoch_bytes || node_id_commitment,
    len  = 32
)
```

Where `node_id_commitment` is a 32-byte value committed to at node creation time and stored alongside the master key, providing binding without exposing the master key.

### 5.3 Forward Secrecy Guarantee

Forward secrecy in I2PFS operates at two levels:

**Session level:** All AnonSwap sessions use Noise_IKpsk2 with ephemeral X25519 keys generated fresh for each session. Compromise of the Provider Static Key does not expose content of past sessions, only enables impersonation for future sessions until the next key rotation.

**Content level:** Each published file uses a fresh, randomly generated `content_root_key` stored only in the ReadCap (Section 6). Compromise of the publisher's node after publication does not expose the content if the ReadCap was distributed out-of-band and the node's key material is destroyed.

### 5.4 Nonce Management

Nonce reuse under the same key is catastrophic for ChaCha20-Poly1305 (it reveals the XOR of two plaintexts). I2PFS prevents reuse through deterministic nonce derivation:

**Per-chunk content nonces:**
```
chunk_nonce[i] := BLAKE3(content_root_key ‖ "nonce" ‖ uint64_le(i))[:12]
```

Because `content_root_key` is unique per publish, and `i` is the chunk index, no two encryptions under the same key ever produce the same nonce.

**Session nonces (Noise transport phase):**
Standard Noise protocol counter nonces (64-bit little-endian counter, starting at 0). Sessions MUST be torn down and re-established before the counter reaches 2^64 - 1 (in practice, session rotation at 2^32 messages is enforced by the ratchet).

**No random nonces for content encryption.** Random nonces are only used in one context: the ephemeral key wrapping in the ReadCap (Section 6.3), where each wrapping operation uses a fresh random 12-byte nonce. The wrapping key is itself ephemeral (used once), so the nonce space is non-reusable by construction.

---

## 6. Anonymous Content Identifier (ACI) and Capability Model

### 6.1 Capability Separation

v1.0's ACI design had a critical flaw: it embedded `blind_factor` — a value that must remain secret to prevent preimage attacks — directly inside the shareable 68-byte ACI. This allowed anyone who received the ACI to also learn the secret that was supposed to protect privacy.

v2.0 separates content addressing into two distinct objects:

- **ACI (Anonymous Content Identifier)** — the public, shareable address. Uniquely identifies a specific piece of content. Contains no secret material. Can be shared openly.
- **ReadCap (Read Capability)** — the secret required to decrypt and verify the content. Shared only with authorized recipients via the encrypted recipient header. Never shared openly.

**Neither the ACI nor the ReadCap alone is sufficient to retrieve and decrypt content.** Both are required.

### 6.2 ACI Binary Structure

The ACI is a 38-byte fixed-width binary value:

```
 Byte offset:  0    1    2    3    4    5    6   ... 37
               ├────┬─────────┬────┬────────────────────┤
               │Ver │ Codec   │Hash│  Content Hash      │
               │ 1B │   2B    │ 1B │      32 bytes      │
               └────┴─────────┴────┴────────────────────┘
               Total: 38 bytes
```

| Field | Size | Values | Description |
|-------|------|--------|-------------|
| `version` | 1 byte | `0x02` | Protocol version; v2.0 ACIs are incompatible with v1.0 |
| `codec` | 2 bytes | `0x0000`=raw, `0x0001`=i2pfs-dag | Content type identifier |
| `hash_algo` | 1 byte | `0x1E`=BLAKE3 | Hash algorithm identifier |
| `content_hash` | 32 bytes | — | `BLAKE3(ciphertext_root)` — the BLAKE3 hash of the root block's encrypted payload |

**Key design change from v1.0:** `content_hash` is computed over the **ciphertext** (encrypted root block payload), not over the plaintext. This means:

1. No secret material (blind_factor) is needed to compute or verify the ACI.
2. The ACI can be verified by any holder of the ciphertext block, without requiring decryption.
3. The ACI does NOT commit to the plaintext content directly — decryption correctness is verified via the ReadCap (Section 6.3).

### 6.3 ReadCap Structure

The ReadCap is the secret capability required to decrypt content. It is a 76-byte binary structure:

```
 Byte offset:  0                           31  32                          75
               ├──────────────────────────────┬───────────────────────────────┤
               │      content_root_key        │        blind_factor            │
               │         32 bytes             │          44 bytes              │
               └──────────────────────────────┴───────────────────────────────┘
               Total: 76 bytes
```

| Field | Size | Description |
|-------|------|-------------|
| `content_root_key` | 32 bytes | Root key for deriving per-chunk encryption keys (see Section 13.2) |
| `blind_factor` | 32 bytes | Secret value such that `BLAKE3(plaintext ‖ blind_factor)` provides publisher-binding verification |
| `version_tag` | 12 bytes | `"I2PFSReadCap2"` — magic identifier for format validation |

The ReadCap is **never transmitted in plaintext**. It is distributed to authorized consumers via the encrypted recipient header in the root block (Section 13.2). Unauthorized nodes who receive the root block cannot decrypt the ReadCap and therefore cannot verify or decrypt the content.

### 6.4 DHT Key Derivation

Content is located in the DHT using a key derived from the ACI and a time-based epoch:

```
DHT_key := HKDF-SHA256(
    ikm  = ACI_bytes,           // 38 bytes; no secret material
    salt = "I2PFS-DHT-v2",
    info = uint64_le(epoch),    // epoch = unix_time_seconds / 21600 (6-hour epochs)
    len  = 32
)
```

**Why epoch rotation:** A static ACI→DHT_key mapping allows long-term observation of the DHT to build a persistent content→location map. Epoch rotation means that observing DHT traffic in epoch N reveals no useful information for epoch N+1. The 6-hour epoch provides four rotation points per day, balancing privacy against the cost of re-announcement.

**Provider re-announcement timing:** Providers MUST re-announce content in the DHT at the start of each new epoch while still re-announcing before their current-epoch TTL expires. This ensures continuous availability across epoch boundaries.

### 6.5 Human-Readable Encoding

ACIs and ReadCaps are encoded for out-of-band sharing:

```
ACI:     i2pfs2:[BASE32(aci_bytes)]
ReadCap: i2pfsrc:[BASE32(readcap_bytes)]

Example retrieval link (ACI + ReadCap, separated by #):
i2pfs2:ABCDEF...#i2pfsrc:GHIJKL...
```

Base32 (RFC 4648, no padding) is used for case-insensitivity and absence of visually ambiguous characters. The `#` separator between ACI and ReadCap mirrors the URL fragment convention, where the fragment (ReadCap) is never transmitted to servers in HTTP — analogously, the ReadCap should be treated as local-only material and never logged.

---

## 7. Data Model and Chunking

### 7.1 Chunking Strategy

I2PFS splits content into fixed-size **chunks** optimized for I2P's tunnel and bandwidth characteristics.

**Default chunk size: 32,768 bytes (32 KB)**

Rationale for 32 KB:
- Fits within I2P's effective streaming MTU (~60 KB) with room for framing, encryption overhead (28 bytes per chunk), and the AnonSwap frame header (~24 bytes), well within a single I2P streaming window
- Large enough to amortize the DHT lookup cost per chunk (each lookup costs ~30–120 s of latency; a lookup to retrieve 32 KB is far more efficient than one to retrieve 1 KB)
- Small enough that a failed chunk transfer wastes at most 32 KB of bandwidth, not megabytes
- Aligns cleanly with I2P's ~1 KB raw MTU: each 32 KB chunk requires exactly 32 I2NP message segments, predictable for flow control

**Special cases:**

| Content Size | Handling |
|---|---|
| ≤ 4,096 bytes | **Inline block**: Content stored directly in root block body; no leaf chunks; single ACI |
| 4,097 – 32,768 bytes | **Single chunk**: Root block contains one leaf ACI; used to avoid trivially-identifiable single-block files |
| > 32,768 bytes | **Multi-chunk DAG**: Standard chunking into 32 KB leaves with root block DAG |

### 7.2 Merkle DAG Structure (Binary Format)

Blocks are organized into a Merkle DAG. v2.0 uses a compact binary DAG format instead of v1.0's JSON representation.

**Root DAG Block binary format:**

```
Root DAG Block (little-endian binary)
├── magic        : 4 bytes  = 0x49 0x32 0x44 0x47  ("I2DG")
├── version      : 1 byte   = 0x02
├── flags        : 1 byte   bit 0=inline_content, bit 1=has_recipients, bit 2=compressed
├── total_size   : 8 bytes  uint64, original plaintext size in bytes
├── chunk_count  : 4 bytes  uint32, number of leaf chunks
├── chunk_size   : 4 bytes  uint32, nominal chunk size (last chunk may be smaller)
├── recip_count  : 2 bytes  uint16, number of recipient entries in header
├── recip_header_len : 4 bytes uint32, byte length of the recipient header block
├── recipient_header : recip_header_len bytes  (see Section 13.3)
└── chunk_table  : chunk_count × 38 bytes, ordered array of leaf chunk ACIs
```

Total root block size for a 1 MB file (32 chunks): `4+1+1+8+4+4+2+4 = 28` bytes header + recipient header + `32 × 38 = 1,216` bytes chunk table = ~1,244 bytes + recipient entries. This fits comfortably in a single I2P message.

**Leaf Chunk Block format:**

```
Leaf Chunk Block
├── magic        : 4 bytes  = 0x49 0x32 0x43 0x4B  ("I2CK")
├── version      : 1 byte   = 0x02
├── chunk_index  : 4 bytes  uint32, position in the DAG (for ordering verification)
├── payload_len  : 4 bytes  uint32, encrypted payload length
├── encrypted_payload : payload_len bytes  (see Section 13.2)
└── poly1305_tag : 16 bytes  (part of AEAD, appended for framing clarity)
```

**Note on v1.0's double-encryption:** v1.0 wrapped every leaf block in an additional ChaCha20-Poly1305 envelope at the block transport layer, separate from the content encryption. This added 28 bytes of overhead per chunk and a full AEAD operation per block with no security benefit (the content is already authenticated via BLAKE3 in the ACI). v2.0 has a single encryption layer: each leaf chunk's `encrypted_payload` is the AEAD ciphertext produced during content encryption (Section 13.2), and the `poly1305_tag` is the authentication tag from that operation. No separate transport-layer envelope encryption is applied.

### 7.3 Block Deduplication

BLAKE3 content hashing provides automatic deduplication at the chunk level. Two identical 32 KB spans of content from different files will produce the same leaf chunk ACI and will be stored as a single block on any given provider node.

**Cross-publisher deduplication limitation:** Two different publishers encrypting the same plaintext chunk will produce different ciphertexts (because chunk encryption keys are derived from `content_root_key`, which is unique per publish). Cross-publisher deduplication at the block level is intentionally prevented — it would create a traffic analysis oracle. However, publishers can optionally use a deterministic `content_root_key` derived from the plaintext hash for public, deduplication-desirable content (the ReadCap then becomes predictable, so this option SHOULD only be used for content where the publisher explicitly intends broad discoverability).

### 7.4 Download Resume Checkpointing

v2.0 introduces a **block bitmap** checkpoint for resuming interrupted downloads. When a consumer has downloaded some but not all chunks of a file, it persists:

```
Resume Checkpoint (stored locally, never transmitted)
├── root_aci     : 38 bytes
├── readcap      : 76 bytes (encrypted on disk with node key)
├── chunk_count  : uint32
├── bitmap       : ceil(chunk_count / 8) bytes, bit N=1 if chunk N is verified and stored
└── provider_cache : serialized provider list from last DHT lookup
```

On resume, the consumer skips chunks where `bitmap[i] == 1` and only fetches missing chunks. The provider cache avoids a full DHT lookup if the resume occurs within the epoch.

---

## 8. Distributed Hash Table — I2P-DHT

### 8.1 Design Principles

I2P-DHT is a modified Kademlia DHT rebuilt for I2P's high-latency, high-churn environment. The design starts from the observation that standard Kademlia parameters were derived empirically for 50–200 ms RTTs; applying them to 500–2000 ms RTTs without modification produces a broken system. Every timing parameter, concurrency limit, and routing table parameter has been re-derived for the I2P operating envelope.

**Core modifications from standard Kademlia:**

| Parameter | Standard Kademlia | I2P-DHT v2 | Rationale |
|-----------|------------------|------------|-----------|
| k (bucket size) | 8 | 20 | Higher churn rate requires larger peer sets for consistent routing |
| α (parallelism) | 3 | 1 → 3 (adaptive) | Initial sequential operation avoids tunnel pool exhaustion; parallelism increases after first successful hop confirms tunnel health |
| RPC timeout | 1–3 s | 30 s | Must accommodate tunnel build time + full RTT |
| Lookup timeout | ~10 s | 120 s | Allows 4 sequential hops at 30 s/hop |
| Refresh interval | 1 hour | 30 minutes | Higher churn requires more frequent routing table maintenance |
| Republish interval | 24 hours | 20 hours | Re-announce before TTL expiry with 4-hour safety margin |

### 8.2 Node ID Space and Derivation

**Critical v1.0 fix:** v1.0 derived DHT IDs as `BLAKE3(i2p_destination ‖ epoch)`, creating a mathematical link between the node's I2P routing identity and its DHT identity. v2.0 generates DHT IDs independently:

```
At first launch:
  dht_seed := CSPRNG(32 bytes)   // stored encrypted on disk, never transmitted
  DHT_ID[epoch=0] := BLAKE3(dht_seed ‖ "I2PFS-DHT-v2-epoch" ‖ uint64_le(0))

At each epoch change (every 48 hours):
  DHT_ID[epoch=N] := BLAKE3(dht_seed ‖ "I2PFS-DHT-v2-epoch" ‖ uint64_le(N))
```

The `dht_seed` is completely independent of the I2P Destination and the master identity key. An observer who knows a node's DHT ID cannot derive its I2P Destination, and vice versa.

**Old ID validity:** DHT_ID[N-1] remains valid for one full epoch after rotation to epoch N, allowing graceful migration without dropping routing table entries. Nodes MUST accept RPCs addressed to either their current or previous DHT ID.

### 8.3 Routing Table Structure

```
k-bucket structure (k=20, 256 buckets):

Bucket b contains peers at XOR distance [2^b, 2^(b+1))

Each bucket entry:
{
  dht_id          : 32 bytes     (peer's current DHT identity)
  sam_destination : 387 bytes    (compact I2P SAM destination, see Section 8.3.1)
  last_seen       : uint64       (unix seconds)
  rtt_p50         : uint32       (milliseconds, 50th percentile RTT estimate)
  rtt_p95         : uint32       (milliseconds, 95th percentile RTT estimate)
  fail_count      : uint8        (consecutive RPC failures; evict at 3)
  epoch_seen      : uint8        (DHT epoch when this peer was last verified live)
}
```

#### 8.3.1 Compact SAM Destination Format

v1.0 stored full I2P Destinations (516 bytes) in DHT routing table entries. Full Destinations include a long-lived public key that functions as a stable identifier. v2.0 stores only the **SAM address hash** (387 bytes: 256-byte ElGamal key + 128-byte signing key + 3-byte cert), which is sufficient to establish a SAM STREAM CONNECT without exposing the full static key material to DHT observers. The full Destination is resolved via SAM LOOKUP when a connection is needed.

### 8.4 DHT Record Types

#### PROVIDER Record (v2.0)

v1.0 embedded the full I2P Destination in PROVIDER records, creating a permanent content→node-identity mapping in the DHT. v2.0 uses a **blinded provider token** that allows consumers to contact the provider via I2P without the DHT record itself revealing which node is the provider.

```
PROVIDER record (binary, 142 bytes fixed):
├── record_type    : 1 byte   = 0x01
├── dht_key        : 32 bytes (epoch-derived DHT key for this content)
├── provider_token : 64 bytes (blinded; see derivation below)
├── timestamp      : 8 bytes  uint64, unix ms
├── ttl            : 4 bytes  uint32, seconds (max 86,400)
├── epoch          : 1 byte   DHT epoch index (low 8 bits)
└── signature      : 32 bytes Ed25519 signature over all preceding fields
                              (using DHT Signing Key; NOT the provider session key)
```

**Blinded Provider Token derivation:**

```
provider_token := BLAKE3(
    provider_session_pubkey ‖   // 32-byte X25519 public key of Provider SAM Session
    dht_key ‖                   // epoch-rotated content DHT key
    "I2PFS-provider-token-v2"
)[:64]
```

The `provider_session_pubkey` is included in the encrypted Lease Set of the Provider SAM Session. A consumer who receives the PROVIDER record and wants to connect:

1. Resolves the provider's encrypted Lease Set (addressed by `BLAKE3(provider_token)[:32]` as a pseudonymous NetDB address) from the I2P NetDB.
2. Decrypts the Lease Set using a handshake key derivation detailed in Section 9.3.
3. Establishes an AnonSwap session to the provider.

This ensures the DHT record itself does not contain any directly-usable routing information; it requires an additional lookup step that provides a natural anonymization break.

**Verification:** PROVIDER records are signed by the publisher's epoch-specific DHT Signing Key. Nodes MUST verify the signature before storing or forwarding PROVIDER records. Invalid signatures result in immediate peer reputation penalty.

#### VALUE Record

Used for Mutable Pointer storage:

```
VALUE record (binary, variable length):
├── record_type   : 1 byte  = 0x02
├── dht_key       : 32 bytes
├── seq           : 8 bytes uint64, monotonically increasing
├── value_len     : 2 bytes uint16
├── value         : value_len bytes (≤ 1000 bytes)
└── signature     : 64 bytes Ed25519 over (dht_key ‖ seq ‖ value_len ‖ value)
```

### 8.5 Lookup Protocol

```
FIND_PROVIDERS(target_dht_key):

Preconditions: routing table populated with at least k/2 = 10 live peers.

1.  Select initial_peers = 3 closest peers to target_dht_key from local routing table.
    If fewer than 3 are available, set α=1 and use all available peers.

2.  Initialize: 
      queried = {}
      candidates = initial_peers (sorted by XOR distance to target)
      results = {}  (PROVIDER records)

3.  Round loop:
    a. Select up to α peers from candidates not in queried, closest to target.
    b. For each selected peer p:
         send FIND_PROVIDERS_RPC(target_dht_key) via DHT SAM Session.
         Set RPC timer = 30 s.
    c. On response from peer p within timeout:
         Mark p as queried.
         Update p's RTT estimate.
         For each returned PROVIDER record:
           Verify signature. If valid, add to results.
         For each returned peer reference:
           If not already in queried and closer than farthest queried: add to candidates.
         If α == 1 and at least one response received: set α = 3.
    d. On timeout from peer p:
         Mark p as queried (failed). Increment p's fail_count.
    e. If α == 1: wait for current RPC to complete before sending next.
       If α == 3: send next batch when any response arrives.

4.  Termination conditions (check after each round):
    a. |results| >= MIN_PROVIDERS (3): success, return results.
    b. All candidates within k-closest distance have been queried: exhausted, return results (may be empty).
    c. Total elapsed time > LOOKUP_TIMEOUT (120 s): timeout, return results (may be empty).

5.  Return results sorted by (signature_valid, timestamp DESC).
```

**Critical: Parallelism starts at α=1.** On first invocation, I2PFS cannot know whether the tunnel pool is healthy. Sending α=3 simultaneous RPCs on a degraded tunnel pool wastes 3 tunnel slots and produces 3 timeouts instead of 1. α increases to 3 only after confirming at least one peer is reachable.

### 8.6 DHT Security

**Record authentication:** All DHT records are signed. Unsigned or invalid-signature records MUST be silently rejected and the sending peer's fail_count incremented. Three consecutive failures cause eviction from the routing table.

**Sybil resistance:** DHT IDs require computational work to generate that mimics the cost of creating I2P Destinations. Specifically: an I2P Destination requires publishing to the NetDB, which involves a proof-of-work challenge. A node's DHT ID is bound to a `dht_seed` that was committed to at the time the node's SAM sessions were established. Bulk Sybil attacks require bulk I2P Destination creation, which is computationally expensive.

**Eclipse attack mitigation:** The routing table bucket population algorithm ensures that no single I2P address prefix range occupies more than k/4 = 5 slots in any bucket. Address prefix diversity is estimated from the first 2 bytes of the SAM destination hash.

**Replay protection:** PROVIDER records include a `timestamp` field. Records with timestamps more than 30 minutes in the past MUST be rejected. Records with timestamps in the future MUST be rejected.

---

## 9. Block Exchange Protocol — AnonSwap

### 9.1 Overview and Design Philosophy

AnonSwap replaces I2PFS v1.0's AnonBitSwap. The name change reflects significant protocol redesign, not superficial revision. The core design philosophy is:

1. **Fixed binary frames only.** No CBOR, no JSON, no variable-length schema-implicit encoding. Every field is at a fixed offset with a known size, enabling zero-copy deserialization and eliminating parser attack surface.
2. **No persistent peer state.** AnonSwap maintains no per-peer state beyond the current session's Noise handshake keys and ratchet state. There are no credit scores, no want-list histories, no connection scores.
3. **Sessions are ephemeral by design.** An AnonSwap session is created for a specific content retrieval and torn down when that retrieval completes (or fails). Session IDs are never reused.
4. **I2P tunnel budget awareness.** Every design decision accounts for the fact that sending a message costs real tunnel bandwidth. Unnecessary round trips are eliminated by design.

### 9.2 Key Differences from v1.0 AnonBitSwap

| Aspect | v1.0 AnonBitSwap | v2.0 AnonSwap |
|--------|-----------------|---------------|
| Message encoding | CBOR (variable-length, schema-implicit) | Fixed binary frames |
| Session establishment | Noise_XX (3 messages, 1.5 RTT, exposes initiator static key) | Noise_IKpsk2 (2 messages, 1 RTT, responder static key hidden) |
| Session identity | DHT identity key (48h rotation breaks sessions) | Purpose-built provider session key (24h or 1000-session rotation) |
| Post-compromise security | None | Symmetric ratchet every 1000 messages |
| Resumable downloads | Not specified | Block bitmap checkpointing |
| Optimistic send threshold | < 4 KB | Any block (SEND_BLOCK sent immediately, saving 1 RTT) |
| Want-list broadcast | Unicast to DHT providers | Unicast; no broadcast capability exists |

### 9.3 Session Establishment

AnonSwap uses **Noise_IKpsk2** for session establishment. This pattern was chosen for three reasons:

1. **1 RTT establishment**: The initiator sends its ephemeral key and encrypted payload in the first message. The responder replies with its encrypted response. Application data can begin flowing after message 1 (in the payload) or after message 2 (guaranteed decryptable).
2. **Responder static key privacy**: The responder's static key is never revealed to passive observers. An eavesdropper who intercepts the handshake cannot determine which node is acting as provider.
3. **Pre-shared key slot (psk2)**: The PSK position allows use of a content-derived key as an additional authentication factor, ensuring that even with a compromised provider session key, a consumer with the correct ACI can authenticate the session.

```
Noise_IKpsk2 handshake:

  Prologue: "I2PFS-AnonSwap-v2" (used as Noise prologue; both parties must match)

  Initiator (Consumer)                 Responder (Provider)
  ─────────────────────────────────────────────────────────
  → e, es, s, ss, psk                  ← first message (encrypted)
                                        e, ee, se         ← second message
  ─────────────────────────────────────────────────────────
  [Both parties derive: send_key, recv_key, nonce_counters]

  PSK derivation:
    psk := HKDF-SHA256(
        ikm  = dht_key_for_content,    // 32-byte DHT key (public, epoch-derived)
        salt = "I2PFS-session-psk-v2",
        info = session_id,             // 16-byte ephemeral random session ID
        len  = 32
    )
```

The PSK derived from the content's DHT key means that only a consumer who correctly performed a DHT lookup for the specific content can complete the handshake. This is a lightweight capability check that does not require the provider to maintain a per-content access list.

**Noise message sizes:**
- Message 1 (consumer → provider): 32 (ephemeral) + 32+16 (encrypted static) + 32+16 (encrypted payload) = ~128 bytes
- Message 2 (provider → consumer): 32 (ephemeral) + payload = ~32 bytes + response payload

### 9.4 Message Frame Format

All AnonSwap messages use a fixed binary frame:

```
AnonSwap Frame (binary, little-endian)
├── frame_type    : 1 byte   (message type ID; see table below)
├── session_id    : 16 bytes (ephemeral, per-retrieval; random; never reused)
├── sequence      : 8 bytes  uint64, per-session counter; monotonically increasing
├── payload_len   : 4 bytes  uint32
└── payload       : payload_len bytes (type-specific; see below)

Total overhead: 29 bytes per message
```

| frame_type | Name | Direction | Payload |
|------------|------|-----------|---------|
| `0x01` | WANT_BLOCK | Consumer → Provider | `aci[38] ‖ max_size[4]` (42 bytes) |
| `0x02` | HAVE_BLOCK | Provider → Consumer | `aci[38] ‖ size[4] ‖ chunk_index[4]` (46 bytes) |
| `0x03` | SEND_BLOCK | Provider → Consumer | `aci[38] ‖ chunk_index[4] ‖ block_data[var]` |
| `0x04` | DONT_HAVE | Provider → Consumer | `aci[38]` (38 bytes) |
| `0x05` | ACK | Consumer → Provider | `aci[38] ‖ chunk_index[4]` (42 bytes) |
| `0x06` | RESUME_REQ | Consumer → Provider | `aci[38] ‖ bitmap_len[4] ‖ bitmap[var]` |
| `0x07` | RESUME_ACK | Provider → Consumer | `aci[38] ‖ available_bitmap_len[4] ‖ available_bitmap[var]` |
| `0x08` | RATCHET | Either → Either | `new_ratchet_key[32]` (32 bytes; see Section 9.6) |
| `0x09` | SESSION_END | Either → Either | `reason[1]` (1 byte; 0x00=normal, 0x01=error, 0x02=provider_full) |

**Rationale for fixed-width fields:** CBOR (used in v1.0) requires a general-purpose parser with dynamic field lookup. Fixed binary frames are parseable with a single `memcpy` per field — zero allocation, zero schema discovery, zero parser ambiguity. The overhead saving vs. CBOR varies by message type but is typically 15–30 bytes per message.

### 9.5 Retrieval Session Lifecycle

```
Consumer                              Provider
   │                                       │
   │   [Derive PSK from content DHT_key]   │
   │                                       │
   │──── Noise_IKpsk2 Message 1 ──────────►│
   │                                       │  [Verify PSK; establish session]
   │◄─── Noise_IKpsk2 Message 2 ───────────│
   │                                       │
   │  [Optional: RESUME_REQ if checkpoint] │
   │──── RESUME_REQ(aci, bitmap) ─────────►│
   │◄─── RESUME_ACK(aci, avail_bitmap) ────│
   │                                       │
   │  [Send WANT_BLOCK for needed chunks]  │
   │──── WANT_BLOCK(aci, chunk_i) ────────►│  (pipeline: send up to WINDOW_SIZE)
   │──── WANT_BLOCK(aci, chunk_j) ────────►│
   │                                       │
   │◄─── SEND_BLOCK(aci, chunk_i, data) ───│  (optimistic: no HAVE_BLOCK round trip)
   │◄─── SEND_BLOCK(aci, chunk_j, data) ───│
   │                                       │
   │──── ACK(aci, chunk_i) ───────────────►│  (slide window)
   │                                       │
   │  [Verify: BLAKE3(chunk_data) == ACI]  │
   │  [Decrypt chunk, write to buffer]     │
   │                                       │
   │──── SESSION_END(0x00) ───────────────►│
```

**Optimistic sending (v2.0 default):** Providers send `SEND_BLOCK` immediately after receiving `WANT_BLOCK` without waiting for any confirmation. This eliminates the `HAVE_BLOCK → WANT_BLOCK[confirm]` round trip that was optional in v1.0 but wasted 500–800 ms per block. The consumer's `ACK` serves only as a flow control signal (sliding window), not as a delivery confirmation.

Providers that cannot serve a block (not found, capacity exceeded) MUST send `DONT_HAVE` within 5 seconds of receiving `WANT_BLOCK`. Silence is not an acceptable response.

### 9.6 Session Anonymity

**Ephemeral session IDs:** Each AnonSwap session uses a 16-byte random `session_id` generated freshly for the session. Session IDs are never reused. A provider who receives two sessions with different `session_id` values cannot determine whether they originate from the same consumer.

**Per-retrieval client SAM session:** The consumer's AnonSwap sessions originate from a **Client SAM Session** that is rotated for each top-level content retrieval (one root ACI = one SAM session rotation). This ensures the I2P Destination seen by the provider changes with each new retrieval, preventing the provider from building a retrieval history for any consumer.

**No persistent want-list:** v2.0 has no concept of a broadcast want-list. WANT_BLOCK messages are sent unicast only to providers discovered via DHT lookup. A provider learns only that a consumer (with an unknown identity) wants a specific block — not what other blocks the same consumer may be fetching.

### 9.7 Symmetric Ratchet

After every 1,000 AnonSwap messages within a session (tracking the `sequence` counter), both parties advance the symmetric ratchet:

```
RATCHET procedure:

1. The party whose sequence counter hits the 1000-message mark:
   a. Generates ratchet_key_new := CSPRNG(32 bytes)
   b. Sends RATCHET frame with payload = ratchet_key_new
      (RATCHET frame is encrypted under the current Noise send key)
   c. Derives new send key:
        send_key_new := BLAKE3(current_send_key ‖ ratchet_key_new ‖ "ratchet-send")[:32]
   d. Sets send_key = send_key_new; resets sequence counter to 0

2. The receiving party, on RATCHET frame:
   a. Derives new recv key:
        recv_key_new := BLAKE3(current_recv_key ‖ ratchet_key_new ‖ "ratchet-recv")[:32]
   b. Sets recv_key = recv_key_new
```

**Post-compromise security guarantee:** If an attacker compromises the current session keys and then loses access, the ratchet step (which incorporates fresh random material from `ratchet_key_new`) ensures the attacker cannot continue to decrypt future messages. The window of exposure for any compromise is bounded to at most 1,000 messages (~32 MB at 32 KB per message).

---

## 10. Replication and Fault Tolerance

### 10.1 Replication Model

I2PFS uses **proactive push replication**: when content is published, it is actively transferred to multiple independent provider nodes. This contrasts with IPFS's passive model, where content remains only on the publisher's node until other nodes request and cache it.

Proactive replication is essential on I2P because:
- I2P nodes are frequently offline (60–70% reachability)
- Publishers may go offline immediately after publishing, before any consumer has retrieved the content
- Content discovery via DHT requires that at least one provider is online and reachable at retrieval time

**Target replication factor: R = 5**

Statistical basis:
```
P(at least 1 replica available) = 1 − (1 − p_available)^R

At p_available = 0.65: P = 1 − 0.35^5 ≈ 99.47%
At p_available = 0.50: P = 1 − 0.50^5 ≈ 96.88%
At p_available = 0.40: P = 1 − 0.60^5 ≈ 92.22%
```

R=5 provides acceptable availability even in degraded network conditions. R=3 (v1.0's `REPLICATION_MIN`) is a floor, not a target.

### 10.2 Replica Selection Algorithm

Replicas are selected to maximize geographic and structural independence on the I2P network. Because I2P does not expose geographic information, independence is estimated via:

1. **DHT ID diversity:** Replicas should be spread across the 256-bit XOR metric space. Specifically, no two replicas should be within XOR distance 2^240 of each other.
2. **I2P router family diversity:** I2P router families are estimated by the first 8 bytes of the SAM destination hash. No two replicas should share the same estimated router family.
3. **Recent uptime score:** Prefer nodes seen within the past 2 hours.
4. **Capacity signal:** Avoid nodes that have sent `SESSION_END(0x02 = provider_full)` in recent sessions.

```
SELECT_REPLICAS(aci, R=5, candidate_pool):

1.  candidate_pool = k=20 nearest DHT peers to ACI's DHT_key
    (obtained as a side effect of DHT FIND_PROVIDERS, even if no providers exist yet)

2.  For each candidate c in pool:
      score(c) = base_score(c)
               + CSPRNG_jitter(±15%)  // prevents fingerprinting via deterministic selection
    where:
      base_score(c) = w1 × uptime_score(c)
                    + w2 × xor_diversity_score(c, already_selected)
                    + w3 × router_family_diversity_score(c, already_selected)
      w1=0.4, w2=0.4, w3=0.2

3.  Greedily select top R non-overlapping candidates by score.

4.  For each selected replica r:
      a. Establish AnonSwap session (Noise_IKpsk2)
      b. Send all chunks via pipelined SEND_BLOCK frames
      c. Await ACK for each chunk (with timeout)
      d. On full receipt: r publishes PROVIDER record in DHT

5.  If fewer than R replicas acknowledge all chunks within PUBLICATION_TIMEOUT (300 s):
      Retry with next-best candidates from pool.
      Log warning if final replication count < REPLICATION_MIN (3).
```

The ±15% random jitter on scores prevents a traffic analysis attack where an observer can predict which nodes the publisher selected as replicas by running the deterministic selection algorithm themselves.

### 10.3 Replication Maintenance

Provider nodes take active responsibility for content health:

- **TTL re-announcement:** Providers MUST re-announce their PROVIDER record at 20 hours (4 hours before the 24-hour TTL expires). Re-announcement uses the next epoch's DHT_key if a new epoch starts before re-announcement.
- **Under-replication detection:** When a provider performs FIND_PROVIDERS for content it stores (as part of routine DHT participation) and finds fewer than REPLICATION_MIN = 3 providers, it SHOULD trigger a repair operation.
- **Epoch transition:** At each 6-hour epoch boundary, providers MUST re-announce all stored content under the new epoch's DHT_key within a jittered window of ±30 minutes to avoid synchronized announcement storms.

### 10.4 Repair Protocol

```
REPAIR procedure (triggered when provider_count < REPLICATION_MIN):

1.  Verify all stored chunks of the content are intact (BLAKE3 verify each chunk).
    If any chunk fails verification: delete, do not propagate corrupt data.

2.  Run SELECT_REPLICAS to find new candidate nodes not already serving the content.

3.  Contact each candidate via AnonSwap:
      a. Perform Noise_IKpsk2 handshake
      b. For each missing chunk: WANT_BLOCK → SEND_BLOCK (self-push: provider acts as initiator)
         Wait: candidates pull from the repairing provider, who serves SEND_BLOCK

4.  New providers publish PROVIDER records in DHT.

5.  Verify via FIND_PROVIDERS that provider count has recovered to ≥ REPLICATION_MIN.

Repair is performed asynchronously and does not block normal content serving.
```

### 10.5 Pinning

Content can be pinned to prevent garbage collection:

| Pin type | Scope | GC behavior |
|----------|-------|-------------|
| `RECURSIVE` | Root ACI + all leaf chunk ACIs recursively | Never GC'd; always re-announced |
| `DIRECT` | Only the specified ACI block | Block itself is not GC'd; leaf chunks may be |
| `INDIRECT` | Automatically applied to leaf chunks of a RECURSIVE-pinned root | Never GC'd independently |

Unpinned content is garbage collected after `gc_unpinned_after_days` (default: 7) days without a provider re-announcement from any peer requesting that block.

---

## 11. Latency Optimization Layer

### 11.1 Tunnel Pool Management

Tunnel build time (5–30 s) is the largest single source of latency in I2PFS operations. The correct approach is to ensure that by the time any operation needs a tunnel, pre-built tunnels are already available.

**Pre-warmed pool requirements:**

| Session | Min pool size | Rebuild trigger |
|---------|--------------|-----------------|
| DHT SAM Session | 3 outbound + 3 inbound | 2 minutes before expiry |
| Provider SAM Session | 2 outbound + 2 inbound | 2 minutes before expiry |
| Client SAM Session | 1 outbound + 2 inbound | Per-retrieval (new session) |

Pool maintenance runs continuously in the background with P3 (low) bandwidth priority. If a pool falls below minimum during active operations, tunnel builds are escalated to P0 priority.

**Tunnel expiry jitter:** Tunnels are built with a random ±30 second jitter on the rebuild trigger time. This prevents synchronized rebuild storms where all tunnels in the pool expire simultaneously under high load.

### 11.2 Request Pipelining

Consumers SHOULD pipeline chunk requests rather than sequentially waiting for each chunk:

```
Sequential retrieval (naive, 10 chunks at 800 ms RTT):
  WANT block_0 → 800 ms → receive → WANT block_1 → 800 ms → ...
  Total: ~8 seconds + transfer time

Pipelined retrieval (I2PFS default, pipeline depth = WINDOW_SIZE):
  WANT block_0 ┐
  WANT block_1 ├─ sent simultaneously (no waiting)
  WANT block_2 ┘
  ...
  receive block_0 → ACK → WANT block_3 (window slides)
  receive block_1 → ACK → WANT block_4
  ...
  Total: ~800 ms + (10 × transfer_time_per_block)
```

**Adaptive window size:**

```
WINDOW_SIZE = clamp(
    floor(RTT_p50 / block_transfer_time_estimate) + 2,
    min=2,
    max=6
)
```

At 800 ms RTT and 32 KB blocks at an effective 16 KB/s tunnel rate (2 seconds per block):
`WINDOW_SIZE = floor(800/2000) + 2 = 2`

At 500 ms RTT and 32 KB/s bandwidth (1 second per block):
`WINDOW_SIZE = floor(500/1000) + 2 = 2`

**Maximum window size is capped at 6** (not 8 as in v1.0). The v1.0 cap of 8 at typical bandwidth rates put 256 KB in flight simultaneously, filling 8–32 seconds of tunnel capacity. This risks tunnel saturation on the provider side and produces self-inflicted congestion. A cap of 6 (192 KB in flight) is more conservative and appropriate for the I2P bandwidth envelope.

### 11.3 Speculative Prefetch

During sequential file reading (e.g., media playback buffering), I2PFS prefetches ahead of the current consumption position:

```
PREFETCH_DEPTH = min(
    ceil(RTT_p50_ms / block_transfer_time_ms) + 1,
    WINDOW_SIZE
)
```

Prefetched blocks are held in a local read-ahead cache with a 10-minute TTL. They are not counted against the provider's WINDOW_SIZE — prefetch requests use a separate sub-session to avoid head-of-line blocking on the primary retrieval session.

### 11.4 Adaptive Timeout Chain

Single-timeout designs fail catastrophically on I2P: a single unresponsive tunnel stalls the entire retrieval. I2PFS uses a cascaded timeout chain that gracefully escalates to backup providers:

```
Phase 1 [0 – 30 s]:    Wait for SEND_BLOCK from primary provider (P1)
Phase 2 [30 – 60 s]:   Send WANT_BLOCK to secondary provider (P2) in parallel
                        Continue waiting for P1 (cancel if P2 responds first)
Phase 3 [60 – 90 s]:   Send WANT_BLOCK to tertiary provider (P3) in parallel
                        Continue waiting for first response from P1 or P2
Phase 4 [90 – 120 s]:  Re-run full DHT FIND_PROVIDERS
                        (provider list may be stale due to epoch change)
                        Select 3 new providers from updated list
Phase 5 [120 s+]:      Return RETRIEVAL_TIMEOUT error to caller
                        Persist block bitmap checkpoint for resume
```

This cascade ensures that:
- A single slow provider causes at most 30 s additional latency (not failure)
- A stale provider list causes at most 90 s additional latency before refresh
- Complete retrieval failure only occurs after exhausting all 6+ providers and a fresh DHT lookup

### 11.5 Garlic Message Batching

Messages destined for the same I2P Destination within a 50 ms window are batched into a single garlic message:

```
Candidate cloves for a single garlic message (same destination, ≤ 48 KB total):
├── DHT FIND_PROVIDERS RPC                    (~80 bytes)
├── AnonSwap WANT_BLOCK for chunk 0           (~71 bytes)
├── AnonSwap WANT_BLOCK for chunk 1           (~71 bytes)
├── AnonSwap WANT_BLOCK for chunk 2           (~71 bytes)
└── AnonSwap ACK for previously received      (~71 bytes)

Total: ~364 bytes in one tunnel traversal instead of 5 separate traversals
```

The 50 ms batching window is chosen as a balance between responsiveness and efficiency: waiting longer produces better batching but adds latency to time-sensitive requests.

### 11.6 RTT Estimation

Each routing table entry maintains rolling RTT statistics:

```
On each RPC completion:
  sample = measured_rtt_ms

  // Exponential weighted moving average (Jacobson/Karels algorithm)
  rtt_err = sample - rtt_p50
  rtt_p50 += 0.125 * rtt_err
  rtt_var += 0.25 * (|rtt_err| - rtt_var)
  rto = max(30_000, rtt_p50 + 4 * rtt_var)   // in milliseconds

  // P95 via reservoir sampling (last 100 samples)
  rtt_samples.append(sample)
  if len(rtt_samples) > 100: rtt_samples.remove_oldest()
  rtt_p95 = percentile_95(rtt_samples)
```

Nodes with `rtt_p95 > 1500 ms` are deprioritized for interactive block retrieval but still used for background replication tasks where latency is not user-visible.

---

## 12. Node Identity and Anonymity Model

### 12.1 Three-Layer Identity Separation

I2PFS strictly separates identity into three independent layers. No layer should be derivable from or linkable to another by an external observer.

```
Layer 1: I2P Transport Identity
  └── I2P Destination (per SAM session)
  └── Changes every 24 h (Provider) or per-retrieval (Client)
  └── Used ONLY for I2P packet routing; never appears in I2PFS protocol messages
  └── Three separate Destinations: DHT / Provider / Client (separate SAM sessions)

Layer 2: DHT Routing Identity
  └── DHT_ID (derived from independent dht_seed; no relation to I2P Destinations)
  └── Rotates every 48 h
  └── Used ONLY for DHT routing table participation; never appears in block data
  └── The DHT Signing Key is also epoch-specific and rotates with DHT_ID

Layer 3: Content Identity
  └── Per-publish content_root_key (random; never reused)
  └── Per-publish blind_factor (random; stored only in ReadCap; never in ACI)
  └── Content signing keys are ephemeral and single-use
  └── Never linked to any node's DHT ID or I2P Destination
```

**The invariant:** An adversary who compromises one layer gains no information about the other layers. Specifically:
- Knowing a node's DHT_ID does not reveal which I2P Destinations it uses
- Knowing a node's Provider SAM Destination does not reveal its DHT_ID
- Knowing the ACI + ReadCap for content does not reveal who published it or from which node it can be retrieved
- Knowing a PROVIDER record in the DHT does not directly reveal the provider's I2P Destination (blinded token; Section 8.4)

### 12.2 Threat Mitigation Table

| Attack Vector | Mitigation | Residual Risk |
|---|---|---|
| Passive DHT observation | DHT IDs rotate every 48 h; all DHT traffic encrypted by I2P | Long-term statistical correlation over months (accepted) |
| Active DHT query correlation | Per-RPC ephemeral session nonces; jittered timing | Requires sustained active monitoring |
| Publisher identification via ACI preimage | ACI hashes ciphertext, not plaintext; ReadCap contains blind_factor which is secret | None if ReadCap is kept secret |
| Publisher identification via DHT insert timing | Pre-warmed tunnel pools; random jitter 0–300 s on publication | Timing correlation attacks over many publications |
| Consumer identification via block requests | Per-retrieval Client SAM session rotation; per-session ephemeral IDs | Single-session correlation (provider sees one session) |
| Consumer identification via Lease Set | Encrypted Lease Sets; 5-minute rotation; Blinded Lease Set keys | Lease Set metadata observable by I2P floodfill nodes |
| PROVIDER record identity linkage | Blinded provider tokens; token resolved via separate NetDB lookup | Multi-step correlation requiring both DHT and NetDB observation |
| Replica selection fingerprinting | ±15% random score jitter in SELECT_REPLICAS | Probabilistic inference with large observation set |
| Sybil nodes poisoning DHT | Signed records; fail_count-based eviction; PoW via I2P Destination cost | >10% Sybil node ratio degrades routing quality |
| Eclipse attack | Routing table diversity requirements (address prefix, DHT distance) | Sustained targeted attack with many Sybil nodes |

### 12.3 Encrypted Lease Sets

I2P Lease Sets published to the NetDB advertise a node's inbound tunnels and are observable by I2P floodfill nodes. I2PFS MUST use I2P's **Type 3 Encrypted Lease Sets** (also called Blinded Lease Sets) for all SAM sessions:

- The Lease Set is encrypted such that only nodes in possession of a blinding factor can decode it
- The blinding factor for the Provider SAM Session is distributed only to nodes that have completed a valid AnonSwap session (i.e., proved they know the content's DHT key via the PSK)
- Lease Sets are rotated every 5 minutes (vs. I2P's default 10 minutes) to limit the window for traffic correlation

---

## 13. End-to-End Encryption Protocol

### 13.1 Encryption Architecture Overview

v2.0 introduces a fundamental architectural change from v1.0:

**v1.0 (flawed):**
```
plaintext → [encrypt entire file with content_key] → ciphertext → [chunk ciphertext]
                                                                         ↓
                                              [wrap each chunk in block envelope with separate AEAD]
```
This required the entire plaintext in memory before any block could leave the node, and applied two layers of AEAD encryption (block envelope + content) with no security benefit from the outer layer.

**v2.0 (corrected):**
```
plaintext → [chunk plaintext into 32 KB segments]
                         ↓
             [encrypt each chunk independently with derived key and deterministic nonce]
                         ↓
             [each encrypted chunk IS the block stored and transmitted]
                         ↓
             [root block contains chunk ACIs + encrypted ReadCap for each recipient]
```

This enables streaming encryption (chunks can be encrypted and sent while later chunks are still being read), per-chunk verification without full file reassembly, and a single encryption layer with no redundant AEAD overhead.

### 13.2 Content Encryption

```
ENCRYPT_AND_CHUNK(plaintext_bytes, recipients[]):

1.  content_root_key := CSPRNG(32 bytes)   // Never transmitted; stored only in ReadCap
    blind_factor      := CSPRNG(32 bytes)   // Never transmitted; stored only in ReadCap

2.  Split plaintext_bytes into chunks:
    chunks[0..N-1], where chunks[i] = plaintext_bytes[i*32768 : (i+1)*32768]
    (Last chunk may be shorter than 32768 bytes)

3.  For each chunk chunks[i]:
    a.  Derive per-chunk key:
          chunk_key[i] := HKDF-SHA256(
              ikm  = content_root_key,
              salt = "I2PFS-chunk-key-v2",
              info = uint32_le(i),
              len  = 32
          )

    b.  Derive deterministic nonce (prevents nonce reuse under any circumstances):
          chunk_nonce[i] := BLAKE3(
              content_root_key ‖ "I2PFS-nonce-v2" ‖ uint32_le(i)
          )[:12]

    c.  AAD for this chunk (binds ciphertext to its position in the file):
          chunk_aad[i] := root_aci_bytes ‖ uint32_le(i)

    d.  Encrypt:
          (ciphertext[i], tag[i]) := ChaCha20-Poly1305-Seal(
              key   = chunk_key[i],
              nonce = chunk_nonce[i],
              aad   = chunk_aad[i],
              data  = chunks[i]
          )
          // Note: tag is appended to ciphertext in the leaf block format (Section 7.2)

    e.  Compute leaf ACI:
          leaf_content_hash[i] := BLAKE3(ciphertext[i] ‖ tag[i])
          leaf_aci[i] := ACI{ version=0x02, codec=0x0000,
                              hash_algo=0x1E, content_hash=leaf_content_hash[i] }

4.  Compute root content commitment:
    root_commitment := BLAKE3(
        leaf_aci[0] ‖ leaf_aci[1] ‖ ... ‖ leaf_aci[N-1] ‖ blind_factor
    )
    root_aci := ACI{ version=0x02, codec=0x0001,
                     hash_algo=0x1E, content_hash=root_commitment }

5.  Build ReadCap:
    readcap := ReadCap{
        content_root_key = content_root_key,
        blind_factor     = blind_factor,
        version_tag      = "I2PFSReadCap2"
    }

6.  For each recipient R[j]:
    a.  Generate ephemeral X25519 keypair: (ek_pub[j], ek_priv[j]) := X25519_keygen()
    b.  shared_secret[j] := X25519(ek_priv[j], R[j].encryption_pubkey)
    c.  wrap_key[j] := HKDF-SHA256(
            ikm  = shared_secret[j],
            salt = ek_pub[j],
            info = "I2PFS-readcap-wrap-v2" ‖ root_aci_bytes,
            len  = 32
        )
    d.  wrap_nonce[j] := CSPRNG(12 bytes)
        // Random nonce is safe here because wrap_key[j] is single-use (derived from ephemeral key)
    e.  wrapped_readcap[j] := ChaCha20-Poly1305-Seal(
            key   = wrap_key[j],
            nonce = wrap_nonce[j],
            aad   = root_aci_bytes,
            data  = readcap_bytes        // 76 bytes
        )   // Output: 76 + 16 = 92 bytes ciphertext+tag

    f.  recipient_entry[j] := ek_pub[j] ‖ wrap_nonce[j] ‖ wrapped_readcap[j]
                            // 32 + 12 + 92 = 136 bytes per recipient

7.  Build recipient_header:
    recipient_header := uint16_le(len(recipients)) ‖ recipient_entry[0] ‖ ... ‖ recipient_entry[M-1]
    // Total: 2 + M × 136 bytes

8.  Build root DAG block:
    root_block := RootDAGBlock{
        magic=0x49324447, version=0x02, flags=...,
        total_size=len(plaintext_bytes), chunk_count=N,
        chunk_size=32768, recip_count=M,
        recip_header_len=len(recipient_header),
        recipient_header=recipient_header,
        chunk_table=[leaf_aci[0], ..., leaf_aci[N-1]]
    }

9.  Return:
    - root_aci (shareable identifier)
    - root_block + [leaf_block[0], ..., leaf_block[N-1]] (blocks to replicate)
    - DO NOT return readcap directly; it is embedded in the encrypted recipient header
```

### 13.3 Decryption Flow

```
RETRIEVE_AND_DECRYPT(root_aci, root_block, leaf_blocks[], recipient_privkey):

1.  Verify root block integrity:
    root_commitment_actual := BLAKE3(root_block.chunk_table ‖ ??)
    // Wait: we need blind_factor to verify, which is in the ReadCap.
    // Proceed to step 2 first.

2.  Parse recipient_header from root_block.

3.  Find and decrypt ReadCap:
    For each recipient_entry in recipient_header:
        ek_pub := entry[0:32]
        wrap_nonce := entry[32:44]
        wrapped_readcap := entry[44:136]

        shared_secret := X25519(recipient_privkey, ek_pub)
        wrap_key := HKDF-SHA256(
            ikm  = shared_secret,
            salt = ek_pub,
            info = "I2PFS-readcap-wrap-v2" ‖ root_aci_bytes,
            len  = 32
        )
        Try: readcap_bytes := ChaCha20-Poly1305-Open(
                 key=wrap_key, nonce=wrap_nonce, aad=root_aci_bytes,
                 ciphertext=wrapped_readcap
             )
        If tag validates: readcap found. Break.

    If no readcap found: return DECRYPTION_FAILED (not a recipient).

4.  Parse readcap: content_root_key, blind_factor, version_tag.
    Verify version_tag == "I2PFSReadCap2"; else abort.

5.  Verify root block commitment:
    expected_commitment := BLAKE3(
        leaf_aci[0] ‖ ... ‖ leaf_aci[N-1] ‖ blind_factor
    )
    If expected_commitment != root_aci.content_hash: INTEGRITY_FAILED. Abort.
    // This verifies the entire chunk table structure.

6.  For each leaf chunk (in parallel, pipeline per Section 9):
    a.  Verify ciphertext integrity:
          actual_hash := BLAKE3(ciphertext[i] ‖ tag[i])
          If actual_hash != leaf_aci[i].content_hash: CHUNK_INTEGRITY_FAILED. Discard, retry.

    b.  Derive decryption parameters:
          chunk_key[i]   := HKDF-SHA256(content_root_key, "I2PFS-chunk-key-v2", uint32_le(i), 32)
          chunk_nonce[i] := BLAKE3(content_root_key ‖ "I2PFS-nonce-v2" ‖ uint32_le(i))[:12]
          chunk_aad[i]   := root_aci_bytes ‖ uint32_le(i)

    c.  Decrypt:
          plaintext_chunk[i] := ChaCha20-Poly1305-Open(
              key=chunk_key[i], nonce=chunk_nonce[i], aad=chunk_aad[i],
              ciphertext=ciphertext[i] ‖ tag[i]
          )
          If AEAD fails: CHUNK_DECRYPTION_FAILED. Block is corrupt or tampered. Retry from different provider.

7.  Reassemble: plaintext = plaintext_chunk[0] ‖ ... ‖ plaintext_chunk[N-1]
    Truncate to root_block.total_size bytes (removes padding in last chunk).

8.  Return plaintext.
```

**Security note on the AAD binding (`chunk_aad[i] = root_aci ‖ chunk_index`):** This binding prevents a provider from shuffling encrypted chunks between different files or different positions within the same file. A chunk intended for position 5 of file A cannot be passed off as position 3 of file B — the AEAD authentication tag will fail because the AAD does not match.

### 13.4 Anonymous Multi-Party Sharing

Multiple recipients are supported by adding additional entries to the `recipient_header` (step 6 of Section 13.2). Each entry is 136 bytes. For M recipients, the header grows by 136×M bytes.

**Recipient count privacy:** All recipient entries are fixed-size and indistinguishable from each other to non-holders. A node that receives the root block but is not a designated recipient can see how many recipients exist (from `recip_count`) but cannot determine the identities of any recipient. If the publisher wishes to conceal the number of recipients, they MAY add dummy entries encrypted to randomly generated keys (not corresponding to any real recipient). Consumers who are genuine recipients iterate all entries until their decryption succeeds.

---

## 14. Content Publication and Retrieval Flows

### 14.1 Publication Flow

```
PUBLISH(file_path, recipients=[]):

Phase 1: Local preparation (no network operations; ~50–200 ms)
  1.  Read file_path → plaintext_bytes
  2.  Run ENCRYPT_AND_CHUNK(plaintext_bytes, recipients)
      → root_aci, root_block, leaf_blocks[]
  3.  Persist root_block and leaf_blocks[] to local Block Store (encrypted at rest)

Phase 2: DHT discovery (concurrent with Phase 3; ~60 s)
  4.  Derive DHT_key from root_aci
  5.  Run DHT FIND_PROVIDERS(DHT_key)
      → candidate_pool (20 nearest DHT nodes; actual providers irrelevant for new content)

Phase 3: Replica push (concurrent with Phase 2 once first DHT response arrives)
  6.  Run SELECT_REPLICAS(root_aci, R=5, candidate_pool)
      → replica_nodes[]
  7.  For each replica_node in parallel (up to R connections simultaneously):
      a.  Establish AnonSwap session (Noise_IKpsk2 handshake)
      b.  Send root_block via SEND_BLOCK (single block, immediate)
      c.  For each leaf_block[i]:
            Pipeline SEND_BLOCK frames up to WINDOW_SIZE
            Handle ACK frames to advance pipeline window
      d.  On all chunks ACK'd: replica_node is confirmed

Phase 4: DHT announcement (after Phase 3 completes; ~30 s)
  8.  For each confirmed replica_node:
        replica_node publishes PROVIDER record for root_aci in DHT
        replica_node publishes PROVIDER record for each leaf_aci in DHT
  9.  Local node also publishes PROVIDER record (self is also a provider initially)

Phase 5: Return to caller
  10. Return (root_aci, readcap_not_transmitted)
      // The ReadCap was embedded in the root_block's recipient_header in Phase 1.
      // The caller may extract it by decrypting with their own recipient_privkey
      // after fetching the root_block they just published. This is intentional:
      // the publisher accesses their own content the same way any other recipient does.
```

**Expected publication time for a 1 MB file (32 chunks):**

```
Phase 1 (encryption + chunking):         ~100 ms
Phase 2 (DHT lookup, 3 hops):            ~60–90 s
Phase 3 (replica push × 5, pipelined):   ~25–40 s (overlaps with Phase 2 after first hop)
Phase 4 (DHT announcement, async):       ~30 s (overlaps with Phase 3)
─────────────────────────────────────────────────────────
Total:                                   ~70–110 s
```

The dominant cost is DHT discovery. For a publisher who has recently published other content and has a warm routing table, Phase 2 can complete in ~20 s.

### 14.2 Retrieval Flow

```
RETRIEVE(root_aci, recipient_privkey):

Phase 1: Root block discovery (~20–90 s)
  1.  Derive DHT_key from root_aci (epoch-dependent; use current epoch)
  2.  Run DHT FIND_PROVIDERS(DHT_key) → provider_list
      (Adaptive timeout chain; see Section 11.4)
  3.  Send WANT_BLOCK(root_aci) to top-3 providers simultaneously
  4.  Receive root_block from first responding provider
  5.  Verify: BLAKE3(root_block.encrypted_payload) == root_aci.content_hash
      If fails: discard, retry on next provider

Phase 2: ReadCap extraction and root verification (~1 ms local)
  6.  Parse recipient_header from root_block
  7.  Decrypt ReadCap using recipient_privkey (iterate recipient entries)
      If not found: DECRYPTION_FAILED (caller is not a recipient)
  8.  Verify root block commitment using blind_factor from ReadCap

Phase 3: Chunk retrieval (parallel; ~15–40 s for 1 MB)
  9.  Check local resume checkpoint: load block bitmap if it exists for this root_aci
  10. For each chunk i where bitmap[i] == 0 (not yet retrieved):
      a.  Derive leaf_dht_key from leaf_aci[i]
      b.  Check provider_cache in checkpoint; if valid for current epoch, skip DHT lookup
          Otherwise: run DHT FIND_PROVIDERS(leaf_dht_key)
      c.  Send WANT_BLOCK(leaf_aci[i]) via AnonSwap (pipelined, WINDOW_SIZE chunks)
      d.  Receive SEND_BLOCK response
      e.  Verify: BLAKE3(chunk_data ‖ tag) == leaf_aci[i].content_hash
          If fails: retry on next provider
      f.  Decrypt chunk using chunk_key[i] and chunk_nonce[i] (derived from ReadCap)
      g.  Verify AEAD tag: if fails, block is corrupt, retry from different provider
      h.  Write plaintext_chunk[i] to output buffer
      i.  Update bitmap[i] = 1; persist checkpoint

Phase 4: Reassembly (~1 ms local)
  11. Reassemble: plaintext = plaintext_chunk[0] ‖ ... ‖ plaintext_chunk[N-1]
  12. Truncate to root_block.total_size
  13. Delete resume checkpoint (retrieval complete)
  14. Return plaintext
```

**Expected retrieval time for a 1 MB file (32 leaf chunks):**

```
Phase 1 (DHT lookup + root block):         ~20–60 s typical, ~90 s worst case
Phase 2 (ReadCap decrypt, local):          < 1 ms
Phase 3 (leaf chunk DHT + transfer):       ~15–30 s (parallelized, pipelined)
Phase 4 (reassembly, local):               < 5 ms
───────────────────────────────────────────────────────────────────────────
Total (typical):                           ~35–90 s
Total (worst case, cold routing table):    ~120–150 s
```

**Leaf chunk DHT lookup optimization:** Leaf chunk ACIs derived from the same root will typically be stored on the same set of providers as the root block. After locating root block providers in Phase 1, I2PFS SHOULD preemptively query those same providers for all leaf chunks before running separate DHT lookups. This collapses N DHT lookups (one per chunk) into 0 extra lookups in the common case.

---

## 15. Mutable Content and Naming

### 15.1 The Mutability Problem

Content-addressed systems are inherently immutable: the ACI of a file is derived from its content, so any change produces a new ACI. For evolving content (a continuously updated news feed, a software repository, a personal file storage root), users need a stable address that can be updated to point to new content versions.

I2PFS provides **Mutable Pointers (MPs)**: signed DHT records that map a stable public key to a current root ACI.

### 15.2 Mutable Pointer Key Blinding

v1.0 exposed the `mp_pubkey` in MP_VALUE records stored in the DHT, making the public key observable. v2.0 uses a **blinded signing scheme** where the DHT key for the MP record is derived from but does not reveal the publisher's stable signing key.

```
MP key setup:

  mp_master_privkey := Ed25519_keygen()  // long-term; kept secret
  mp_master_pubkey  := corresponding public key  // shared as "stable address"

  // Blind factor changes each epoch to prevent long-term tracking
  mp_blind_factor[epoch] := HKDF-SHA256(
      ikm  = mp_master_privkey,
      salt = "I2PFS-MP-blind-v2",
      info = uint64_le(epoch),
      len  = 32
  )

  // Blinded signing key for epoch (derived, not the master key)
  mp_epoch_privkey[epoch], mp_epoch_pubkey[epoch] := Ed25519_derive_subkey(
      master_key = mp_master_privkey,
      derivation_path = "I2PFS-MP-epoch" ‖ uint64_le(epoch)
  )

  // DHT key for epoch
  MP_DHT_key[epoch] := BLAKE3(
      mp_master_pubkey ‖ "I2PFS-MP-DHT-v2" ‖ uint64_le(epoch)
  )
```

The `mp_master_pubkey` is the persistent "channel address" shared with subscribers. Subscribers who know `mp_master_pubkey` can derive `MP_DHT_key[current_epoch]` to look up the current content pointer. The DHT record itself is signed with the epoch-specific subkey, which rotates every 48 hours.

An observer who captures MP records from multiple epochs cannot link them to the same publisher without knowing the `mp_master_pubkey` (which the subscriber must know out-of-band).

### 15.3 Mutable Pointer Record Structure

```
MP_VALUE record (binary, 183 bytes):
├── record_type         : 1 byte   = 0x03
├── dht_key             : 32 bytes (MP_DHT_key for current epoch)
├── epoch               : 8 bytes  uint64
├── seq                 : 8 bytes  uint64 (monotonically increasing; higher wins)
├── root_aci            : 38 bytes (current content ACI)
├── timestamp           : 8 bytes  uint64 unix ms
├── ttl                 : 4 bytes  uint32 seconds (max 172,800 = 48 h)
├── epoch_pubkey        : 32 bytes (mp_epoch_pubkey for this epoch; for signature verification)
├── pubkey_binding_proof: 64 bytes (Ed25519 signature of epoch_pubkey by mp_master_privkey)
└── signature           : 64 bytes (Ed25519 signature by mp_epoch_privkey over all preceding)
```

The `pubkey_binding_proof` allows a subscriber who knows `mp_master_pubkey` to verify that the `epoch_pubkey` is legitimately derived from the master key, without the master key appearing in the DHT record itself.

### 15.4 MP Resolution

```
RESOLVE_MP(mp_master_pubkey):
1.  Compute MP_DHT_key for current epoch
2.  Run DHT GET_VALUE(MP_DHT_key) → list of MP_VALUE records from multiple nodes
3.  For each record:
    a.  Verify epoch matches current epoch (±1 epoch for transition tolerance)
    b.  Verify pubkey_binding_proof using mp_master_pubkey
    c.  Verify signature using epoch_pubkey
    d.  If all valid: add to candidate set
4.  Return root_aci from the record with highest valid seq
    (highest seq is the most recent update; this is a last-writer-wins scheme)
```

**Epoch transition handling:** During the 48-hour epoch change, both the old and new epoch's MP_DHT_key are valid. Resolvers SHOULD check both keys and merge results by seq number.

### 15.5 MP Update

```
UPDATE_MP(mp_master_privkey, new_root_aci):
1.  Fetch current MP_VALUE to get the latest seq (via RESOLVE_MP)
    new_seq := latest_seq + 1
2.  Construct MP_VALUE:
    a.  Derive mp_epoch_privkey for current epoch
    b.  epoch_pubkey := corresponding Ed25519 public key
    c.  pubkey_binding_proof := Ed25519_sign(mp_master_privkey, epoch_pubkey)
    d.  Fill all fields; seq = new_seq
    e.  signature := Ed25519_sign(mp_epoch_privkey, all_preceding_fields)
3.  Run DHT PUT_VALUE(MP_DHT_key, MP_VALUE)
    DHT nodes store update only if new_seq > stored_seq for same dht_key
4.  Return MP_DHT_key (stable lookup address for this epoch)
```

**Replay prevention:** Nodes MUST reject PUT_VALUE for a dht_key if the incoming `seq` ≤ stored `seq`. This prevents a replay of an old (lower-seq) record from rolling back content.

---

## 16. Bandwidth and QoS Management

### 16.1 Flow Control

I2PFS implements window-based flow control at the application layer, independent of and in addition to I2P's own streaming flow control. The purpose is to prevent a fast publisher from overwhelming a slow provider's tunnel capacity.

```
Per-session flow control:

SEND_WINDOW = clamp(
    value  = floor(RTT_p50_ms / block_transfer_time_ms) + 2,
    min    = 2,
    max    = 6     // not 8 as in v1.0; see Section 11.2 rationale
)

block_transfer_time_ms = (32768 * 8) / (effective_bandwidth_kbps * 1000)

// Window management:
in_flight = 0  // count of SEND_BLOCK frames sent but not ACK'd
On SEND_BLOCK sent:  in_flight++
On ACK received:     in_flight--; send next pending block if in_flight < SEND_WINDOW
On TIMEOUT (30 s):   in_flight--; halve SEND_WINDOW (min 1); retransmit
On DONT_HAVE:        in_flight--; route WANT_BLOCK to next provider
```

### 16.2 Bandwidth Quotas

Nodes SHOULD implement configurable bandwidth quotas to prevent I2PFS traffic from monopolizing the shared I2P router's tunnel capacity:

```
Recommended defaults (configurable):
  MAX_UPLOAD_RATE:      16 KB/s   // 50% of typical tunnel capacity
  MAX_DOWNLOAD_RATE:    24 KB/s   // 75% — downloads user-facing, prioritized
  MAX_REPLICATION_RATE: 6 KB/s    // Background only; below v1.0's 8 KB/s (too aggressive)
  MAX_DHT_RATE:         2 KB/s    // DHT traffic is small but high-frequency
```

Bandwidth quotas are enforced using a token bucket per traffic class. The token bucket refill rate equals the configured quota; a bucket depth of 3× the per-message size allows small bursts.

### 16.3 Priority Queues

The outbound message scheduler uses a strict priority queue:

| Priority | Class | Message types | Notes |
|----------|-------|--------------|-------|
| P0 — Critical | Handshake | Noise handshake messages, SESSION_END, RATCHET | Never queued; immediate |
| P1 — Interactive | Retrieval | WANT_BLOCK, SEND_BLOCK, ACK for active consumer sessions | Low-latency user-facing |
| P2 — Normal | Serving | SEND_BLOCK responses to provider requests from other nodes | Best-effort serving |
| P3 — Background | DHT | DHT RPCs, FIND_PROVIDERS, PROVIDER announcements | Non-urgent |
| P4 — Bulk | Replication | Background SEND_BLOCK for repair operations | Lowest priority |

P0 messages are always sent immediately regardless of bandwidth quotas. P1–P4 messages are subject to per-class token buckets. P4 (replication) is paused entirely when tunnel rebuild rate exceeds 2/minute (congestion indicator).

### 16.4 Congestion Detection

I2P provides no explicit congestion signals. I2PFS infers congestion from observable proxy metrics:

| Signal | Threshold | Response |
|--------|-----------|----------|
| RTT increase | > 1.5× baseline for > 60 s | Reduce WINDOW_SIZE by 1; reduce MAX_REPLICATION_RATE by 25% |
| RPC timeout rate | > 20% of RPCs timing out | Halve active outbound connections; pause P4 traffic |
| Tunnel rebuild rate | > 2 rebuilds/minute | Pause P4 traffic entirely; reduce P3 by 50% |
| Sequential DONT_HAVE | > 3 from same provider | Deprioritize provider; escalate to next in timeout chain |

Congestion responses are reversed (relaxed) after the triggering metric returns to normal for 120 seconds continuously.

---

## 17. Security Threat Model

### 17.1 Adversary Model

I2PFS is designed to provide security against the following adversary capabilities:

- **Global passive adversary (GPA):** Can observe all I2P network traffic metadata (timing, sizes, source/destination at the I2P overlay level) but cannot decrypt I2P garlic-encrypted message content.
- **Active network adversary:** Can inject, replay, delay, or selectively drop I2P messages. Cannot break ChaCha20-Poly1305 AEAD.
- **Sybil adversary:** Can operate up to 10% of DHT nodes. Cannot generate I2P Destinations with specific DHT IDs without solving a computational proof-of-work.
- **Malicious DHT node:** A DHT participant who stores false records, returns incorrect routing information, or attempts to correlate queries with identities.
- **Malicious provider:** A content provider who receives block requests and attempts to fingerprint or deanonymize the consumer requesting blocks.
- **Malicious consumer:** A consumer who makes block requests and attempts to identify the publisher or other consumers of the same content.
- **Colluding nodes:** Up to K nodes may collude. K is assumed ≤ 20% of the network.

**Out of scope (accepted risks):**

- An adversary who controls the user's machine (software, OS, hardware) — this breaks any cryptographic system
- Long-term statistical traffic analysis requiring months of observation of a single user's behavior (acceptable for a non-real-time bulk storage system)
- Physical seizure and forensic analysis of a running node (mitigated by in-memory key handling but not fully preventable)
- Quantum adversaries (post-quantum migration path defined in Section 17.4)

### 17.2 Attack Mitigations

#### Content Integrity Attacks

| Attack | How it works | Mitigation | Residual risk |
|--------|-------------|------------|---------------|
| Block substitution | Provider sends a different block for a given ACI | BLAKE3 verification before accept: `BLAKE3(received) == ACI.content_hash` | None — invalid blocks are always detected |
| Chunk position swap | Shuffle encrypted chunks within a file | Per-chunk AAD binds chunk to `(root_aci, chunk_index)`: AEAD tag fails for wrong position | None |
| Partial delivery | Provider sends fewer chunks than claimed | Root block commit includes all leaf ACIs via BLAKE3 tree: missing chunks detected | Consumer must retry; denial-of-service possible if all providers are malicious |
| DHT PROVIDER record poisoning | Sybil node publishes false PROVIDER records | Records are signed with DHT Signing Key; nodes without valid signatures are rejected and their fail_count incremented | Sybil nodes can suppress content discovery if > 50% of nearest-k nodes are adversarial |
| ACI forgery | Publish a different block under the same ACI | BLAKE3 is collision-resistant; second-preimage resistance means forged ACI binding is infeasible | None |
| Root commitment attack | Modify the chunk table in the root block | Root block's content_hash is BLAKE3(ciphertext) of root block, verifiable via ACI | None |
| Rollback attack on MPs | Replay an old MP_VALUE to revert content | Monotonic seq field; nodes reject updates with seq ≤ stored seq | Network partition: if new update never reaches all DHT nodes, old seq persists there |
| Chosen-ciphertext attack | Modify encrypted block, observe decryption behavior | ChaCha20-Poly1305 rejects modified ciphertext; no decryption oracle exposed | None |

#### Identity Anonymity Attacks

| Attack | How it works | Mitigation | Residual risk |
|--------|-------------|------------|---------------|
| ACI preimage to find publisher | Given ACI, search for matching content in DHT to identify publisher | ACI commits to ciphertext, not plaintext; without ReadCap no preimage search is possible | If ReadCap leaks, content is attributable but publisher still not identified |
| DHT insert timing correlation | Observe when PROVIDER records appear; correlate with when a user connected | Random jitter 0–300 s on publication; pre-warmed tunnel pools remove connection-to-publish timing | Statistical correlation over many publications is possible for a GPA |
| Block request correlation across sessions | Provider observes same consumer identity across multiple sessions | Per-retrieval Client SAM session rotation; per-session ephemeral IDs | Single-session: provider can log that one (unknown) client requested blocks B1..BN in one session |
| PROVIDER record → node identity | DHT record reveals which node is serving a block | Blinded provider tokens in PROVIDER records; full identity requires additional NetDB resolution step | Multi-step correlation attack requires both DHT and NetDB observation |
| Long-term routing table correlation | Observing the same DHT_ID over time to track a node | DHT_ID rotation every 48 h; independent of I2P Destination | Long-term adversary can still correlate by routing behavior pattern |
| Want-list broadcast deanonymization | IPFS-style broadcast want-lists reveal consumer interests to many peers | No broadcast; all WANT_BLOCK messages sent unicast to DHT-discovered providers only | Provider learns which specific blocks are requested in this session |

### 17.3 Cryptographic Agility

All wire-format objects include version fields that enable future cryptographic algorithm migration. The upgrade path is:

1. **Proposal phase:** New algorithm identifiers are registered in an I2PFS Crypto Registry (published on I2P eepSites). New algorithms require at least 6 months of review before production activation.
2. **Dual-support phase:** A new minor version MUST support both old and new algorithms. Nodes MUST store and serve blocks under both old-version and new-version ACIs if they exist.
3. **Deprecation phase:** Old algorithm support is removed after at least 12 months and when < 5% of the network still uses the old version.

Nodes MUST support all minor versions within their major version. A major version bump indicates a breaking protocol change; old-major-version nodes will not be able to exchange content with new-major-version nodes.

### 17.4 Post-Quantum Migration Path

The current primitive set (X25519, Ed25519, ChaCha20-Poly1305) is not quantum-resistant. When quantum-safe algorithms are sufficiently standardized and performant, I2PFS SHOULD migrate:

| Current | Future replacement | Status |
|---------|------------------|--------|
| X25519 key agreement | ML-KEM (CRYSTALS-Kyber, FIPS 203) | Standardized; awaiting I2P adoption |
| Ed25519 signatures | ML-DSA (CRYSTALS-Dilithium, FIPS 204) | Standardized; larger keys |
| ChaCha20-Poly1305 | No change needed | Symmetric ciphers require 256-bit keys for quantum resistance; ChaCha20-256 already provides this |

The hybrid approach (X25519 + ML-KEM combined key agreement) should be used during the transition period to maintain security against both classical and quantum adversaries simultaneously.

---

## 18. Interoperability and Migration

### 18.1 Migration from v1.0

v1.0 and v2.0 ACIs are incompatible. v1.0 ACIs are 68 bytes; v2.0 ACIs are 38 bytes. The version byte (`0x01` vs `0x02`) distinguishes them. A v2.0 node encountering a v1.0 ACI SHOULD:

1. Attempt a DHT lookup using the v1.0 DHT key derivation (documented in the v1.0 spec)
2. If a provider is found, retrieve the v1.0 block, verify it, and re-publish it under a v2.0 ACI
3. Return both the v1.0 ACI (for legacy compatibility) and the v2.0 ACI (for v2.0 operations)

v1.0 blocks stored on providers are valid content and SHOULD be migrated to v2.0 format when served, if the migration node has access to the necessary key material (typically the publisher must re-publish).

### 18.2 IPFS Content Bridge

I2PFS can operate an optional **bridge mode** that fetches content from IPFS (via Tor or a pre-configured trusted clearnet gateway) and re-publishes it to I2PFS:

1. Fetch IPFS CID via gateway (outside I2PFS protocol)
2. Verify content against IPFS CID (multihash verification)
3. Run `PUBLISH(fetched_content, recipients=[])` to produce a v2.0 root ACI
4. The I2PFS ACI will differ from the original IPFS CID — this is by design

This process is explicitly a content-bridging operation, not a CID translation. The relationship between an IPFS CID and its I2PFS ACI is maintained only in the bridging node's local mapping table, not in any network-visible record.

### 18.3 SAM Bridge Requirements

I2PFS v2.0 requires I2P's SAM (Simple Anonymous Messaging) bridge v3.3 or later.

**Required SAM capabilities:**

```
STREAM CONNECT                          — outbound TCP-like connections
STREAM ACCEPT                           — inbound connection acceptance
DATAGRAM SEND / RECV (repliable)        — for DHT UDP-like RPCs
DEST GENERATE style=SIGNATURE_TYPE=7    — Ed25519 destination generation
DEST GENERATE style=SIGNATURE_TYPE=7,   — with integrated encryption keys
             crypto=ECIES-X25519-AEAD-Ratchet
LOOKUP name=<b64dest>                   — NetDB resolution
SESSION CREATE STYLE=STREAM             — streaming session
SESSION CREATE STYLE=DATAGRAM           — datagram session
SESSION SET DESTINATION=TRANSIENT       — per-retrieval ephemeral destinations
BLINDING GENERATE                       — blinded Lease Set keys
```

**Three separate SAM sessions MUST be created at startup:**

```
Session 1 (DHT): 
    SESSION CREATE STYLE=DATAGRAM DESTINATION=TRANSIENT
    ID=I2PFS-DHT-[random8] ...

Session 2 (Provider):
    SESSION CREATE STYLE=STREAM DESTINATION=TRANSIENT
    ID=I2PFS-PROVIDER-[random8] ...

Session 3 (Client):
    SESSION CREATE STYLE=STREAM DESTINATION=TRANSIENT
    ID=I2PFS-CLIENT-[random8] ...
    // Rotated per retrieval via SESSION REMOVE + SESSION CREATE
```

### 18.4 Configuration Reference

```toml
[node]
dht_id_rotation_hours        = 48
lease_set_rotation_minutes   = 5
provider_session_rotation_hours = 24
provider_session_max_connections = 1000

[dht]
k_bucket_size     = 20
alpha_initial     = 1
alpha_max         = 3
rpc_timeout_s     = 30
lookup_timeout_s  = 120
min_providers     = 3
refresh_interval_minutes = 30
republish_hours   = 20

[storage]
chunk_size_bytes         = 32768
replication_factor       = 5
replication_min          = 3
provider_ttl_hours       = 24
provider_reannounce_hours = 20
gc_unpinned_after_days   = 7
block_store_backend      = "rocksdb"

[bandwidth]
max_upload_kbps         = 16
max_download_kbps       = 24
max_replication_kbps    = 6
max_dht_kbps            = 2
pipeline_window_max     = 6
garlic_batch_window_ms  = 50
garlic_batch_max_bytes  = 49152   # 48 KB

[crypto]
noise_pattern   = "IKpsk2"
aead            = "chacha20-poly1305"
hash            = "blake3"
kdf             = "hkdf-sha256"
argon2id_t      = 3
argon2id_m      = 65536
argon2id_p      = 4
ratchet_interval_messages = 1000

[tunnels]
outbound_pool_size          = 3
inbound_pool_size           = 3
rebuild_before_expiry_minutes = 2
rebuild_jitter_seconds      = 30

[sam]
host    = "127.0.0.1"
port    = 7656
version = "3.3"
```

---

## 19. Implementation Recommendations

### 19.1 Language and Runtime

| Language | Assessment | Notes |
|----------|-----------|-------|
| **Rust** | **Strongly recommended** | Memory safety without GC pauses; excellent async runtime (tokio); existing SAM bridge crates (`i2p-rs`, `ire`); best performance for crypto operations |
| Go | Acceptable | Good concurrency primitives; adequate crypto libraries; some GC pause risk during streaming |
| Java/Kotlin | Possible | Native I2P is JVM-based; enables tighter I2P integration; GC pauses are a concern for timing-sensitive anonymity operations |
| Python | Not recommended | GC and GIL make it unsuitable for high-throughput crypto; acceptable for prototyping only |

**Critical implementation requirement:** All cryptographic operations (AEAD, key derivation, handshakes) MUST use constant-time implementations. Timing side-channels in crypto code can leak key material. Use audited libraries (libsodium, ring, BoringSSL) rather than custom implementations.

### 19.2 Storage Backend

| Backend | Recommendation | Notes |
|---------|---------------|-------|
| **RocksDB** | Strongly recommended | LSM-tree; high write throughput; compaction handles block churn well; supports prefix scans for DHT bucket queries |
| SQLite | Acceptable for small nodes (< 10 GB) | Simpler deployment; adequate performance under light load; WAL mode required |
| Filesystem (content-addressed) | Acceptable for proof-of-concept | Simple; benefits from OS page cache; degrades with > 100,000 blocks due to directory scalability |

**Mandatory separation:** Block data and DHT routing table MUST be stored in separate databases/directories to allow independent backup, compaction, and emergency clearing.

**At-rest encryption:** The Block Store MUST encrypt stored blocks using a node-local key (derived from the Master Identity Key). This ensures that physical seizure of storage media does not expose content to adversaries without the node's key material.

**Key material storage:** The Master Identity Key MUST be stored encrypted with Argon2id(passphrase) on disk. The passphrase is never stored. The decrypted key is held in memory only. On graceful shutdown, memory containing key material MUST be explicitly zeroed.

### 19.3 Concurrency Architecture

I2PFS requires careful concurrency design to achieve its performance targets without saturating I2P tunnel capacity:

```
Recommended actor/task model (Rust/tokio):

┌─────────────────────────────────────────────────────────────┐
│  Main Runtime                                               │
│                                                             │
│  DHT Actor (1 task)                                         │
│    ├── Routing table maintenance (periodic)                 │
│    ├── RPC dispatch (per RPC, spawned)                      │
│    └── PROVIDER record maintenance (periodic)              │
│                                                             │
│  AnonSwap Actor (1 task per active session, max 10)         │
│    ├── Noise handshake state machine                        │
│    ├── Block send/receive pipeline                          │
│    └── Ratchet management                                   │
│                                                             │
│  Replication Actor (1 task)                                 │
│    ├── Repair detection (periodic)                          │
│    └── Push replication (per repair, spawned, P4 priority)  │
│                                                             │
│  SAM Bridge Actor (1 task per SAM session = 3 tasks)        │
│    ├── Inbound connection handling                          │
│    └── Outbound message queue (priority-ordered)            │
└─────────────────────────────────────────────────────────────┘
```

Maximum concurrent AnonSwap sessions should be capped at 10 to prevent tunnel pool exhaustion. Each session consumes at minimum 1 outbound + 1 inbound tunnel slot.

### 19.4 Mandatory Test Vectors

All implementations MUST pass the following test vectors before production deployment:

1. **ACI v2 Computation:** Given `ciphertext_bytes`, verify `ACI.content_hash == BLAKE3(ciphertext_bytes)` matches reference.
2. **Per-chunk key derivation:** Given `content_root_key` and `chunk_index`, verify `chunk_key` and `chunk_nonce` match reference values.
3. **ReadCap wrapping/unwrapping:** Given recipient keypair and plaintext ReadCap, verify round-trip encrypt/decrypt produces identical ReadCap.
4. **Root block commitment:** Given `leaf_acis[]` and `blind_factor`, verify root commitment matches reference.
5. **DHT key derivation:** Given ACI and epoch number, verify DHT_key matches reference.
6. **Noise_IKpsk2 handshake:** Interoperation test with reference implementation at known PSK and key inputs.
7. **PROVIDER record signing/verification:** Given DHT Signing Key and record fields, verify signature; verify rejection of invalid signatures.
8. **Mutable Pointer epoch blinding:** Given `mp_master_privkey` and epoch, verify MP_DHT_key and epoch subkey derivation match reference.
9. **Symmetric ratchet:** Given session keys at message 1000, verify post-ratchet keys match reference.
10. **Block bitmap checkpoint:** Write checkpoint, simulate crash, resume from checkpoint, verify only missing blocks are re-fetched.

Test vectors are published separately as `I2PFS-testvectors-v2.json` on the I2P eepSite.

### 19.5 Security Audit Requirements

Before any production deployment, implementations MUST undergo the following review steps:

**Cryptographic review:**
- External review of Noise_IKpsk2 handshake implementation, including PSK derivation and prologue
- Audit of per-chunk key and nonce derivation for uniqueness guarantees
- Review of ReadCap wrapping for IND-CCA2 security properties
- Verification of constant-time execution for all X25519, Ed25519, and AEAD operations

**Protocol review:**
- Formal review of the session anonymity model (session ID freshness, Client SAM rotation)
- Analysis of the PROVIDER token blinding scheme for unlinkability
- Review of MP epoch blinding for long-term tracking resistance

**Implementation review:**
- Fuzzing of all binary deserialization paths (root DAG block, leaf chunk block, AnonSwap frames, DHT records)
- Memory safety audit for all key material handling and zeroing on shutdown
- Review of CSPRNG seeding and entropy pool initialization

---

## 20. Appendices

### Appendix A: Comparison with IPFS and I2PFS v1.0

| Aspect | IPFS | I2PFS v1.0 | I2PFS v2.0 |
|--------|------|-----------|-----------|
| Network | Clearnet (+ overlays) | I2P-native | I2P-native |
| Content ID | CID (public multihash) | ACI v1 (blinded, but blind factor in ACI) | ACI v2 (hash of ciphertext; no secret in ACI) |
| Retrieval capability | CID sufficient | ACI + blind_factor (but both in ACI) | ACI + ReadCap (separate; both required) |
| Encryption | Optional | Mandatory; but double-layer overhead | Mandatory; single per-chunk AEAD layer |
| Encryption granularity | Varies | Entire file before chunking (requires full file in memory) | Per-chunk (streaming-capable) |
| Session protocol | libp2p (Noise_XX + Secio) | Noise_XX with DHT identity keys | Noise_IKpsk2 with purpose-built session keys |
| Post-compromise security | No | No | Yes (symmetric ratchet every 1000 messages) |
| Message encoding | Protobuf | CBOR (variable-length) | Fixed binary frames |
| DHT_ID linkage to I2P | N/A | Derived from I2P Destination (privacy flaw) | Independently generated (no linkage) |
| PROVIDER records | Contain peer IDs | Contain full I2P Destination (privacy flaw) | Contain blinded tokens only |
| Resumable downloads | Limited (IPFS pin) | Not specified | Block bitmap checkpointing |
| Mutable pointers | IPNS (public key visible) | MP (pubkey visible in DHT) | MP v2 (epoch-blinded pubkey; only master pubkey stable) |
| DHT timeout | 1–5 s | 30 s | 30 s |
| Replication | Passive announce | Proactive push (R=5) | Proactive push (R=5) with diversity scoring |
| Block size | 256 KB default | 32 KB | 32 KB |
| Pipeline window | Aggressive | 8 (too large for I2P) | 2–6 (adaptive, bandwidth-calibrated) |

### Appendix B: Protocol Error Codes

All SESSION_END messages, DONT_HAVE responses, and error conditions use a standardized 1-byte error code:

| Code | Name | Meaning |
|------|------|---------|
| `0x00` | NORMAL | Clean session end; retrieval complete |
| `0x01` | PROTOCOL_ERROR | Received a malformed or unexpected frame |
| `0x02` | PROVIDER_FULL | Provider's Block Store is at capacity |
| `0x03` | BLOCK_NOT_FOUND | Requested block is not stored here |
| `0x04` | SIGNATURE_INVALID | Received a DHT record with invalid signature |
| `0x05` | ACI_MISMATCH | Block data hash does not match the requested ACI |
| `0x06` | HANDSHAKE_FAILED | Noise handshake could not be completed |
| `0x07` | PSK_MISMATCH | Pre-shared key verification failed |
| `0x08` | RATE_LIMITED | Consumer is sending requests too fast; backoff required |
| `0x09` | EPOCH_EXPIRED | Request used an expired epoch DHT key |
| `0x0A` | SESSION_TIMEOUT | No message received for 300 s |

### Appendix C: Wire Format Summary

```
Object                     Size            Notes
─────────────────────────────────────────────────────────────────
ACI v2                     38 bytes        Fixed; version byte = 0x02
ReadCap                    76 bytes        Fixed; never transmitted in plaintext
AnonSwap Frame Header      29 bytes        Fixed; + variable payload
Noise_IKpsk2 Message 1     ~128 bytes      Initiator → Responder
Noise_IKpsk2 Message 2     ~64 bytes       Responder → Initiator
WANT_BLOCK payload         42 bytes        ACI(38) + max_size(4)
SEND_BLOCK payload         ~32,826 bytes   ACI(38) + chunk_index(4) + data(32768) + tag(16)
DONT_HAVE payload          38 bytes        ACI only
ACK payload                42 bytes        ACI(38) + chunk_index(4)
DHT PROVIDER record        142 bytes       Fixed binary
DHT VALUE record           Variable        ≤ 1,066 bytes (fixed header + ≤1000B value)
MP_VALUE record            183 bytes       Fixed binary
Root DAG Block (1 MB file) ~1,300 bytes    28B header + recipient_header + 32×38B chunk table
Leaf Chunk Block           ~32,817 bytes   13B header + 32,768B ciphertext + 16B tag + 12B nonce

Recipient header per entry: 136 bytes      ek_pub(32) + wrap_nonce(12) + wrapped_readcap(92)
```

### Appendix D: Glossary

| Term | Definition |
|------|-----------|
| **ACI** | Anonymous Content Identifier — I2PFS v2.0's 38-byte content address; commits to ciphertext; contains no secret material |
| **AnonSwap** | I2PFS's binary block exchange protocol; replaces v1.0 AnonBitSwap |
| **Blinded Provider Token** | A 64-byte DHT-stored value that allows consumers to contact a provider without the DHT record itself revealing the provider's I2P identity |
| **Blind Factor** | 32-byte secret value stored in the ReadCap; part of the root block integrity verification; prevents publisher identification via plaintext preimage search |
| **Content Root Key** | 32-byte secret key stored in the ReadCap; used to derive all per-chunk encryption keys |
| **DHT_key** | Epoch-rotated 32-byte key under which PROVIDER records are stored in the DHT; derived from but distinct from the ACI |
| **Garlic Message** | I2P's bundled encrypted message containing multiple "cloves" (sub-messages) delivered in a single tunnel traversal |
| **I2P-DHT** | I2PFS's modified Kademlia DHT adapted for I2P's high-latency, high-churn environment |
| **Lease Set** | I2P data structure advertised in the NetDB listing a node's inbound tunnel endpoints; used by other nodes to reach the destination |
| **MP** | Mutable Pointer — a signed DHT VALUE record mapping a stable public key to a current root ACI; enables versioned/updateable content |
| **Noise_IKpsk2** | Noise Protocol Framework handshake pattern; provides 1-RTT session establishment, responder static key privacy, and PSK-based capability check |
| **Provider** | An I2PFS node that stores and serves blocks for one or more ACIs |
| **ReadCap** | Read Capability — 76-byte secret structure containing the `content_root_key` and `blind_factor` required to decrypt and fully verify a content object; distributed only via the encrypted recipient header |
| **Ratchet** | Symmetric key advancement mechanism; after 1000 messages, fresh random material is mixed into the session keys, providing post-compromise security |
| **SAM** | Simple Anonymous Messaging — I2P's application-layer API for building I2P-enabled applications without requiring I2CP access |
| **Session ID** | 16-byte random value identifying an AnonSwap session; ephemeral, never reused across different retrievals |
| **Tunnel** | I2P's unidirectional encrypted relay path through 2–3 intermediate routers; the fundamental I2P transport primitive |

### Appendix E: Reference Implementation

None

*End of I2PFS Technical Specification v1.0*

*For questions, corrections, or contributions: I2P forum thread [to be announced] or the I2P Git repository above.*
