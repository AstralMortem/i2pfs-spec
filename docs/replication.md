# Replication & Fault Tolerance

---

## Replication Model

I2PFS uses **proactive push replication**: when content is published, it is actively transferred to multiple independent provider nodes before any consumer has requested it. This is necessary on I2P because:

- I2P nodes are frequently offline (60–70% reachability)
- Publishers may go offline immediately after publishing
- Passive announce-based replication (like IPFS) leaves content unavailable until someone has fetched and re-served it

### Target Replication Factor: R = 5

Statistical basis for R=5:

| Availability per node | P(at least 1 of 5 available) |
|---|---|
| 65% | 99.47% |
| 50% | 96.88% |
| 40% | 92.22% |

R=3 is the **minimum acceptable** floor (`REPLICATION_MIN`). R=5 is the target. The replication system will attempt to restore to R=5 whenever it detects fewer than R=3 providers.

---

## Replica Selection Algorithm

Replicas are selected to maximize structural independence across the I2P network:

```python
def SELECT_REPLICAS(aci: ACI, R: int = 5, candidate_pool: list) -> list:

    # candidate_pool = k=20 nearest DHT peers to ACI's DHT_key
    # (obtained as a side effect of FIND_PROVIDERS, even if no providers exist yet)

    def score(candidate, already_selected):
        base = (
            0.4 * uptime_score(candidate) +           # prefer recently seen nodes
            0.4 * xor_diversity(candidate, already_selected) +  # spread across keyspace
            0.2 * router_family_diversity(candidate, already_selected)  # different I2P families
        )
        jitter = CSPRNG_float() * 0.30 - 0.15   # ±15% random jitter
        return base * (1 + jitter)

    selected = []
    while len(selected) < R and candidate_pool:
        best = max(candidate_pool, key=lambda c: score(c, selected))
        selected.append(best)
        candidate_pool.remove(best)

    return selected
```

**Selection diversity constraints:**

- No two replicas within XOR distance 2^240 of each other in the DHT keyspace
- No two replicas sharing the same estimated I2P router family (estimated from first 8 bytes of SAM destination hash)

**The ±15% random jitter** prevents fingerprinting: an observer who knows the scoring algorithm cannot predict exactly which 5 nodes were selected and use that to identify the publisher.

---

## Block Push Procedure

For each selected replica node:

```python
for replica in selected_replicas:
    # 1. Establish AnonSwap session (Noise_IKpsk2 handshake)
    session = AnonSwap.connect(replica, content_dht_key, session_id=CSPRNG(16))

    # 2. Send root block (single SEND_BLOCK frame)
    session.send_block(root_aci, root_block_data)

    # 3. Send all leaf chunks (pipelined, window ≤ 6)
    for i, chunk in enumerate(leaf_chunks):
        session.send_block(leaf_aci[i], chunk_data, chunk_index=i)
        # Pipeline: don't wait for ACK before sending next (up to SEND_WINDOW)

    # 4. Wait for all ACKs
    session.await_all_acks(timeout=PUBLICATION_TIMEOUT)

    # 5. On full acknowledgment: replica publishes PROVIDER record in DHT
    # (replica does this autonomously after verifying all chunks received)
```

If fewer than R replicas acknowledge all chunks within `PUBLICATION_TIMEOUT` (300 s), retry with next-best candidates from the pool. Log a warning if final replication count < `REPLICATION_MIN` (3).

---

## Replication Maintenance

Provider nodes take active responsibility for content health:

| Task | Trigger | Action |
|------|---------|--------|
| **TTL re-announcement** | At 20 h (4 h before 24 h TTL expires) | Re-announce PROVIDER record in DHT |
| **Epoch transition** | At each 6-hour epoch boundary ±30 min jitter | Re-announce all stored content under new epoch's DHT_key |
| **Under-replication detection** | FIND_PROVIDERS returns < REPLICATION_MIN (3) providers | Trigger repair protocol |
| **Node departure detection** | Lease Set no longer in NetDB | Trigger repair protocol |

The ±30 minute jitter on epoch transition announcements prevents synchronized announcement storms where all providers announce simultaneously.

---

## Repair Protocol

```python
def REPAIR(aci: ACI, stored_chunks: list):

    # Step 1: Verify local chunks are intact
    for i, chunk in enumerate(stored_chunks):
        actual_hash = BLAKE3(chunk.ciphertext || chunk.tag)
        if actual_hash != leaf_aci[i].content_hash:
            delete(chunk)                # do not propagate corrupt data
            log_warning(f"chunk {i} failed integrity check; deleted")

    # Step 2: Find new candidate nodes not already serving the content
    provider_list = FIND_PROVIDERS(dht_key)
    existing_providers = {p.provider_token for p in provider_list}
    new_candidates = SELECT_REPLICAS(aci, candidates=[c for c in k_nearest if c not in existing_providers])

    # Step 3: Push missing content to new nodes
    for candidate in new_candidates:
        session = AnonSwap.connect(candidate, ...)
        for chunk in verified_chunks:
            session.send_block(...)

    # Step 4: Verify recovery
    assert FIND_PROVIDERS(dht_key) >= REPLICATION_MIN

# Repair runs asynchronously; does not block normal content serving
```

---

## Pinning

Content can be pinned to prevent garbage collection:

| Pin Type | Scope | GC Behavior |
|----------|-------|-------------|
| `RECURSIVE` | Root ACI + all leaf chunk ACIs recursively | Never GC'd; always re-announced |
| `DIRECT` | Only the specified ACI block | Block itself not GC'd; leaf chunks may be |
| `INDIRECT` | Automatically applied to leaf chunks of a RECURSIVE-pinned root | Never independently GC'd |

Unpinned content is garbage collected after `GC_UNPINNED_AFTER_DAYS` (default: 7 days) without a provider re-announcement from any peer requesting that block.

---

## Garbage Collection

```python
def GC_CYCLE():
    cutoff = now() - GC_UNPINNED_AFTER_DAYS * 86400

    for block in block_store.all_blocks():
        if block.is_pinned():
            continue
        if block.last_announced < cutoff and block.last_served < cutoff:
            block_store.delete(block.aci)
            dht.retract_provider_record(block.aci)
```

GC runs as a background task at P4 (lowest) priority and is paused entirely when tunnel rebuild rate exceeds 2/minute (congestion indicator).
