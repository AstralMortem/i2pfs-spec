# Implementation Guide

---

## Language and Runtime

| Language | Assessment | Notes |
|----------|-----------|-------|
| **Rust** | Strongly recommended | Memory safety without GC pauses; excellent async runtime (tokio); existing SAM bridge crates (`i2p-rs`, `ire`); best performance for crypto operations |
| Go | Acceptable | Good concurrency primitives; adequate crypto libraries; some GC pause risk during streaming |
| Java / Kotlin | Possible | Native I2P is JVM-based; enables tighter I2P integration; GC pauses are a concern for timing-sensitive anonymity operations |
| Python | Not recommended | GIL and GC make it unsuitable for high-throughput crypto; acceptable for prototyping only |

!!! danger "Constant-time requirement"
    All cryptographic operations (AEAD, key derivation, handshakes) MUST use constant-time implementations. Timing side-channels in crypto code can leak key material. Use audited libraries (libsodium, ring, BoringSSL) rather than custom implementations.

---

## Rust Dependency Reference

```toml
[dependencies]
tokio            = { version = "1", features = ["full"] }   # async runtime
snow             = "0.9"                                     # Noise Protocol (IKpsk2)
blake3           = "1"                                       # BLAKE3 hash + KDF
chacha20poly1305 = "0.10"                                    # AEAD (RustCrypto, audited)
x25519-dalek     = { version = "2", features = ["static_secrets"] }  # X25519, constant-time
ed25519-dalek    = "2"                                       # Ed25519, constant-time
hkdf             = "0.12"                                    # HKDF-SHA256
sha2             = "0.10"                                    # SHA-256 for HKDF
argon2           = "0.5"                                     # Argon2id for passphrase KDF
rocksdb          = "0.21"                                    # block store backend
zeroize          = "1"                                       # secure memory zeroing
```

---

## Storage Backend

| Backend | Recommendation | Notes |
|---------|---------------|-------|
| **RocksDB** | Strongly recommended | LSM-tree; high write throughput; compaction handles block churn; supports prefix scans for DHT bucket queries |
| SQLite | Acceptable for small nodes (<10 GB) | Simpler deployment; adequate under light load; WAL mode required |
| Filesystem (content-addressed) | Acceptable for prototyping | Simple; benefits from OS page cache; degrades with >100,000 blocks |

**Mandatory separation:** Block data and DHT routing table MUST be stored in separate databases to allow independent backup, compaction, and emergency clearing.

### At-Rest Encryption

The Block Store MUST encrypt all stored blocks using a node-local key derived from the Master Identity Key. This ensures that physical seizure of storage media does not expose content to adversaries without the node's key material.

```rust
// Node-local storage key derivation
let storage_key = hkdf_sha256(
    ikm  = &master_secret,
    salt = b"I2PFS-storage-key",
    info = &node_id_commitment,
    len  = 32,
);
```

### Key Material Storage

The Master Identity Key MUST be stored encrypted with `Argon2id(passphrase)` on disk. The passphrase is never stored. The decrypted key is held only in memory. On graceful shutdown, memory containing key material MUST be explicitly zeroed (use the `zeroize` crate in Rust).

---

## Concurrency Architecture

I2PFS requires careful concurrency design to achieve performance targets without saturating I2P tunnel capacity.

### Recommended Actor/Task Model (Rust/tokio)

```
Main Runtime
│
├── DHT Actor (1 task)
│   ├── Routing table maintenance (periodic)
│   ├── RPC dispatch (per RPC, spawned)
│   └── PROVIDER record maintenance (periodic)
│
├── AnonSwap Actor (1 task per active session, max 10)
│   ├── Noise handshake state machine
│   ├── Block send/receive pipeline
│   └── Ratchet management
│
├── Replication Actor (1 task)
│   ├── Repair detection (periodic, every 30 min)
│   └── Push replication (per repair, spawned, P4 priority)
│
├── GC Actor (1 task)
│   └── Garbage collection (daily, P4 priority)
│
└── SAM Bridge Actor (1 task per SAM session = 3 tasks)
    ├── Inbound connection handling
    └── Outbound message queue (priority-ordered)
```

**Maximum concurrent AnonSwap sessions: 10.** Each session consumes at minimum 1 outbound + 1 inbound tunnel slot. Exceeding this risks tunnel pool exhaustion.

---

## Startup Sequence

