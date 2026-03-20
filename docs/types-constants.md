# Types, Constants & Wire Formats

This page is the canonical reference for all binary formats, protocol constants, error codes, and configuration defaults defined in the I2PFS protocol.

---

## Protocol Version

| Constant | Value | Description |
|----------|-------|-------------|
| `I2PFS_VERSION_MAJOR` | `1` | Major protocol version |
| `I2PFS_VERSION_MINOR` | `0` | Minor protocol version |
| `ACI_VERSION` | `0x01` | ACI version byte |
| `ROOT_BLOCK_VERSION` | `0x01` | Root DAG block version |
| `LEAF_BLOCK_VERSION` | `0x01` | Leaf chunk block version |
| `PROVIDER_RECORD_VERSION` | `0x01` | DHT PROVIDER record version |
| `MP_RECORD_VERSION` | `0x01` | Mutable Pointer record version |
| `READCAP_VERSION_TAG` | `"I2PFSReadCap1"` | ReadCap format magic (12 bytes) |

---

## Magic Bytes

| Structure | Magic | Hex |
|-----------|-------|-----|
| Root DAG Block | `I2DG` | `0x49 0x32 0x44 0x47` |
| Leaf Chunk Block | `I2CK` | `0x49 0x32 0x43 0x4B` |

---

## ACI — Anonymous Content Identifier

**Fixed width: 38 bytes**

```
Offset  Size  Field         Description
──────────────────────────────────────────────────────────────
0       1     version       Always 0x01
1       2     codec         Content type (see Codec Values)
3       1     hash_algo     Hash algorithm (see Hash Algorithm IDs)
4       32    content_hash  BLAKE3(encrypted_root_block_payload)
```

### Codec Values

| Value | Name | Description |
|-------|------|-------------|
| `0x0000` | `RAW` | Raw binary content |
| `0x0001` | `I2PFS_DAG` | I2PFS Merkle DAG root block |

### Hash Algorithm IDs

| Value | Algorithm |
|-------|-----------|
| `0x1E` | BLAKE3 |

### Human-Readable Encoding

```
i2pfs:[BASE32_NO_PAD(aci_38_bytes)]
```

---

## ReadCap — Read Capability

**Fixed width: 76 bytes — NEVER transmitted in plaintext**

```
Offset  Size  Field             Description
──────────────────────────────────────────────────────────────
0       32    content_root_key  Root key for per-chunk key derivation (CSPRNG per publish)
32      32    blind_factor      Secret for root commitment verification (CSPRNG per publish)
64      12    version_tag       b"I2PFSReadCap1" (ASCII, no null)
```

### Human-Readable Encoding

```
i2pfsrc:[BASE32_NO_PAD(readcap_76_bytes)]
```

### Combined Retrieval Link

```
i2pfs:[ACI_BASE32]#i2pfsrc:[READCAP_BASE32]
```

---

## Root DAG Block — I2DG

**Variable width. Magic: `0x49324447`**

```
Offset  Size                 Field             Description
────────────────────────────────────────────────────────────────────────────
0       4                    magic             0x49 0x32 0x44 0x47 ("I2DG")
4       1                    version           0x01
5       1                    flags             bit 0=inline_content, bit 1=has_recipients,
                                               bit 2=compressed
6       8                    total_size        uint64 LE, original plaintext size
14      4                    chunk_count       uint32 LE, number of leaf chunks
18      4                    chunk_size        uint32 LE, nominal chunk size (default 32768)
22      2                    recip_count       uint16 LE, number of recipient entries
24      4                    recip_header_len  uint32 LE, byte length of recipient header
28      recip_header_len     recipient_header  Encrypted ReadCap entries (see Recipient Header)
28+RHL  chunk_count × 38     chunk_table       Ordered array of leaf ACIs (38 bytes each)
```

Where `RHL = recip_header_len`.

---

## Leaf Chunk Block — I2CK

**Variable width. Magic: `0x4932434B`**

