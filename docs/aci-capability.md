# ACI & Capability Model

---

## Two-Object Access Model

Content retrieval in I2PFS requires **two distinct objects**. Neither alone is sufficient:

| Object | Size | Visibility | Purpose |
|--------|------|-----------|---------|
| **ACI** (Anonymous Content Identifier) | 38 bytes | Public, shareable | Uniquely identifies a piece of content; used for DHT lookup and block verification |
| **ReadCap** (Read Capability) | 76 bytes | Secret | Contains the keys required to decrypt and verify the content |

An adversary who obtains only the ACI can locate and download the ciphertext but **cannot decrypt it**. An adversary who obtains only the ReadCap has decryption keys but **cannot locate the data** without the ACI. Both are required.

---

## ACI Binary Structure

The ACI is a **38-byte fixed-width** binary value:

```
Byte offset:  0    1    2    3    4    5   ...  37
              ├────┬─────────┬────┬─────────────────┤
              │Ver │ Codec   │Hash│  Content Hash   │
              │ 1B │   2B    │ 1B │     32 bytes    │
              └────┴─────────┴────┴─────────────────┘
```

| Field | Offset | Size | Values |
|-------|--------|------|--------|
| `version` | 0 | 1 byte | `0x01` |
| `codec` | 1 | 2 bytes | `0x0000` = raw, `0x0001` = i2pfs-dag |
| `hash_algo` | 3 | 1 byte | `0x1E` = BLAKE3 |
| `content_hash` | 4 | 32 bytes | `BLAKE3(encrypted_root_block_payload)` |

!!! important "ACI hashes ciphertext, not plaintext"
    `content_hash` is the BLAKE3 hash of the **encrypted** root block payload. This means:

    - The ACI can be verified by any holder of the ciphertext without decryption
    - No secret material is needed to compute or verify the ACI
    - The ACI does NOT directly commit to the plaintext content — plaintext correctness is verified via the ReadCap

---

## ReadCap Binary Structure

The ReadCap is the secret capability required to decrypt content. It is a **76-byte fixed-width** binary structure:

```
Byte offset:  0                           31  32                          63  64         75
              ├──────────────────────────────┬───────────────────────────────┬─────────────┤
              │      content_root_key        │        blind_factor            │ version_tag │
              │         32 bytes             │          32 bytes              │  12 bytes   │
              └──────────────────────────────┴───────────────────────────────┴─────────────┘
```

| Field | Offset | Size | Description |
|-------|--------|------|-------------|
| `content_root_key` | 0 | 32 bytes | Root key for deriving all per-chunk encryption keys |
| `blind_factor` | 32 | 32 bytes | Secret value used in root block commitment verification |
| `version_tag` | 64 | 12 bytes | `"I2PFSReadCap1"` — format validation magic |

!!! danger "ReadCap is never transmitted in plaintext"
    The ReadCap is distributed to authorized consumers via the encrypted recipient header in the root block. It is wrapped with X25519 ECIES per recipient. Unauthorized nodes who receive the root block cannot extract the ReadCap.

---

## DHT Key Derivation

Content is located in the DHT using a key derived from the ACI and a time-based epoch:

```python
DHT_key := HKDF-SHA256(
    ikm  = ACI_bytes,           # 38 bytes; contains no secret material
    salt = "I2PFS-DHT-key",
    info = uint64_le(epoch),    # epoch = unix_time_seconds / 21600  (6-hour epochs)
    len  = 32
)
```

### Why Epoch Rotation?

A static ACI→DHT_key mapping allows long-term DHT observation to build a persistent content→location map. Epoch rotation means that observing DHT traffic in epoch N reveals no useful information for epoch N+1.

The 6-hour epoch provides four rotation points per day, balancing privacy against the cost of re-announcement.

**Provider re-announcement timing:** Providers MUST re-announce content in the DHT at the start of each new epoch while also re-announcing before their current-epoch TTL expires. This ensures continuous availability across epoch boundaries.

---

## Human-Readable Encoding

ACIs and ReadCaps use Base32 encoding (RFC 4648, no padding) for human display and out-of-band sharing:

```
ACI:     i2pfs:[BASE32(aci_bytes)]
ReadCap: i2pfsrc:[BASE32(readcap_bytes)]
```

For a combined retrieval link (ACI + ReadCap together):

```
i2pfs:[BASE32(aci_bytes)]#i2pfsrc:[BASE32(readcap_bytes)]
```

The `#` separator mirrors the URL fragment convention, where the fragment (ReadCap) is treated as local-only material that should never be logged, transmitted to servers, or included in access logs.

**Base32 rationale:** Case-insensitive; avoids visually ambiguous characters (0/O, 1/l/I); safe for copy-paste and QR codes.

---

## ACI Verification

Any node that holds an encrypted block can verify it against an ACI **without decryption**:

```python
def verify_block(block_data: bytes, block_tag: bytes, aci: ACI) -> bool:
    actual_hash = BLAKE3(block_data || block_tag)
    return actual_hash == aci.content_hash
```

This enables providers to verify integrity of blocks they store and serve without ever having the ReadCap. A tampered or corrupt block is always detected before it reaches a consumer.

---

## Root Block Commitment

The root ACI's `content_hash` commits to both the chunk table and the blind_factor:

```python
root_commitment := BLAKE3(
    leaf_aci[0] || leaf_aci[1] || ... || leaf_aci[N-1] || blind_factor
)
```

This means:

1. **Chunk table integrity:** The list of all leaf chunk ACIs cannot be modified without breaking the root commitment
2. **Publisher binding:** The `blind_factor` (secret to the publisher) is included in the commitment, preventing an adversary who has the plaintext from computing the same root ACI without the ReadCap
3. **Verified on decryption:** The consumer recomputes this commitment after extracting `blind_factor` from the ReadCap, providing final proof that the received root block is authentic