```rust
async fn startup(config: Config, passphrase: SecretString) -> Result<Node> {
    // 1. Load or generate Master Identity Key
    let master_key = load_or_generate_master_key(&config.keys_dir, &passphrase).await?;

    // 2. Derive dht_seed (independent of master key path)
    let dht_seed = load_or_generate_dht_seed(&config.keys_dir, &master_key).await?;

    // 3. Create three SAM sessions
    let sam_dht      = SamSession::create_datagram("I2PFS-DHT", &config.sam).await?;
    let sam_provider = SamSession::create_stream("I2PFS-PROVIDER", &config.sam).await?;
    let sam_client   = SamSession::create_stream("I2PFS-CLIENT", &config.sam).await?;

    // 4. Pre-warm tunnel pools for all three sessions
    sam_dht.prewarm_tunnels(3, 3).await?;
    sam_provider.prewarm_tunnels(2, 2).await?;
    sam_client.prewarm_tunnels(1, 2).await?;

    // 5. Load DHT routing table; begin background refresh
    let dht = DHTStore::open(&config.dht_dir)?;
    dht.start_refresh_loop(sam_dht.clone());

    // 6. Load Block Store; resume pending PROVIDER announcements
    let blocks = BlockStore::open(&config.blocks_dir, &master_key)?;
    blocks.resume_announcements(&dht).await?;

    // 7. Begin accepting inbound AnonSwap connections
    sam_provider.accept_loop(blocks.clone()).await?;

    Ok(Node { master_key, dht, blocks, sam_dht, sam_provider, sam_client })
}
```

---

## Shutdown Sequence

```rust
async fn shutdown(node: Node) {
    // 1. Stop accepting new connections
    node.sam_provider.stop_accepting();

    // 2. Allow in-progress sessions to complete (grace period: 30 s)
    node.session_manager.drain(Duration::from_secs(30)).await;

    // 3. Flush all pending DHT announcements
    node.dht.flush_announcements().await;

    // 4. Flush Block Store writes
    node.blocks.flush().await;

    // 5. Close SAM sessions
    node.sam_dht.close().await;
    node.sam_provider.close().await;
    node.sam_client.close().await;

    // 6. Zero all key material in memory
    node.master_key.zeroize();
    node.dht_seed.zeroize();
    // All derived keys are zeroized by Drop impls
}
```

---

## Configuration Reference

```toml
[node]
dht_id_rotation_hours              = 48
lease_set_rotation_minutes         = 5
provider_session_rotation_hours    = 24
provider_session_max_connections   = 1000

[dht]
k_bucket_size                      = 20
alpha_initial                      = 1
alpha_max                          = 3
rpc_timeout_s                      = 30
lookup_timeout_s                   = 120
min_providers                      = 3
refresh_interval_minutes           = 30
republish_hours                    = 20

[storage]
chunk_size_bytes                   = 32768
replication_factor                 = 5
replication_min                    = 3
provider_ttl_hours                 = 24
provider_reannounce_hours          = 20
gc_unpinned_after_days             = 7
block_store_backend                = "rocksdb"

[bandwidth]
max_upload_kbps                    = 16
max_download_kbps                  = 24
max_replication_kbps               = 6
max_dht_kbps                       = 2
pipeline_window_max                = 6
garlic_batch_window_ms             = 50
garlic_batch_max_bytes             = 49152

[crypto]
noise_pattern                      = "IKpsk2"
aead                               = "chacha20-poly1305"
hash                               = "blake3"
kdf                                = "hkdf-sha256"
argon2id_t                         = 3
argon2id_m                         = 65536
argon2id_p                         = 4
ratchet_interval_messages          = 1000

[tunnels]
outbound_pool_size                 = 3
inbound_pool_size                  = 3
rebuild_before_expiry_minutes      = 2
rebuild_jitter_seconds             = 30

[sam]
host                               = "127.0.0.1"
port                               = 7656
version                            = "3.3"
```

---

## Compliance Checklist

Before declaring an implementation production-ready:

- [ ] All test vectors pass (`I2PFS-testvectors-v1.json`)
- [ ] External cryptographic review complete
- [ ] All deserialization paths fuzzed
- [ ] CSPRNG seeding and entropy reviewed
- [ ] Constant-time execution verified for X25519, Ed25519, AEAD
- [ ] Memory zeroing on shutdown verified
- [ ] Three independent SAM sessions with independent Destinations
- [ ] DHT_ID generated independently from I2P Destinations
- [ ] PROVIDER records use blinded tokens (no raw I2P Destinations)
- [ ] Per-retrieval Client SAM session rotation implemented
- [ ] Block bitmap checkpointing implemented and tested
- [ ] Adaptive timeout chain implemented for all block fetches
- [ ] Symmetric ratchet implemented and tested
- [ ] At-rest Block Store encryption implemented
- [ ] Bandwidth quotas and priority queues implemented
