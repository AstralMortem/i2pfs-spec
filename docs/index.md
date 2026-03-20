# I2PFS — Anonymous Distributed File System

**Technical Specification v1.0 · March 2026**

---

!!! abstract "Status"
    This document is the normative specification for the I2PFS protocol. It is intended for implementors, security researchers, and contributors.

    **Network target:** I2P (Invisible Internet Project)  
    **Design principle:** Anonymity and I2P operational constraints are first-class invariants, not afterthoughts.

---

## Abstract

**I2PFS** (Invisible Internet Project File System) is a decentralized, content-addressed distributed file system designed exclusively for the I2P anonymous overlay network. It provides computationally unlinkable sender and recipient anonymity, per-chunk end-to-end encryption with forward secrecy, fault-tolerant distributed storage, and a block exchange protocol optimized for I2P's high-latency, bandwidth-constrained unidirectional tunnel architecture.

I2PFS is designed from cryptographic and network primitives upward, with anonymity and I2P's operational envelope as first-class constraints. Every protocol design decision is evaluated against I2P's 500–2000 ms round-trip times, 10-minute tunnel lifetimes, 8–32 KB/s effective bandwidth, and the fundamental requirement that no operation reveals publisher or consumer identity to any observer — including other I2PFS nodes.

---

## Core Guarantees

| Guarantee | Description |
|-----------|-------------|
| **Complete anonymity** | Neither publishers nor consumers can be identified by any network-level or protocol-level observer |
| **Capability-based access** | Retrieving content requires both an ACI and a ReadCap; neither alone is sufficient |
| **Per-chunk encryption** | All data is encrypted per-chunk before leaving the originating node; no plaintext traverses the network |
| **Decentralization** | No central servers, authorities, or trust anchors exist in the system |
| **Fault tolerance** | Content survives significant node churn through proactive adaptive replication |
| **High-latency operation** | All protocol operations succeed under 200–2000 ms RTTs using asynchronous, pipelined designs |
| **Forward secrecy** | Compromise of long-term keys does not expose past data |
| **Post-compromise security** | A lightweight symmetric ratchet limits the blast radius of session key compromise |

---

## Non-Goals

- Real-time media streaming (I2P latency variance makes this infeasible)
- Sub-10-second file retrieval (incompatible with I2P tunnel architecture; expect 35–120 s)
- Clearnet accessibility (I2PFS is strictly an I2P-internal protocol)
- Compatibility with IPFS wire protocol (incompatible by design)
- Payment channels or incentive mechanisms (out of scope; nodes participate voluntarily)

---

## Protocol Stack

```
┌──────────────────────────────────────────────────────────────┐
│  L7: Content API                                             │
│      Put(file) → RootACI + ReadCap                           │
│      Get(RootACI, ReadCap) → file                            │
│      Pin / Unpin / List / Resolve                            │
├──────────────────────────────────────────────────────────────┤
│  L6: Mutable Pointer Layer                                   │
│      MP resolution: stable_key → current RootACI            │
│      Blinded update protocol                                 │
├──────────────────────────────────────────────────────────────┤
│  L5: AnonSwap (Block Exchange Protocol)                      │
│      Fixed-binary-frame block request/response               │
│      Ephemeral session IDs; no persistent peer identity      │
│      Resumable download checkpointing                        │
├──────────────────────────────────────────────────────────────┤
│  L4: I2PFS-DHT (Content Routing)                             │
│      Modified Kademlia; 30 s RPC timeout; k=20               │
│      Blinded provider tokens; signed records                 │
├──────────────────────────────────────────────────────────────┤
│  L3: Session Encryption (Noise_IKpsk2 + Ratchet)             │
│      Mutual authentication; forward secrecy                  │
│      Symmetric ratchet for post-compromise security          │
├──────────────────────────────────────────────────────────────┤
│  L2: I2P Streaming / Datagram Transport (SAM v3.3)           │
│      Reliable delivery; unidirectional tunnel abstraction    │
├──────────────────────────────────────────────────────────────┤
│  L1: I2P Garlic Routing (provided by I2P router)             │
│      [Out of scope — operated by local I2P router process]   │
└──────────────────────────────────────────────────────────────┘
```

---

## How to Read This Specification

This specification is organized into logical layers. New readers should start with:

1. [I2P Network Constraints](network-constraints.md) — understand why every design decision exists
2. [System Architecture](architecture.md) — the node model and session separation
3. [ACI & Capability Model](aci-capability.md) — the two-object access model
4. [Publish & Retrieve Flows](flows.md) — end-to-end worked examples

Protocol implementors should additionally study:

- [Cryptographic Foundations](cryptography.md) and [End-to-End Encryption](encryption.md) for the complete crypto layer
- [Types, Constants & Wire Formats](types-constants.md) for all binary formats, constants, and error codes

---

## Reference Implementation

```
I2P eepSite:    http://i2pfs.i2p/
I2P Git:        http://git.idk.i2p/i2pfs/i2pfs-core.git
```

Written in Rust, licensed MIT. Key dependencies: `tokio`, `snow` (Noise), `blake3`, `chacha20poly1305`, `x25519-dalek`, `ed25519-dalek`, `rocksdb`.
