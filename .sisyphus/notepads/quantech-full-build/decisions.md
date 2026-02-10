# Decisions — Quantech Full Build

## Architectural Decisions
- bbolt over RocksDB: eliminates CGo complexity for MVP
- cgofuse over bazil.org/fuse: single API for Linux+macOS+Windows
- BIP39 standard wordlist for recovery keys
- Immutable chunks (content-addressed) for safe deterministic nonces
- JSON for metadata serialization (no protobuf)
