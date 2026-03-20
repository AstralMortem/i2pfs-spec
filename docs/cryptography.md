# Cryptographic Foundations

---

## Primitive Selection

| Primitive | Algorithm | Rationale |
|-----------|-----------|-----------|
| AEAD encryption | ChaCha20-Poly1305 (RFC 8439) | Constant-time on all hardware; no AES-NI requirement; 256-bit security |
| Key agreement | X25519 (ECDH) | Constant-time; 32-byte keys; native I2P crypto compatibility; high performance |
| Signatures | Ed25519 | 64-byte signatures; fast batch verification; native I2P support; 128-bit security |
| Content hashing | BLAKE3 | ~3× faster than SHA-256; built-in parallel tree hashing enables per-chunk streaming verification |
| Key derivation | HKDF-SHA256 (RFC 5869) | Composable; domain-separated; widely audited |
| Passphrase KDF | Argon2id (RFC 9106) | Memory-hard; resists GPU and ASIC brute-force of user-chosen passphrases |
| Session handshake | Noise_IKpsk2 | Pre-shared key enables fast reconnection; responder static key hidden from passive observers; 1 RTT establishment |
| Symmetric ratchet | BLAKE3-based KDF chain | Lightweight; provides break-in recovery without Double Ratchet overhead impractical for bulk transfer |
| Random generation | OS CSPRNG (`getrandom` / `CryptGenRandom`) | Platform-native; no custom entropy pools |

!!! danger "Implementation requirement"
    All cryptographic operations (AEAD, key derivation, handshakes) **MUST** use constant-time implementations. Timing side-channels in crypto code can leak key material. Use audited libraries (libsodium, ring, BoringSSL) rather than custom implementations.

---

## Argon2id Parameters

Minimum parameters for all Argon2id invocations in this protocol:

```
t (iterations)  = 3
m (memory)      = 65536  (64 MB)
p (parallelism) = 4
```

These are minimums. Implementations MAY use higher parameters where user experience permits.

---

## Forward Secrecy

Forward secrecy in I2PFS operates at two levels:

**Session level:** All AnonSwap sessions use Noise_IKpsk2 with ephemeral X25519 keys generated fresh for each session. Compromise of the Provider Static Key does not expose content of past sessions — only enables impersonation for future sessions until the next key rotation.

**Content level:** Each published file uses a fresh randomly generated `content_root_key` stored only in the ReadCap. Compromise of the publisher's node after publication does not expose the content if the ReadCap was distributed out-of-band and the node's key material is subsequently destroyed.

---

## Nonce Management

!!! danger "Critical"
    Nonce reuse under the same key is **catastrophic** for ChaCha20-Poly1305 — it reveals the XOR of two plaintexts. I2PFS prevents reuse through deterministic nonce derivation, eliminating any reliance on random nonce uniqueness for content encryption.

### Per-Chunk Content Nonces

```python
chunk_nonce[i] := BLAKE3(content_root_key || "I2PFS-nonce" || uint32_le(i))[:12]
```

Because `content_root_key` is unique per publish and `i` is the chunk index, no two encryptions under the same key ever produce the same nonce.

### Session Nonces (Noise Transport Phase)

Standard Noise protocol 64-bit little-endian counter nonces, starting at 0. Sessions MUST be torn down and re-established before the counter reaches 2^32 messages (enforced by the symmetric ratchet at every 1,000 messages — see [AnonSwap Protocol](anonswap.md#symmetric-ratchet)).

### ReadCap Wrapping Nonces

Random 12-byte nonces are used **only** for wrapping the ReadCap per recipient. This is safe because each wrapping uses a fresh ephemeral X25519 key, making the wrapping key itself single-use by construction.

---

## HKDF Domain Separation

All HKDF invocations in this protocol use explicit domain separation labels in the `info` parameter. The following labels are defined:

| Label | Context |
|-------|---------|
| `"I2PFS-DHT-epoch"` | DHT_ID epoch derivation |
| `"I2PFS-DHT-key"` | Content DHT_key from ACI |
| `"I2PFS-chunk-key"` | Per-chunk encryption key from content_root_key |
| `"I2PFS-readcap-wrap"` | ReadCap wrapping key from X25519 shared secret |
| `"I2PFS-session-psk"` | AnonSwap session PSK |
| `"I2PFS-MP-blind"` | Mutable Pointer epoch blind factor |
| `"I2PFS-MP-epoch"` | Mutable Pointer epoch subkey |
| `"I2PFS-provider-token"` | Blinded provider token |

---

## Post-Quantum Migration Path

The current primitive set (X25519, Ed25519, ChaCha20-Poly1305) is not quantum-resistant. When quantum-safe algorithms are sufficiently standardized and performant, I2PFS SHOULD migrate:

| Current | Replacement | Status |
|---------|-------------|--------|
| X25519 key agreement | ML-KEM (CRYSTALS-Kyber, FIPS 203) | Standardized; awaiting I2P adoption |
| Ed25519 signatures | ML-DSA (CRYSTALS-Dilithium, FIPS 204) | Standardized; larger keys |
| ChaCha20-Poly1305 | No change needed | 256-bit symmetric keys already quantum-resistant |

The hybrid approach (X25519 + ML-KEM combined key agreement) SHOULD be used during the transition period to maintain security against both classical and quantum adversaries simultaneously.

Algorithm identifiers are versioned in all wire formats to support this migration. See [Types, Constants & Wire Formats](types-constants.md) for version field definitions.
