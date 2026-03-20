# End-to-End Encryption Protocol

---

## Architecture

Content encryption in I2PFS uses a single per-chunk AEAD layer. There is no outer transport-layer envelope encryption — the BLAKE3 hash in each leaf ACI provides transport integrity, and the ChaCha20-Poly1305 AEAD provides confidentiality and tamper detection.

**Encryption flow:**

```
plaintext
    │
    ▼
Split into 32 KB chunks
    │
    ▼  (for each chunk independently)
Derive chunk_key[i] and chunk_nonce[i] from content_root_key
    │
    ▼
ChaCha20-Poly1305-Seal(key, nonce, aad=root_aci‖chunk_index, plaintext_chunk)
    │
    ▼
Encrypted chunk + AEAD tag → Leaf Chunk Block (I2CK)
    │
    ▼
BLAKE3(ciphertext‖tag) → leaf_aci[i]
```

This enables **streaming encryption**: chunks can be encrypted and sent while later chunks are still being read from disk. It also enables **incremental verification**: each chunk is verified independently without reconstructing the full file.

---

## Content Encryption

```python
def ENCRYPT_AND_CHUNK(plaintext_bytes: bytes, recipients: list) -> tuple:

    # Step 1: Generate secrets
    content_root_key = CSPRNG(32)   # never transmitted; stored only in ReadCap
    blind_factor     = CSPRNG(32)   # never transmitted; stored only in ReadCap

    # Step 2: Split plaintext into 32 KB chunks
    chunks = split_into_chunks(plaintext_bytes, chunk_size=32768)

    encrypted_chunks = []
    leaf_acis        = []

    # Step 3: Encrypt each chunk independently
    for i, chunk in enumerate(chunks):
        chunk_key   = HKDF_SHA256(content_root_key, "I2PFS-chunk-key", uint32_le(i), 32)
        chunk_nonce = BLAKE3(content_root_key || "I2PFS-nonce" || uint32_le(i))[:12]
        chunk_aad   = root_aci_bytes || uint32_le(i)   # computed after root_aci is known

        ciphertext, tag = ChaCha20Poly1305_Seal(chunk_key, chunk_nonce, chunk_aad, chunk)

        leaf_hash     = BLAKE3(ciphertext || tag)
        leaf_aci[i]   = ACI(version=0x01, codec=0x0000, hash_algo=0x1E, content_hash=leaf_hash)
        encrypted_chunks.append((ciphertext, tag))
        leaf_acis.append(leaf_aci[i])

    # Step 4: Compute root commitment
    root_commitment = BLAKE3(leaf_acis[0] || leaf_acis[1] || ... || leaf_acis[N-1] || blind_factor)
    root_aci = ACI(version=0x01, codec=0x0001, hash_algo=0x1E, content_hash=root_commitment)

    # Step 5: Build ReadCap
    readcap = ReadCap(
        content_root_key = content_root_key,
        blind_factor     = blind_factor,
        version_tag      = b"I2PFSReadCap1"
    )

    # Step 6: Wrap ReadCap for each recipient
    recipient_entries = []
    for recipient_pubkey in recipients:
        ek_pub, ek_priv = X25519_keygen()
        shared_secret   = X25519(ek_priv, recipient_pubkey)
        wrap_key        = HKDF_SHA256(shared_secret, ek_pub, "I2PFS-readcap-wrap" || root_aci_bytes, 32)
        wrap_nonce      = CSPRNG(12)   # safe: wrap_key is single-use
        wrapped_readcap, wrap_tag = ChaCha20Poly1305_Seal(wrap_key, wrap_nonce, root_aci_bytes, readcap_bytes)

        # 32 + 12 + 76 + 16 = 136 bytes per recipient entry
        recipient_entries.append(ek_pub || wrap_nonce || wrapped_readcap || wrap_tag)

    # Step 7: Build root block
    recipient_header = uint16_le(len(recipients)) || join(recipient_entries)
    root_block = RootDAGBlock(
        total_size       = len(plaintext_bytes),
        chunk_count      = len(chunks),
        chunk_size       = 32768,
        recip_count      = len(recipients),
        recipient_header = recipient_header,
        chunk_table      = leaf_acis
    )

    return root_aci, root_block, encrypted_chunks
    # Note: ReadCap is NOT returned directly — it is embedded in root_block.recipient_header
    # The publisher accesses their own content by decrypting their own recipient entry
```

---

## Chunk AAD Binding

