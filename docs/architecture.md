# System Architecture

---

## Node Model

Every I2PFS node is **fully symmetric** — all nodes are protocol peers capable of fulfilling all roles simultaneously. Role specialization is a runtime performance optimization, never a protocol requirement. A node with modest storage and a residential I2P connection can provide all four functions.

### Node Roles

| Role | Function |
|------|----------|
| **Publisher** | Encrypts content per-chunk, computes root ACI, selects replicas, initiates push replication |
| **Provider** | Stores replicated encrypted blocks, responds to AnonSwap block requests, maintains PROVIDER records in DHT |
| **Router** | Participates in DHT routing, forwards RPCs; does not necessarily store content blocks |
| **Consumer** | Resolves ACIs via DHT, fetches blocks from providers, verifies and decrypts |

---

## Internal Node Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                          I2PFS Node                                │
│                                                                    │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────────────┐    │
│  │  Block Store  │  │   DHT Store   │  │   Routing Table     │    │
│  │  (per-chunk   │  │  (k-buckets)  │  │ (DHT IDs + SAM      │    │
│  │   encrypted)  │  │               │  │  Destinations)      │    │
│  └──────┬────────┘  └──────┬────────┘  └──────────┬──────────┘    │
│         │                  │                      │               │
│  ┌──────▼──────────────────▼──────────────────────▼────────────┐  │
│  │                  I2PFS Protocol Engine                      │  │
│  │                                                             │  │
│  │  ┌────────────┐  ┌──────────┐  ┌────────────┐  ┌────────┐  │  │
│  │  │  AnonSwap  │  │ I2PFS-   │  │    ACI     │  │Replica-│  │  │
│  │  │  Engine    │  │  DHT     │  │  Resolver  │  │  tor   │  │  │
│  │  └────────────┘  └──────────┘  └────────────┘  └────────┘  │  │
│  │                                                             │  │
│  │  ┌──────────────────────────────────────────────────────┐   │  │
│  │  │          Session Manager (Noise_IKpsk2 + Ratchet)    │   │  │
│  │  └──────────────────────────────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                               │                                    │
│  ┌────────────────────────────▼───────────────────────────────┐    │
│  │         I2P SAM Bridge Interface (three sessions)          │    │
│  │   DHT SAM Session │ Provider SAM Session │ Client SAM Sess │    │
│  └────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────┘
```

---

## SAM Session Separation

!!! danger "Mandatory"
    Three independent SAM sessions with three independent I2P Destinations are **required**. This is not optional. Collapsing sessions onto a single Destination links DHT identity to content serving to content retrieval, breaking the anonymity model.

| Session | I2P Destination | Purpose | Rotation |
|---------|----------------|---------|----------|
| **DHT Session** | Destination A | DHT RPC traffic only | Every 48 h (with DHT_ID rotation) |
| **Provider Session** | Destination B | Serving blocks to consumers | Every 24 h or 1,000 connections |
| **Client Session** | Destination C | Fetching content | Per retrieval session |

**Why this separation is necessary:**

- A compromised or observed DHT session reveals nothing about which content the node is serving or retrieving
- A compromised Provider session reveals which content is being served but not the node's DHT identity or retrieval history
- A compromised Client session reveals one retrieval in progress but nothing persistent

SAM session creation at startup:

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

---

## Startup Sequence

```
1. Load or generate Master Identity Key (decrypt from disk with Argon2id(passphrase))
2. Derive dht_seed (independent of Master Key; see Identity model)
3. Create three SAM sessions (DHT, Provider, Client)
4. Pre-warm tunnel pools for all three sessions
5. Load DHT routing table from disk; begin background refresh
6. Load Block Store index; resume any pending PROVIDER announcements
7. Begin accepting inbound AnonSwap connections on Provider session
8. Node is ready
```

---

## Storage Layout

```
{data_dir}/
├── keys/
│   ├── master.key.enc       (Master Identity Key, Argon2id-encrypted)
│   └── dht_seed.enc         (DHT seed, encrypted)
├── blocks/
│   └── {rocksdb_data}/      (Block Store — encrypted blocks, keyed by ACI)
├── dht/
│   └── {rocksdb_data}/      (DHT routing table + record cache)
├── checkpoints/
│   └── {aci_hex}.chk        (Resume checkpoints for interrupted downloads)
└── config.toml
```

Block data and DHT routing table **must** be stored in separate databases to allow independent compaction, backup, and emergency clearing.