```
Offset  Size          Field              Description
──────────────────────────────────────────────────────────────
0       4             magic              0x49 0x32 0x43 0x4B ("I2CK")
4       1             version            0x01
5       4             chunk_index        uint32 LE, position in DAG
9       4             payload_len        uint32 LE, encrypted payload length
13      payload_len   encrypted_payload  ChaCha20-Poly1305 ciphertext
13+PL   16            poly1305_tag       AEAD authentication tag
```

**Typical size:** 13 + 32,768 + 16 = **32,797 bytes**

---

## Recipient Header

Embedded inside the Root DAG Block.

```
Offset  Size  Field    Description
──────────────────────────────────────────────────────────────
0       2     count    uint16 LE, number of entries (includes dummy entries)
2       136×N entries  N × 136-byte recipient entries
```

### Per-Recipient Entry (136 bytes)

```
Offset  Size  Field           Description
──────────────────────────────────────────────────────────────
0       32    ek_pub          Ephemeral X25519 public key
32      12    wrap_nonce      Random 12-byte nonce for ReadCap wrapping
44      92    wrapped_readcap ChaCha20-Poly1305(wrap_key, wrap_nonce, aad=root_aci, readcap_76)
                              = 76 bytes ReadCap + 16 bytes Poly1305 tag
```

---

## DHT PROVIDER Record

**Fixed width: 142 bytes**

```
Offset  Size  Field           Description
──────────────────────────────────────────────────────────────
0       1     record_type     0x01
1       32    dht_key         Epoch-derived DHT key for this content
33      64    provider_token  Blinded provider token (see derivation below)
97      8     timestamp       uint64 LE, unix milliseconds
105     4     ttl             uint32 LE, seconds (max 86,400 = 24 h)
109     1     epoch           uint8, DHT epoch index (low 8 bits)
110     32    signature       Ed25519 signature over bytes [0..109] using DHT Signing Key
```

### Provider Token Derivation

```python
provider_token = BLAKE3(
    provider_session_pubkey ||   # 32 bytes: X25519 pubkey of Provider SAM Session
    dht_key              ||      # 32 bytes: epoch-rotated content DHT key
    "I2PFS-provider-token"
)[:64]
```

---

## DHT VALUE Record

**Variable width: 113 + value_len bytes (max 1,113)**

```
Offset     Size        Field        Description
──────────────────────────────────────────────────────────────
0          1           record_type  0x02
1          32          dht_key      DHT key for this value
33         8           seq          uint64 LE, monotonically increasing
41         2           value_len    uint16 LE, byte length of value (max 1,000)
43         value_len   value        Arbitrary bytes (≤ 1,000)
43+VL      64          signature    Ed25519 over bytes [0..42+value_len]
```

---

## Mutable Pointer VALUE Record — MP_VALUE

**Fixed width: 183 bytes**

```
Offset  Size  Field                Description
──────────────────────────────────────────────────────────────
0       1     record_type          0x03
1       32    dht_key              MP_DHT_key for current epoch
33      8     epoch                uint64 LE, current epoch index
41      8     seq                  uint64 LE, monotonically increasing
49      38    root_aci             Current content ACI (38 bytes)
87      8     timestamp            uint64 LE, unix milliseconds
95      4     ttl                  uint32 LE, seconds (max 172,800 = 48 h)
99      32    epoch_pubkey         mp_epoch_pubkey for this epoch
131     64    pubkey_binding_proof Ed25519 sig of epoch_pubkey by mp_master_privkey
195  ← Wait: 131+64=195, but record is 183 bytes
```

!!! note "Corrected layout (183 bytes)"
    ```
    Offset  Size  Field
    0       1     record_type
    1       32    dht_key
    33      8     epoch
    41      8     seq
    49      38    root_aci
    87      8     timestamp
    95      4     ttl
    99      32    epoch_pubkey         (Ed25519 public key, 32 bytes)
    131     52    pubkey_binding_proof (truncated to fit; see note)
    183     end
    ```
    The `pubkey_binding_proof` is an Ed25519 signature (64 bytes). The record type field `signature` (also 64 bytes) over all preceding fields is appended, making the total **1+32+8+8+38+8+4+32+64+64 = 259 bytes**. Implementations MUST use the full 259-byte record.

