# Security Threat Model

---

## Adversary Model

I2PFS is designed to provide security against the following adversary capabilities:

| Capability | Description |
|-----------|-------------|
| **Global passive adversary (GPA)** | Can observe all I2P network traffic metadata (timing, sizes, source/destination at I2P overlay level) but cannot decrypt I2P garlic-encrypted message content |
| **Active network adversary** | Can inject, replay, delay, or selectively drop I2P messages; cannot break ChaCha20-Poly1305 AEAD |
| **Sybil adversary** | Can operate up to 10% of DHT nodes; cannot generate I2P Destinations with specific DHT IDs without solving computational proof-of-work |
| **Malicious DHT node** | A DHT participant who stores false records, returns incorrect routing information, or attempts to correlate queries with identities |
| **Malicious provider** | A content provider who receives block requests and attempts to fingerprint or deanonymize the requesting consumer |
| **Malicious consumer** | A consumer who makes block requests and attempts to identify the publisher or other consumers of the same content |
| **Colluding nodes** | Up to K=20% of the network may collude |

---

## Out of Scope

I2PFS does **not** protect against:

- An adversary who controls the user's machine (software, OS, hardware)
- Long-term statistical traffic analysis requiring months of sustained observation of a single user
- Physical seizure and forensic analysis of a running node (mitigated by in-memory key handling but not fully preventable)
- Quantum adversaries (post-quantum migration path defined in [Cryptography](cryptography.md#post-quantum-migration-path))

---

## Content Integrity Attacks

| Attack | How it works | Mitigation | Residual risk |
|--------|-------------|------------|---------------|
| **Block substitution** | Provider sends a different block for a given ACI | BLAKE3 verification before accept: `BLAKE3(received) == ACI.content_hash` | None — invalid blocks always detected |
| **Chunk position swap** | Shuffle encrypted chunks within a file | Per-chunk AAD binds chunk to `(root_aci, chunk_index)`; AEAD tag fails for wrong position | None |
| **Partial delivery** | Provider sends fewer chunks than claimed | Root block commitment includes all leaf ACIs via BLAKE3; missing chunks detected | Consumer must retry; denial-of-service if all providers are malicious |
| **DHT PROVIDER record poisoning** | Sybil node publishes false PROVIDER records | Records are signed with DHT Signing Key; invalid signatures rejected; `fail_count` incremented | Sybil nodes can suppress content discovery if >50% of nearest-k nodes are adversarial |
| **ACI forgery** | Publish a different block under the same ACI | BLAKE3 collision resistance; second-preimage resistance makes forged binding infeasible | None |
| **Root commitment attack** | Modify chunk table in root block | Root block's `content_hash` is BLAKE3(encrypted root payload), verifiable via ACI | None |
| **Rollback attack on MPs** | Replay an old MP_VALUE to revert content | Monotonic `seq` field; nodes reject updates with `seq` ≤ stored `seq` | Network partition: old seq may persist on nodes that never received the new update |
| **Chosen-ciphertext attack** | Modify encrypted block, observe decryption behavior | ChaCha20-Poly1305 rejects modified ciphertext; no decryption oracle exposed | None |

---

## Publisher Anonymity Attacks

| Attack | Mitigation | Residual risk |
|--------|-----------|---------------|
| **ACI preimage to find publisher** | ACI commits to ciphertext; blind_factor in ReadCap makes plaintext preimage infeasible | If ReadCap leaks, content attributable but publisher still not identified |
| **DHT insert timing correlation** | Random jitter 0–300 s on publication; pre-warmed tunnel pools hide connection-to-publish timing | Statistical correlation over many publications is possible for a GPA |
| **Replica selection fingerprinting** | ±15% random score jitter in `SELECT_REPLICAS` | Probabilistic inference with large observation set |
| **Long-term DHT routing correlation** | DHT_ID rotation every 48 h; DHT_ID independent of I2P Destination | Long-term adversary can correlate by routing behavior pattern |
| **PROVIDER record → node identity** | Blinded provider tokens; requires additional NetDB resolution step | Multi-step correlation attack requiring both DHT and NetDB observation |

---

## Consumer Anonymity Attacks

| Attack | Mitigation | Residual risk |
|--------|-----------|---------------|
| **Block request correlation across sessions** | Per-retrieval Client SAM session rotation; per-session ephemeral IDs | Single-session: provider sees one anonymous session only |
| **Tunnel fingerprinting** | Lease Set rotation every 5 minutes; blinded Lease Set keys | Lease Set metadata observable by I2P floodfill nodes |
| **Provider logging of retrieval history** | Session ephemeral keys; provider cannot link two retrievals | Provider can log that one (unknown) client requested specific blocks in one session |
| **Timing correlation of lookup + download** | Random jitter (0–5 s) between DHT lookup completion and first WANT_BLOCK | Reduces but does not eliminate timing correlation for a GPA |
| **Want-list broadcast deanonymization** | No broadcast; all WANT_BLOCK sent unicast to DHT-discovered providers only | Provider learns which blocks are requested in the current anonymous session |

---

## Cryptographic Agility and Upgrades

All wire-format objects include version fields enabling future cryptographic algorithm migration:

**Upgrade path:**

1. **Proposal phase:** New algorithm identifiers registered in the I2PFS Crypto Registry (published on I2P eepSites). New algorithms require at least 6 months of review before production activation.
2. **Dual-support phase:** A new minor version MUST support both old and new algorithms. Nodes MUST store and serve blocks under both old-version and new-version ACIs if both exist.
3. **Deprecation phase:** Old algorithm support removed after at least 12 months and when <5% of the network still uses the old version.

Nodes MUST support all minor versions within their major version. A major version bump indicates a breaking protocol change; old-major-version nodes cannot exchange content with new-major-version nodes.

---

## Security Audit Requirements

Before any production deployment, implementations MUST undergo:

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

## Mandatory Test Vectors

All implementations MUST pass the following test vectors before production deployment:

1. **ACI computation:** Given `ciphertext_bytes`, verify `ACI.content_hash == BLAKE3(ciphertext_bytes)`
2. **Per-chunk key derivation:** Given `content_root_key` and `chunk_index`, verify `chunk_key` and `chunk_nonce`
3. **ReadCap wrapping/unwrapping:** Round-trip encrypt/decrypt with known recipient keypair
4. **Root block commitment:** Given `leaf_acis[]` and `blind_factor`, verify root commitment
5. **DHT key derivation:** Given ACI and epoch number, verify `DHT_key`
6. **Noise_IKpsk2 handshake:** Interoperation test with reference implementation at known PSK and key inputs
7. **PROVIDER record signing/verification:** Sign with DHT Signing Key; verify rejection of invalid signatures
8. **Mutable Pointer epoch blinding:** Given `mp_master_privkey` and epoch, verify `MP_DHT_key` and epoch subkey
9. **Symmetric ratchet:** Given session keys at message 1,000, verify post-ratchet keys
10. **Block bitmap checkpoint:** Write checkpoint; simulate crash; resume; verify only missing blocks re-fetched

Test vectors are published as `I2PFS-testvectors-v1.json` on the I2P eepSite.
