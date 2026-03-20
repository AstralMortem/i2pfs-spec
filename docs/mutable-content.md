# Mutable Content & Naming

---

## The Mutability Problem

Content-addressed systems are inherently immutable: the ACI of a file is derived from its content, so any change produces a new ACI. For evolving content — a continuously updated feed, a software repository, a personal file storage root — users need a stable address that can be updated to point to new content versions.

I2PFS provides **Mutable Pointers (MPs)**: signed DHT records that map a stable public key to a current root ACI.

---

## Mutable Pointer Key Blinding

To prevent the stable signing key from being a long-term tracking identifier in the DHT, I2PFS uses an **epoch-blinded signing scheme**:

```python
# Long-term stable key (the "channel address" shared with subscribers)
mp_master_privkey = Ed25519_keygen()
mp_master_pubkey  = corresponding_pubkey

# Per-epoch blind factor (derived from master key; changes every 48 h)
mp_blind_factor[epoch] = HKDF_SHA256(
    ikm  = mp_master_privkey,
    salt = "I2PFS-MP-blind",
    info = uint64_le(epoch),
    len  = 32
)

# Per-epoch signing key (derived; not the master key)
mp_epoch_privkey[epoch], mp_epoch_pubkey[epoch] = Ed25519_derive_subkey(
    master_key       = mp_master_privkey,
    derivation_path  = "I2PFS-MP-epoch" || uint64_le(epoch)
)

# Per-epoch DHT key
MP_DHT_key[epoch] = BLAKE3(
    mp_master_pubkey || "I2PFS-MP-DHT" || uint64_le(epoch)
)
```

**`mp_master_pubkey`** is the persistent channel address shared with subscribers. Subscribers who know it can derive `MP_DHT_key[current_epoch]` to look up the current content pointer. The DHT record is signed with the epoch-specific subkey, which rotates every 48 hours. An observer who captures MP records from multiple epochs cannot link them to the same publisher without already knowing `mp_master_pubkey`.

---

## Mutable Pointer Record Structure

```
MP_VALUE record (binary, 183 bytes fixed):
├── record_type          : 1 byte   = 0x03
├── dht_key              : 32 bytes (MP_DHT_key for current epoch)
├── epoch                : 8 bytes  uint64
├── seq                  : 8 bytes  uint64  (monotonically increasing; higher seq wins)
├── root_aci             : 38 bytes (current content ACI)
├── timestamp            : 8 bytes  uint64 unix ms
├── ttl                  : 4 bytes  uint32 seconds (max 172,800 = 48 h)
├── epoch_pubkey         : 32 bytes (mp_epoch_pubkey for this epoch; for signature verification)
├── pubkey_binding_proof : 64 bytes (Ed25519 sig of epoch_pubkey by mp_master_privkey)
└── signature            : 32 bytes (Ed25519 sig by mp_epoch_privkey over all preceding fields)
```

The `pubkey_binding_proof` allows a subscriber who knows `mp_master_pubkey` to verify that the `epoch_pubkey` is legitimately derived from the master key, without the master key appearing in the DHT record itself.

---

## MP Resolution

```python
def RESOLVE_MP(mp_master_pubkey: bytes) -> ACI:

    # Compute DHT key for current epoch
    MP_DHT_key = BLAKE3(mp_master_pubkey || "I2PFS-MP-DHT" || uint64_le(current_epoch))

    # Run DHT lookup
    records = DHT.GET_VALUE(MP_DHT_key)

    valid_records = []
    for record in records:
        # Check epoch (±1 for transition tolerance)
        if abs(record.epoch - current_epoch) > 1:
            continue
        # Verify epoch_pubkey is legitimately bound to master pubkey
        if not Ed25519_verify(mp_master_pubkey, record.epoch_pubkey, record.pubkey_binding_proof):
            continue
        # Verify record signature
        if not Ed25519_verify(record.epoch_pubkey, record.all_preceding_fields, record.signature):
            continue
        valid_records.append(record)

    # Last-writer-wins: return the ACI from the highest valid seq
    best = max(valid_records, key=lambda r: r.seq)
    return best.root_aci
```

**Epoch transition handling:** During the 48-hour epoch change, both the old and new epoch's `MP_DHT_key` are valid. Resolvers SHOULD check both keys and merge results by `seq` number.

---

## MP Update

```python
def UPDATE_MP(mp_master_privkey: bytes, new_root_aci: ACI):

    # Step 1: Fetch current MP to get latest seq
    latest_seq = RESOLVE_MP(corresponding_pubkey).seq if exists else 0
    new_seq = latest_seq + 1

    # Step 2: Derive epoch keys
    mp_epoch_privkey = derive_epoch_privkey(mp_master_privkey, current_epoch)
    mp_epoch_pubkey  = corresponding_pubkey
    pubkey_binding_proof = Ed25519_sign(mp_master_privkey, mp_epoch_pubkey)

    # Step 3: Build and sign record
    record = MP_VALUE(
        dht_key              = MP_DHT_key(mp_master_pubkey, current_epoch),
        epoch                = current_epoch,
        seq                  = new_seq,
        root_aci             = new_root_aci,
        timestamp            = now_ms(),
        ttl                  = 172800,
        epoch_pubkey         = mp_epoch_pubkey,
        pubkey_binding_proof = pubkey_binding_proof,
        signature            = Ed25519_sign(mp_epoch_privkey, all_preceding_fields)
    )

    # Step 4: Publish to DHT
    DHT.PUT_VALUE(MP_DHT_key, record)
    # DHT nodes store update only if new_seq > stored_seq for same dht_key
```

**Replay prevention:** Nodes MUST reject `PUT_VALUE` for a `dht_key` if the incoming `seq` ≤ stored `seq`. This prevents an attacker from rolling back content by replaying an old record.

---

## MP Anonymity Properties

| Property | Mechanism |
|----------|-----------|
| Publisher not in DHT record | DHT record contains only `epoch_pubkey` and `pubkey_binding_proof`; master privkey never on wire |
| DHT key rotates per epoch | `MP_DHT_key[epoch]` changes every 48 h; multi-epoch observation required to link records |
| MP updates are anonymous | Updates sent via Publisher SAM session, which has no linkage to DHT identity |
| Content updates don't correlate | Each update points to a new root ACI with fresh encryption keys; content itself cannot be linked across updates |