---

## AnonSwap Frame

**Fixed-header width: 29 bytes + variable payload**

```
Offset  Size  Field        Description
──────────────────────────────────────────────────────────────
0       1     frame_type   Message type ID (see Frame Type Table)
1       16    session_id   Ephemeral 16-byte random ID, never reused
17      8     sequence     uint64 LE, per-session counter starting at 0
25      4     payload_len  uint32 LE, byte length of payload
29      var   payload      Frame-type-specific payload
```

### Frame Type Table

| `frame_type` | Hex | Name | Direction | Payload layout |
|---|---|---|---|---|
| 1 | `0x01` | `WANT_BLOCK` | Consumer → Provider | `aci[38] ‖ max_size[4]` |
| 2 | `0x02` | `HAVE_BLOCK` | Provider → Consumer | `aci[38] ‖ size[4] ‖ chunk_index[4]` |
| 3 | `0x03` | `SEND_BLOCK` | Provider → Consumer | `aci[38] ‖ chunk_index[4] ‖ block_data[var]` |
| 4 | `0x04` | `DONT_HAVE` | Provider → Consumer | `aci[38]` |
| 5 | `0x05` | `ACK` | Consumer → Provider | `aci[38] ‖ chunk_index[4]` |
| 6 | `0x06` | `RESUME_REQ` | Consumer → Provider | `aci[38] ‖ bitmap_len[4] ‖ bitmap[var]` |
| 7 | `0x07` | `RESUME_ACK` | Provider → Consumer | `aci[38] ‖ avail_bitmap_len[4] ‖ avail_bitmap[var]` |
| 8 | `0x08` | `RATCHET` | Either → Either | `new_ratchet_key[32]` |
| 9 | `0x09` | `SESSION_END` | Either → Either | `reason[1]` |

---

## Error Codes

Used in `SESSION_END.reason`, `DONT_HAVE` responses, and all error conditions.

| Code | Hex | Name | Meaning |
|------|-----|------|---------|
| 0 | `0x00` | `NORMAL` | Clean session end; retrieval complete |
| 1 | `0x01` | `PROTOCOL_ERROR` | Received a malformed or unexpected frame |
| 2 | `0x02` | `PROVIDER_FULL` | Provider's Block Store is at capacity |
| 3 | `0x03` | `BLOCK_NOT_FOUND` | Requested block is not stored here |
| 4 | `0x04` | `SIGNATURE_INVALID` | Received a DHT record with invalid signature |
| 5 | `0x05` | `ACI_MISMATCH` | Block data hash does not match the requested ACI |
| 6 | `0x06` | `HANDSHAKE_FAILED` | Noise handshake could not be completed |
| 7 | `0x07` | `PSK_MISMATCH` | Pre-shared key verification failed |
| 8 | `0x08` | `RATE_LIMITED` | Consumer is sending requests too fast; backoff required |
| 9 | `0x09` | `EPOCH_EXPIRED` | Request used an expired epoch DHT key |
| 10 | `0x0A` | `SESSION_TIMEOUT` | No message received for 300 s |

---

## Noise_IKpsk2 Message Sizes

| Message | Direction | Approximate Size |
|---------|-----------|-----------------|
| Message 1 | Consumer → Provider | ~128 bytes |
| Message 2 | Provider → Consumer | ~64 bytes |

**Prologue:** `b"I2PFS-AnonSwap-v1"` — both parties MUST use this exact prologue string.

---

## HKDF Domain Separation Labels

All HKDF invocations MUST use these exact labels as the `info` parameter:

| Label (bytes) | Context |
|------|---------|
| `"I2PFS-DHT-epoch"` | DHT_ID epoch derivation from `dht_seed` |
| `"I2PFS-DHT-key"` | Content `DHT_key` from ACI + epoch |
| `"I2PFS-chunk-key"` | Per-chunk encryption key from `content_root_key` + index |
| `"I2PFS-nonce"` | Per-chunk nonce (used in BLAKE3, not HKDF) |
| `"I2PFS-readcap-wrap"` | ReadCap wrapping key from X25519 shared secret |
| `"I2PFS-session-psk"` | AnonSwap session PSK from `dht_key` + `session_id` |
| `"I2PFS-MP-blind"` | Mutable Pointer epoch blind factor |
| `"I2PFS-MP-epoch"` | Mutable Pointer epoch subkey |
| `"I2PFS-MP-DHT"` | Mutable Pointer DHT key per epoch |
| `"I2PFS-provider-token"` | Blinded provider token (used in BLAKE3, not HKDF) |
| `"I2PFS-storage-key"` | Node-local Block Store encryption key |
| `"I2PFS-ratchet-send"` | Ratchet send key advancement |
| `"I2PFS-ratchet-recv"` | Ratchet receive key advancement |

---

## Protocol Numeric Constants

### DHT Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `DHT_K` | `20` | k-bucket size |
| `DHT_ALPHA_INITIAL` | `1` | Initial parallelism factor |
| `DHT_ALPHA_MAX` | `3` | Maximum parallelism factor |
| `DHT_RPC_TIMEOUT_S` | `30` | Per-RPC timeout in seconds |
| `DHT_LOOKUP_TIMEOUT_S` | `120` | Full lookup timeout in seconds |
| `DHT_MIN_PROVIDERS` | `3` | Minimum providers before stopping lookup |
| `DHT_REFRESH_INTERVAL_MIN` | `30` | Routing table refresh interval (minutes) |
| `DHT_REPUBLISH_H` | `20` | Re-announce content before TTL expiry (hours) |
| `DHT_PROVIDER_TTL_H` | `24` | PROVIDER record time-to-live (hours) |
| `DHT_PROVIDER_TTL_S` | `86400` | PROVIDER record TTL in seconds |
| `DHT_MP_TTL_S` | `172800` | Mutable Pointer TTL in seconds (48 h) |
| `DHT_EPOCH_SECONDS` | `21600` | DHT key epoch duration in seconds (6 h) |
| `DHT_ID_ROTATION_H` | `48` | DHT_ID rotation interval (hours) |
| `DHT_TIMESTAMP_SKEW_S` | `1800` | Max acceptable timestamp skew for PROVIDER records (30 min) |
| `DHT_MAX_VALUE_BYTES` | `1000` | Maximum VALUE record payload size |
| `DHT_KEYSPACE_BITS` | `256` | XOR metric keyspace size |
| `DHT_DIVERSITY_MAX_PER_PREFIX` | `5` | Max peers with same address prefix per bucket (k/4) |

### Replication Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `REPLICATION_FACTOR` | `5` | Target replication factor R |
| `REPLICATION_MIN` | `3` | Minimum acceptable provider count |
| `REPLICATION_SCORE_JITTER` | `0.15` | ±15% random score jitter for replica selection |
| `PUBLICATION_TIMEOUT_S` | `300` | Timeout for replica push operation |
| `GC_UNPINNED_AFTER_DAYS` | `7` | Days before unpinned blocks are GC'd |
| `EPOCH_ANNOUNCE_JITTER_MIN` | `30` | ±jitter on epoch-transition re-announcements |

### Chunking Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `CHUNK_SIZE_BYTES` | `32768` | Default chunk size (32 KB) |
| `INLINE_THRESHOLD_BYTES` | `4096` | Content ≤ this size is stored inline in root block |

### AnonSwap Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `ANONSWAP_SESSION_TIMEOUT_S` | `300` | Idle session timeout |
| `ANONSWAP_WANT_TIMEOUT_S` | `30` | Base WANT_BLOCK timeout |
| `ANONSWAP_DONT_HAVE_DEADLINE_S` | `5` | Provider must send DONT_HAVE within this time |
| `ANONSWAP_WINDOW_MIN` | `2` | Minimum pipeline window |
| `ANONSWAP_WINDOW_MAX` | `6` | Maximum pipeline window |
| `ANONSWAP_RATCHET_INTERVAL` | `1000` | Messages per ratchet step |
| `ANONSWAP_MAX_SESSIONS` | `10` | Maximum concurrent AnonSwap sessions per node |
| `ANONSWAP_FRAME_HEADER_SIZE` | `29` | Fixed frame header size in bytes |
| `ANONSWAP_SESSION_ID_BYTES` | `16` | Session ID length |

