# Publish & Retrieve Flows

This page provides complete worked examples of the two primary I2PFS operations, tracing every step from local key generation through network operations to final plaintext delivery.

---

## What Moves and Who Sees What

Before the flow details, the key mental model:

**Two objects** are always required for retrieval:

| Object | Who has it | What it enables |
|--------|-----------|----------------|
| `ACI` (38 bytes) | Anyone — shareable publicly | Locates the ciphertext via DHT; verifies block integrity |
| `ReadCap` (76 bytes) | Authorized recipients only | Decrypts and fully verifies the plaintext |

**Three identity layers** never cross-contaminate. See [Node Identity & Anonymity](identity-anonymity.md) for the full model.

---

## Publish Flow

### Phase 1: Local Preparation (no network I/O, ~100 ms)

All cryptographically critical operations happen before any tunnel is touched.

```
1.  Read file → plaintext_bytes

2.  Generate secrets:
      content_root_key := CSPRNG(32)   # never transmitted
      blind_factor     := CSPRNG(32)   # never transmitted

3.  Split plaintext into 32 KB chunks:
      chunks[0..N-1]

4.  For each chunk chunks[i]:
      a. chunk_key[i]   = HKDF-SHA256(content_root_key, "I2PFS-chunk-key", uint32_le(i), 32)
      b. chunk_nonce[i] = BLAKE3(content_root_key || "I2PFS-nonce" || uint32_le(i))[:12]
         # Deterministic — nonce reuse structurally impossible
      c. chunk_aad[i]   = root_aci_bytes || uint32_le(i)
         # Binds chunk to its position — prevents position-swap attacks
      d. (ciphertext[i], tag[i]) = ChaCha20-Poly1305-Seal(
             key=chunk_key[i], nonce=chunk_nonce[i], aad=chunk_aad[i], data=chunks[i])
      e. leaf_aci[i].content_hash = BLAKE3(ciphertext[i] || tag[i])

5.  root_commitment = BLAKE3(leaf_aci[0] || ... || leaf_aci[N-1] || blind_factor)
    root_aci.content_hash = root_commitment

6.  Wrap ReadCap for each recipient R[j]:
      ek_pub, ek_priv    = X25519_keygen()
      shared_secret      = X25519(ek_priv, R[j].encryption_pubkey)
      wrap_key           = HKDF-SHA256(shared_secret, ek_pub, "I2PFS-readcap-wrap"||root_aci, 32)
      wrap_nonce         = CSPRNG(12)
      wrapped_readcap[j] = ChaCha20-Poly1305-Seal(wrap_key, wrap_nonce, root_aci, readcap_bytes)
      # 136 bytes per recipient entry: ek_pub(32) + wrap_nonce(12) + wrapped_readcap(92)

7.  Build root block (I2DG): chunk_table + recipient_header

8.  Persist root block + all leaf blocks to local Block Store (encrypted at rest)
```

### Phase 2: DHT Lookup for Candidate Pool (~20–90 s, concurrent with Phase 3 after first hop)

```
1.  Derive DHT_key = HKDF-SHA256(root_aci_bytes, "I2PFS-DHT-key", epoch_bytes, 32)
2.  Run FIND_PROVIDERS(DHT_key)
    → Goal: collect k=20 nearest DHT peers (not existing providers — content is new)
    → These become the replica candidate pool
3.  On first DHT response: begin Phase 3 in parallel
```

### Phase 3: Replica Push (~25–40 s, overlaps with DHT)

```
For each of 5 selected replica nodes (run in parallel):

  a. Establish AnonSwap session:
       PSK = HKDF-SHA256(dht_key, "I2PFS-session-psk", session_id, 32)
       Noise_IKpsk2 handshake (2 messages, 1 RTT, ~500–800 ms)

  b. Send root block:
       SEND_BLOCK(root_aci, root_block_data)
       Await ACK

  c. Pipeline leaf chunks (window ≤ 6):
       for each leaf_aci[i]:
           SEND_BLOCK(leaf_aci[i], ciphertext[i]||tag[i], chunk_index=i)
       Await all ACKs (PUBLICATION_TIMEOUT = 300 s)

  d. On all chunks confirmed:
       Replica autonomously publishes PROVIDER records in DHT
       (for root_aci and all leaf_aci[i])
```

### Phase 4: DHT Announcement (~30 s, overlaps with Phase 3)

```
1.  Local node also publishes PROVIDER record for root_aci
    (self is also a provider initially)
2.  Publish PROVIDER records for all leaf_aci[i]
3.  Return root_aci to caller
    # ReadCap is NOT returned separately — it is embedded in root_block.recipient_header
    # Publisher accesses their own content by decrypting their own recipient entry
```

### Expected Publication Times

| Phase | Duration |
|-------|---------|
| Local encryption + chunking | ~100 ms |
| DHT lookup (3–4 hops × 20–30 s) | ~60–90 s |
| Replica push × 5, pipelined (overlaps DHT) | ~25–40 s |
| DHT announcement (overlaps push) | ~30 s |
| **Total** | **~70–110 s** |

---

## Retrieve Flow

### Phase 1: Root Block Discovery (~20–90 s)

