# Node Identity & Anonymity

---

## Three-Layer Identity Separation

I2PFS strictly separates identity concerns into three independent layers. **No layer is derivable from or linkable to another** by an external observer.

```
Layer 1: I2P Transport Identity
  └── Three separate I2P Destinations (one per SAM session)
  └── DHT Destination rotates every 48 h
  └── Provider Destination rotates every 24 h / 1,000 connections
  └── Client Destination rotates per-retrieval
  └── Used ONLY for I2P packet routing; never appears in I2PFS protocol messages

Layer 2: DHT Routing Identity
  └── DHT_ID — derived from an independent dht_seed; no relation to I2P Destinations
  └── Rotates every 48 h
  └── Used ONLY for DHT routing table participation
  └── DHT Signing Key is epoch-specific and rotates with DHT_ID

Layer 3: Content Identity
  └── Per-publish content_root_key — random; never reused
  └── Per-publish blind_factor — random; stored only in ReadCap; never in ACI
  └── Content signing keys are ephemeral and single-use
  └── Never linked to any node's DHT_ID or I2P Destination
```

**The invariant:** An adversary who compromises one layer gains zero information about the other layers.

---

## Key Hierarchy

The Master Identity Key is a node's long-term identity and **MUST NEVER appear on the wire** in any form, directly or derivably.

```
Master Identity Key (Ed25519, stored Argon2id-encrypted on disk)
│
├── DHT Signing Key          [epoch-derived; rotates every 48 h]
│   └── Signs DHT records; NEVER used for encryption
│
├── Provider Static Key      [derived; rotates every 24 h or 1,000 sessions]
│   └── Noise_IKpsk2 responder static key for AnonSwap sessions
│   └── Advertised via Blinded Provider Token
│
├── Mutable Pointer Key      [derived; rotated per named content channel]
│   └── Ed25519 key for signing MP_VALUE records
│
└── Content Read Key         [derived per-publish from random material]
    └── Stored in ReadCap; used as root of per-chunk key derivation
```

### Subkey Derivation

All subkeys are derived via HKDF-SHA256 with mandatory domain separation:

```
SubKey := HKDF-SHA256(
    ikm  = master_secret_bytes,
    salt = "I2PFS-" || subkey_label,
    info = epoch_bytes || node_id_commitment,
    len  = 32
)
```

`node_id_commitment` is a 32-byte value committed at node creation time, stored alongside the master key, providing binding without exposing the master key.

---

## DHT Identity Generation

!!! warning "Critical design point"
    DHT IDs are generated **independently** from I2P Destinations. Never derive a DHT_ID from an I2P Destination — doing so creates a mathematical link between routing identity and transport identity.

```python
# At first launch:
dht_seed := CSPRNG(32 bytes)   # stored encrypted on disk; never transmitted
DHT_ID[epoch=0] := BLAKE3(dht_seed || "I2PFS-DHT-epoch" || uint64_le(0))

# At each epoch change (every 48 hours):
DHT_ID[epoch=N] := BLAKE3(dht_seed || "I2PFS-DHT-epoch" || uint64_le(N))
```

**Old ID validity:** `DHT_ID[N-1]` remains valid for one full epoch after rotation to epoch N, allowing graceful migration. Nodes MUST accept RPCs addressed to either their current or previous DHT_ID.

---

## Encrypted Lease Sets

I2P Lease Sets published to the NetDB advertise a node's inbound tunnels and are observable by I2P floodfill nodes. I2PFS MUST use I2P **Type 3 Encrypted (Blinded) Lease Sets** for all SAM sessions:

- The Lease Set is encrypted such that only nodes in possession of a blinding factor can decode it
- The blinding factor for the Provider SAM Session is distributed only to nodes that have completed a valid AnonSwap handshake (proved they know the content's DHT key via the PSK)
- Lease Sets are rotated every **5 minutes** (vs. I2P's default 10 minutes) to limit traffic correlation windows

---

## Threat Mitigation Summary

| Attack Vector | Mitigation | Residual Risk |
|---|---|---|
| Passive DHT observation | DHT_IDs rotate every 48 h; all DHT traffic encrypted by I2P garlic routing | Long-term statistical correlation over months (accepted) |
| Publisher ID via ACI preimage | ACI hashes ciphertext; ReadCap contains blind_factor which is secret | None if ReadCap is kept secret |
| Publisher ID via DHT insert timing | Pre-warmed tunnel pools; random jitter 0–300 s on publication | Timing correlation over many publications |
| Consumer ID via block requests | Per-retrieval Client SAM rotation; per-session ephemeral IDs | Single-session: provider sees one anonymous session |
| Consumer ID via Lease Set | Encrypted Lease Sets; 5-minute rotation; blinded keys | Lease Set metadata observable by I2P floodfill nodes |
| PROVIDER record identity linkage | Blinded provider tokens; requires additional NetDB lookup step to resolve | Multi-step correlation attack requiring DHT + NetDB observation |
| Replica selection fingerprinting | ±15% random score jitter in `SELECT_REPLICAS` | Probabilistic inference with large observation set |
| Sybil DHT poisoning | Signed records; fail_count eviction; PoW cost of I2P Destination creation | >10% Sybil ratio degrades routing quality |
| Eclipse attack | Routing table diversity requirements (address prefix + XOR distance) | Sustained targeted attack with many Sybil nodes |