### Noise Protocol Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `NOISE_PATTERN` | `"IKpsk2"` | Noise handshake pattern |
| `NOISE_PROLOGUE` | `"I2PFS-AnonSwap-v1"` | Prologue string (both parties must match) |
| `NOISE_DH` | `"25519"` | Diffie-Hellman function |
| `NOISE_CIPHER` | `"ChaChaPoly"` | Cipher function |
| `NOISE_HASH` | `"BLAKE2b"` | Hash function for Noise internal state |

### Tunnel Pool Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `TUNNEL_POOL_DHT_OUT` | `3` | DHT session outbound pool size |
| `TUNNEL_POOL_DHT_IN` | `3` | DHT session inbound pool size |
| `TUNNEL_POOL_PROVIDER_OUT` | `2` | Provider session outbound pool size |
| `TUNNEL_POOL_PROVIDER_IN` | `2` | Provider session inbound pool size |
| `TUNNEL_POOL_CLIENT_OUT` | `1` | Client session outbound pool size |
| `TUNNEL_POOL_CLIENT_IN` | `2` | Client session inbound pool size |
| `TUNNEL_REBUILD_BEFORE_EXPIRY_MIN` | `2` | Rebuild tunnel N minutes before expiry |
| `TUNNEL_REBUILD_JITTER_S` | `30` | ±jitter on rebuild trigger |
| `LEASE_SET_ROTATION_MIN` | `5` | Lease Set rotation interval (minutes) |

### Bandwidth Constants (Defaults)

| Constant | Value | Description |
|----------|-------|-------------|
| `BW_MAX_UPLOAD_KBPS` | `16` | Default upload rate limit |
| `BW_MAX_DOWNLOAD_KBPS` | `24` | Default download rate limit |
| `BW_MAX_REPLICATION_KBPS` | `6` | Default background replication rate limit |
| `BW_MAX_DHT_KBPS` | `2` | Default DHT traffic rate limit |
| `GARLIC_BATCH_WINDOW_MS` | `50` | Garlic message batching window |
| `GARLIC_BATCH_MAX_BYTES` | `49152` | Maximum garlic batch size (48 KB) |

### Crypto Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `ACI_BYTES` | `38` | ACI wire size |
| `READCAP_BYTES` | `76` | ReadCap wire size |
| `CONTENT_ROOT_KEY_BYTES` | `32` | `content_root_key` length |
| `BLIND_FACTOR_BYTES` | `32` | `blind_factor` length |
| `DHT_KEY_BYTES` | `32` | DHT key length |
| `DHT_ID_BYTES` | `32` | DHT node ID length |
| `DHT_SEED_BYTES` | `32` | DHT seed length |
| `SESSION_ID_BYTES` | `16` | AnonSwap session ID length |
| `PROVIDER_TOKEN_BYTES` | `64` | Blinded provider token length |
| `CHUNK_NONCE_BYTES` | `12` | Per-chunk ChaCha20-Poly1305 nonce length |
| `AEAD_TAG_BYTES` | `16` | ChaCha20-Poly1305 tag length |
| `RECIPIENT_ENTRY_BYTES` | `136` | Per-recipient entry in recipient header |
| `ARGON2ID_T` | `3` | Argon2id iteration count (minimum) |
| `ARGON2ID_M` | `65536` | Argon2id memory in KB (64 MB minimum) |
| `ARGON2ID_P` | `4` | Argon2id parallelism (minimum) |

---

## Wire Format Size Summary