The AAD (Additional Authenticated Data) for each chunk encryption is:

```python
chunk_aad[i] = root_aci_bytes || uint32_le(i)
```

This binding prevents two attacks:

1. **Cross-file substitution:** A chunk intended for file A cannot be passed off as part of file B — the root ACI in the AAD won't match.
2. **Position swapping:** A chunk from position 5 cannot be passed off as position 3 — the chunk index in the AAD won't match. The AEAD tag fails immediately.

---

## Content Decryption

```python
def RETRIEVE_AND_DECRYPT(root_aci: ACI, root_block: RootDAGBlock,
                          leaf_blocks: list, recipient_privkey: bytes) -> bytes:

    # Step 1: Verify root block against ACI
    # (done before this call; BLAKE3(root_block.payload) == root_aci.content_hash)

    # Step 2: Parse recipient header
    recipient_entries = parse_recipient_header(root_block.recipient_header)

    # Step 3: Find and decrypt ReadCap
    readcap = None
    for entry in recipient_entries:
        ek_pub, wrap_nonce, wrapped_readcap = parse_entry(entry)  # 32, 12, 92 bytes
        shared_secret = X25519(recipient_privkey, ek_pub)
        wrap_key      = HKDF_SHA256(shared_secret, ek_pub, "I2PFS-readcap-wrap" || root_aci_bytes, 32)
        try:
            readcap_bytes = ChaCha20Poly1305_Open(wrap_key, wrap_nonce, root_aci_bytes, wrapped_readcap)
            readcap = parse_readcap(readcap_bytes)
            break
        except AEADAuthFail:
            continue  # not our entry; try next

    if readcap is None:
        raise DecryptionFailed("not a recipient")

    # Step 4: Verify version tag
    assert readcap.version_tag == b"I2PFSReadCap1"

    # Step 5: Verify root block commitment
    expected_commitment = BLAKE3(
        root_block.chunk_table[0] || ... || root_block.chunk_table[N-1] || readcap.blind_factor
    )
    assert expected_commitment == root_aci.content_hash, "INTEGRITY_FAILED"

    # Step 6: Decrypt each leaf chunk (in parallel)
    plaintext_chunks = []
    for i, (ciphertext, tag) in enumerate(leaf_blocks):
        # Verify ciphertext integrity
        actual_hash = BLAKE3(ciphertext || tag)
        assert actual_hash == root_block.chunk_table[i].content_hash, "CHUNK_INTEGRITY_FAILED"

        # Derive decryption parameters
        chunk_key   = HKDF_SHA256(readcap.content_root_key, "I2PFS-chunk-key", uint32_le(i), 32)
        chunk_nonce = BLAKE3(readcap.content_root_key || "I2PFS-nonce" || uint32_le(i))[:12]
        chunk_aad   = root_aci_bytes || uint32_le(i)

        # Decrypt and verify AEAD tag
        plaintext_chunk = ChaCha20Poly1305_Open(chunk_key, chunk_nonce, chunk_aad, ciphertext || tag)
        plaintext_chunks.append(plaintext_chunk)
        # Update resume checkpoint bitmap[i] = 1

    # Step 7: Reassemble
    plaintext = join(plaintext_chunks)
    return plaintext[:root_block.total_size]   # truncate padding in last chunk
```

---

## Multi-Recipient Sharing

Multiple recipients are supported by adding additional entries to `recipient_header`:

- Each entry is exactly **136 bytes**: `ek_pub(32) + wrap_nonce(12) + wrapped_readcap(76+16=92)`
- All entries are indistinguishable in size and structure to non-holders
- A holder iterates all entries until their decryption succeeds
- Non-recipients can see `recip_count` (how many recipients exist) but cannot determine their identities

**Recipient count privacy:** Publishers who wish to conceal the number of recipients MAY add dummy entries encrypted to randomly generated keys (not corresponding to any real recipient). Consumers iterate all entries regardless and try each.

---

## Recipient Header Structure

```
recipient_header:
├── count     : 2 bytes  uint16, number of entries (including any dummy entries)
└── entries[] : count × 136 bytes
    each entry:
    ├── ek_pub        : 32 bytes  ephemeral X25519 public key
    ├── wrap_nonce    : 12 bytes  random nonce for ReadCap wrapping
    └── wrapped_readcap: 92 bytes  (76-byte ReadCap + 16-byte Poly1305 tag)
                         encrypted with ChaCha20-Poly1305(wrap_key, wrap_nonce, aad=root_aci)
```
