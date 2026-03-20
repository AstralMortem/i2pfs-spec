# I2P Network Constraints

All I2PFS design decisions are constrained by I2P's operational characteristics. Implementors **MUST** internalize these parameters before implementing any subsystem. The single most common failure mode in I2P protocol design is ignoring these constraints and designing for clearnet assumptions.

---

## Network Parameters

| Parameter | Typical | Worst Case | Design Implication |
|-----------|---------|------------|-------------------|
| Round-Trip Time | 500–800 ms | 2000+ ms | Minimum RPC timeout: 30 s. All I/O must be non-blocking. |
| Tunnel Lifetime | 10 minutes | 10 minutes (hard) | No connection state may outlive a tunnel. All requests must be idempotent. |
| Tunnel Build Time | 5–30 s | 60 s | Tunnel pools must be pre-warmed. Tunnel build latency must not appear in operation latency. |
| Raw I2NP MTU | ~1 KB | 64 KB | Protocol messages larger than 1 KB require I2NP-layer fragmentation; I2PFS frames should fit in ≤ 2 I2NP messages. |
| Streaming MTU | ~60 KB | ~64 KB | Application-layer frames must not exceed 48 KB to leave headroom for I2P framing overhead. |
| Effective Bandwidth | 8–32 KB/s | 2 KB/s | Conservative flow control. Pipeline depth must be limited to avoid tunnel saturation. No burst assumptions. |
| Peer Reachability | 60–70% | ~40% | Replication factor of 5 is the minimum for acceptable availability. |
| NetDB Lookup | 5–20 s | 45 s | I2P destination resolution is slow; destinations must be cached aggressively. |

---

## Unidirectional Tunnel Asymmetry

I2P tunnels are strictly **unidirectional**. Every logical "connection" between two nodes consists of two independent tunnel paths:

- **Outbound tunnel** — built by the local node, used for sending
- **Inbound tunnel** — built by the local node, published as a Lease Set, used for receiving responses

The critical implication: a message can be successfully sent via an outbound tunnel while the response fails because the sender's inbound tunnel has expired or failed. I2PFS handles this by:

1. **Idempotent requests** — retransmitting a request always produces the same result; there is no risk of double-processing
2. **Separated failure handling** — an outbound send failure means retransmit on the same tunnel; an inbound response timeout means retry on a different provider or rebuild the inbound tunnel pool
3. **No delivery assumptions** — a successful send never implies a successful receive

---

## Garlic Routing and Message Bundling

I2P uses **garlic routing** — multiple individually encrypted messages (called *cloves*) are bundled into a single encrypted garlic message and delivered in one tunnel traversal. The marginal cost of adding a small clove to an existing garlic message is zero in terms of additional tunnel hops.

I2PFS exploits garlic routing by:

- Bundling DHT responses with block data when both are ready simultaneously
- Batching multiple `WANT_BLOCK` requests for different chunks into a single outbound message to the same provider
- Piggybacking `ACK` messages and `PROVIDER` announcements onto data response messages
- Attaching a Lease Set delivery clove to the first message in a new session, eliminating a separate NetDB lookup round trip

!!! important "Maximum garlic batch size"
    **48 KB** — leaving 12 KB headroom within the 60 KB streaming MTU for I2P framing and per-clove overhead.

---

## SAM Bridge Constraints

I2PFS communicates with I2P exclusively via the **SAM (Simple Anonymous Messaging) bridge v3.3+**. SAM imposes its own constraints:

- SAM streaming sessions multiplex over a single TCP connection to the SAM bridge; high message rates can saturate this connection independently of tunnel bandwidth.
- Each SAM session has a single I2P Destination. Multiple independent Destinations require multiple SAM sessions — which is the correct approach for I2PFS's identity separation model (see [Node Identity & Anonymity](identity-anonymity.md)).
- Datagram sessions via SAM have a **31 KB payload limit** per datagram.
- SAM session creation itself takes 2–10 s; sessions must be created in advance and pooled.

### Required SAM Capabilities

```
STREAM CONNECT                               outbound TCP-like connections
STREAM ACCEPT                                inbound connection acceptance
DATAGRAM SEND / RECV (repliable)             for DHT UDP-like RPCs
DEST GENERATE style=SIGNATURE_TYPE=7         Ed25519 destination generation
  with crypto=ECIES-X25519-AEAD-Ratchet      integrated encryption keys
LOOKUP name=<b64dest>                        NetDB resolution
SESSION CREATE STYLE=STREAM                  streaming session
SESSION CREATE STYLE=DATAGRAM                datagram session
SESSION SET DESTINATION=TRANSIENT            per-retrieval ephemeral destinations
BLINDING GENERATE                            blinded Lease Set keys
```

---

## Why These Constraints Drive Every Decision

| Constraint | Protocol consequence |
|---|---|
| 500–2000 ms RTT | All DHT timeouts ≥ 30 s; async-first design mandatory throughout |
| 10-minute tunnel lifetime | Sessions are stateless and idempotent; no long-lived connection state |
| 8–32 KB/s bandwidth | 32 KB chunk size; pipeline window ≤ 6; garlic batching mandatory |
| 60–70% peer reachability | Replication factor R=5 as minimum; proactive push (not passive announce) |
| Unidirectional tunnels | Per-retrieval session rotation; retry-safe request design |
| No clearnet | All peer discovery via DHT only; no mDNS, no bootstrap servers |