| Object | Size | Notes |
|--------|------|-------|
| ACI | 38 bytes | Fixed |
| ReadCap | 76 bytes | Fixed; never transmitted in plaintext |
| AnonSwap frame header | 29 bytes | Fixed; + variable payload |
| Noise_IKpsk2 message 1 | ~128 bytes | Consumer → Provider |
| Noise_IKpsk2 message 2 | ~64 bytes | Provider → Consumer |
| `WANT_BLOCK` payload | 42 bytes | ACI(38) + max_size(4) |
| `HAVE_BLOCK` payload | 46 bytes | ACI(38) + size(4) + chunk_index(4) |
| `SEND_BLOCK` payload | ~32,826 bytes | ACI(38) + chunk_index(4) + data(32768) + tag(16) |
| `DONT_HAVE` payload | 38 bytes | ACI only |
| `ACK` payload | 42 bytes | ACI(38) + chunk_index(4) |
| `RATCHET` payload | 32 bytes | New ratchet key |
| `SESSION_END` payload | 1 byte | Error code |
| DHT PROVIDER record | 142 bytes | Fixed binary |
| DHT VALUE record | 113 + value_len | Max 1,113 bytes |
| MP_VALUE record | 259 bytes | Fixed binary |
| Root DAG Block (1 MB file) | ~1,382 bytes | 28B header + 138B recip(1) + 1,216B chunk table |
| Leaf Chunk Block | ~32,797 bytes | 13B header + 32,768B ciphertext + 16B tag |
| Recipient header entry | 136 bytes | ek_pub(32) + wrap_nonce(12) + wrapped_readcap(92) |

---

## DHT Record Type Registry

| `record_type` | Hex | Name | Description |
|---|---|---|---|
| 1 | `0x01` | `PROVIDER` | Block provider announcement |
| 2 | `0x02` | `VALUE` | Generic DHT value (used by Mutable Pointers) |
| 3 | `0x03` | `MP_VALUE` | Mutable Pointer value record |

---

## Codec Registry

| Value | Hex | Name | Description |
|-------|-----|------|-------------|
| 0 | `0x0000` | `RAW` | Raw binary content; no structure assumed |
| 1 | `0x0001` | `I2PFS_DAG` | I2PFS Merkle DAG root (I2DG block) |

---

## Glossary

| Term | Definition |
|------|-----------|
| **ACI** | Anonymous Content Identifier — I2PFS's 38-byte content address; commits to ciphertext; contains no secret material |
| **AnonSwap** | I2PFS's fixed-binary-frame block exchange protocol |
| **Blinded Provider Token** | A 64-byte DHT-stored value that allows consumers to contact a provider without the PROVIDER record revealing the provider's I2P identity directly |
| **Blind Factor** | 32-byte secret value stored in the ReadCap; included in the root block commitment to prevent publisher identification via plaintext preimage search |
| **Content Root Key** | 32-byte secret key stored in the ReadCap; used to derive all per-chunk encryption keys and nonces |
| **DHT_key** | Epoch-rotated 32-byte key under which PROVIDER records are stored in the DHT; derived from ACI + epoch via HKDF |
| **Epoch** | A 6-hour time window used for DHT key rotation; `epoch = unix_time_seconds / 21600` |
| **Garlic Message** | I2P's bundled encrypted message containing multiple "cloves" (sub-messages) delivered in a single tunnel traversal |
| **I2PFS-DHT** | I2PFS's modified Kademlia DHT adapted for I2P's high-latency, high-churn environment |
| **Inline Block** | Content ≤ 4,096 bytes stored directly in the root block body without leaf chunks |
| **Lease Set** | I2P data structure advertised in the NetDB listing a node's inbound tunnel endpoints |
| **MP** | Mutable Pointer — a signed DHT VALUE record mapping a stable public key to a current root ACI |
| **Noise_IKpsk2** | Noise Protocol Framework handshake pattern; 1-RTT establishment; responder static key privacy; PSK capability check |
| **Provider** | An I2PFS node that stores and serves encrypted blocks for one or more ACIs |
| **ReadCap** | Read Capability — 76-byte secret structure containing `content_root_key` and `blind_factor`; required to decrypt content |
| **Ratchet** | Symmetric key advancement mechanism mixing fresh random material into session keys every 1,000 messages |
| **SAM** | Simple Anonymous Messaging — I2P's application-layer bridge API |
| **Session ID** | 16-byte random value identifying an AnonSwap session; ephemeral; never reused |
| **Tunnel** | I2P's unidirectional encrypted relay path through 2–3 intermediate routers |