```
1.  Derive DHT_key = HKDF-SHA256(root_aci_bytes, "I2PFS-DHT-key", current_epoch, 32)

2.  Run FIND_PROVIDERS(DHT_key) → provider_list
    (Uses adaptive timeout chain: see Latency Optimization)

3.  Rotate Client SAM Session to fresh Destination
    (per-retrieval anonymity: provider cannot link this retrieval to any past one)

4.  Send WANT_BLOCK(root_aci) to top-3 providers simultaneously

5.  Receive root_block from first responding provider

6.  Verify root block integrity:
      BLAKE3(root_block.encrypted_payload) == root_aci.content_hash ✓
      If fails: discard, retry on next provider
```

### Phase 2: ReadCap Extraction (~1 ms, local)

```
1.  Parse recipient_header from root_block
    (count entries by recip_count; each entry is exactly 136 bytes)

2.  For each recipient entry:
      ek_pub, wrap_nonce, wrapped_readcap = entry[0:32], entry[32:44], entry[44:136]
      shared_secret = X25519(recipient_privkey, ek_pub)
      wrap_key = HKDF-SHA256(shared_secret, ek_pub, "I2PFS-readcap-wrap"||root_aci, 32)
      Try: readcap = ChaCha20-Poly1305-Open(wrap_key, wrap_nonce, root_aci, wrapped_readcap)
      If tag valid → ReadCap found. Break.
      If tag fails → not our entry. Try next.

3.  If no entry decrypts: DECRYPTION_FAILED (caller is not a recipient)

4.  Verify version_tag == "I2PFSReadCap1"

5.  Verify root commitment using blind_factor from ReadCap:
      expected = BLAKE3(chunk_table[0] || ... || chunk_table[N-1] || readcap.blind_factor)
      assert expected == root_aci.content_hash ✓
      # This verifies the entire chunk table structure at once
```

### Phase 3: Leaf Chunk Retrieval (~15–30 s, pipelined and parallel)

```
1.  Check local resume checkpoint:
      If bitmap exists for this root_aci: skip chunks where bitmap[i] == 1

2.  For each needed chunk i (pipeline window ≤ 6):

    a. Check provider cache:
         If same epoch: reuse provider_list from root block lookup
         Else: run FIND_PROVIDERS(HKDF-SHA256(leaf_aci[i], "I2PFS-DHT-key", epoch, 32))

    b. Send WANT_BLOCK(leaf_aci[i]) via AnonSwap session

    c. Receive SEND_BLOCK response:
         Verify: BLAKE3(chunk_data || tag) == leaf_aci[i].content_hash ✓
         If fails: discard; retry on next provider

    d. Decrypt chunk:
         chunk_key[i]   = HKDF-SHA256(readcap.content_root_key, "I2PFS-chunk-key", uint32_le(i), 32)
         chunk_nonce[i] = BLAKE3(readcap.content_root_key || "I2PFS-nonce" || uint32_le(i))[:12]
         chunk_aad[i]   = root_aci_bytes || uint32_le(i)
         plaintext[i]   = ChaCha20-Poly1305-Open(chunk_key[i], chunk_nonce[i], chunk_aad[i], data)
         If AEAD tag fails: block corrupt or tampered; retry from different provider

    e. Write plaintext[i] to output buffer
       Update resume checkpoint: bitmap[i] = 1; persist to disk

    f. Send ACK(leaf_aci[i], chunk_index=i) → window slides
```

### Phase 4: Reassembly (~1 ms, local)

```
1.  plaintext = join(plaintext_chunks[0], ..., plaintext_chunks[N-1])
2.  plaintext = plaintext[:root_block.total_size]  # trim padding from last chunk
3.  Delete resume checkpoint (retrieval complete)
4.  Return plaintext
```

### Expected Retrieval Times

| Phase | Typical | Worst Case |
|-------|---------|------------|
| DHT lookup + root block | 20–60 s | 90 s |
| ReadCap decrypt + verify | <1 ms | <1 ms |
| Leaf chunk retrieval (pipelined) | 15–30 s | 60 s |
| Reassembly | <5 ms | <5 ms |
| **Total** | **35–90 s** | **120–150 s** |

---

## Security Properties of Each Step

| Step | Security property enforced |
|------|---------------------------|
| chunk_nonce derivation | Nonce uniqueness by construction; no random nonce reuse risk |
| chunk_aad = root_aci ‖ chunk_index | Chunk position binding — position-swap attack structurally impossible |
| leaf_aci hashes ciphertext | Block integrity verifiable without decryption; tampered blocks rejected before use |
| root_commitment includes blind_factor | Publisher binding — ACI is not computable from plaintext alone without ReadCap |
| ReadCap wrapped per-recipient | Authorization — only designated recipients can decrypt |
| PSK from DHT_key | Capability check — only nodes that performed DHT lookup can open AnonSwap session |
| Per-retrieval Client SAM rotation | Retrieval anonymity — provider sees one anonymous session per retrieval |
| Symmetric ratchet every 1,000 msgs | Post-compromise security — exposure window bounded to ~32 MB |

---

## Interrupted Download Resume

If a retrieval is interrupted mid-way:

```
1.  Consumer has persisted checkpoint with bitmap[i] = 1 for all received chunks
2.  On resume:
      a. Load checkpoint: root_aci, encrypted readcap, bitmap, provider_cache
      b. Decrypt readcap from checkpoint using node-local key
      c. Check if provider_cache is still valid for current epoch
      d. Skip chunks where bitmap[i] == 1
      e. Resume WANT_BLOCK requests for remaining chunks from step in Phase 3

No re-downloading of already-verified chunks.
No re-running DHT lookup if cache is still valid for current epoch.
```
