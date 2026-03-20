# Data Model & Chunking

---

## Chunking Strategy

I2PFS splits content into fixed-size **chunks** optimized for I2P's tunnel and bandwidth characteristics.

### Default Chunk Size: 32,768 bytes (32 KB)

| Rationale | Detail |
|-----------|--------|
| Fits within I2P streaming MTU | 32 KB + 28 B AEAD overhead + 24 B frame header ≈ 32,820 B — well within 60 KB streaming MTU |
| Amortizes DHT lookup cost | Each DHT lookup takes 30–120 s; a lookup to retrieve 32 KB is far more efficient than one for 1 KB |
| Limits retry waste | A failed chunk transfer wastes at most 32 KB, not megabytes |
| Predictable fragmentation | Each 32 KB chunk requires exactly 32 I2NP message segments (at ~1 KB raw MTU), enabling predictable flow control |

### Content Size Special Cases

| Content Size | Handling |
|---|---|
| ≤ 4,096 bytes | **Inline block**: stored directly in root block body; no leaf chunks; single ACI covers entire content |
| 4,097 – 32,768 bytes | **Single chunk**: root block contains one leaf ACI |
| > 32,768 bytes | **Multi-chunk DAG**: standard chunking into 32 KB leaves with root block containing chunk table |

---

## Merkle DAG Structure

Blocks are organized into a Merkle DAG (Directed Acyclic Graph):

```
Root Block (I2DG)
├── metadata: total_size, chunk_count, chunk_size
├── recipient_header: encrypted ReadCap entries (one per recipient)
└── chunk_table: [leaf_aci[0], leaf_aci[1], ..., leaf_aci[N-1]]

Leaf Chunk Blocks (I2CK)
├── chunk_index: position in the DAG
└── encrypted_payload: ChaCha20-Poly1305 ciphertext + tag
```

The **root ACI** is the canonical content address shared with consumers. Consumers retrieve the root block, parse the chunk table, then fetch leaf blocks in parallel.

---

## Root DAG Block Binary Format

```
Root DAG Block (little-endian binary)
├── magic            : 4 bytes  = 0x49 0x32 0x44 0x47  ("I2DG")
├── version          : 1 byte   = 0x01
├── flags            : 1 byte   bit 0=inline_content, bit 1=has_recipients, bit 2=compressed
├── total_size       : 8 bytes  uint64, original plaintext size in bytes
├── chunk_count      : 4 bytes  uint32, number of leaf chunks
├── chunk_size       : 4 bytes  uint32, nominal chunk size (last chunk may be smaller)
├── recip_count      : 2 bytes  uint16, number of recipient entries in header
├── recip_header_len : 4 bytes  uint32, byte length of the recipient header block
├── recipient_header : recip_header_len bytes  (see Encryption page)
└── chunk_table      : chunk_count × 38 bytes, ordered array of leaf chunk ACIs
```

**Size example for a 1 MB file (32 chunks):**

```
Fixed header:     4+1+1+8+4+4+2+4 = 28 bytes
Recipient header: 2 + M × 136 bytes  (M = number of recipients)
Chunk table:      32 × 38 = 1,216 bytes
─────────────────────────────────────────
Total (1 recipient): 28 + 138 + 1,216 = 1,382 bytes
```

This fits comfortably in a single I2P message.

---

## Leaf Chunk Block Binary Format

```
Leaf Chunk Block (little-endian binary)
├── magic            : 4 bytes  = 0x49 0x32 0x43 0x4B  ("I2CK")
├── version          : 1 byte   = 0x01
├── chunk_index      : 4 bytes  uint32, position in the DAG (for ordering verification)
├── payload_len      : 4 bytes  uint32, encrypted payload length
├── encrypted_payload: payload_len bytes  (ChaCha20-Poly1305 ciphertext)
└── poly1305_tag     : 16 bytes  (AEAD authentication tag)
```

**Single encryption layer only.** The `encrypted_payload` is the AEAD ciphertext produced during content encryption. No additional transport-layer envelope encryption is applied — the BLAKE3 hash in the leaf ACI provides transport integrity, and the AEAD provides confidentiality and tamper detection.

---

## Block Deduplication

BLAKE3 content hashing provides automatic deduplication at the chunk level within a node's Block Store. Two identical 32 KB spans of content produce the same leaf chunk ACI and are stored as a single block on any given provider node.

**Cross-publisher deduplication limitation:** Two publishers encrypting the same plaintext chunk produce different ciphertexts because chunk keys are derived from `content_root_key`, which is unique per publish. This is intentional — cross-publisher deduplication at the block level would create a traffic analysis oracle allowing observers to determine that two publishers stored the same content.

---

## Download Resume Checkpointing

When a consumer downloads some but not all chunks, the protocol persists a **resume checkpoint** locally:

```
Resume Checkpoint (stored locally; never transmitted)
├── root_aci       : 38 bytes
├── readcap_enc    : 76 bytes  (encrypted on disk with node-local key)
├── chunk_count    : uint32
├── bitmap         : ceil(chunk_count / 8) bytes
│                    bit N = 1 if chunk N is verified and stored
└── provider_cache : serialized provider list from last DHT lookup
```

On resume, the consumer skips chunks where `bitmap[i] == 1` and fetches only missing chunks. The provider cache avoids a full DHT lookup if the resume occurs within the same epoch.

**Checkpoint persistence:** After each successfully verified and decrypted chunk, `bitmap[i]` is set to 1 and the checkpoint is written to disk. A crash at any point means the next attempt resumes from the last persisted checkpoint position.
