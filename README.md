# I2PFS — Anonymous Distributed File System for I2P

**I2PFS** (Invisible Internet Project File System) is a decentralized, content-addressed distributed file system designed exclusively for the I2P anonymous overlay network. It provides computationally unlinkable sender and recipient anonymity, per-chunk end-to-end encryption with forward secrecy, fault-tolerant distributed storage, and a block exchange protocol optimized for I2P's high-latency, bandwidth-constrained unidirectional tunnel architecture.

I2PFS is designed from cryptographic and network primitives upward with anonymity and I2P's operational envelope as first-class constraints. Every protocol design decision is evaluated against I2P's 500–2000 ms round-trip times, 10-minute tunnel lifetimes, 8–32 KB/s effective bandwidth, and the fundamental requirement that no operation reveals publisher or consumer identity to any observer, including other I2PFS nodes.

See full specification:
- [SPEC v1.0](/I2PFS.md)