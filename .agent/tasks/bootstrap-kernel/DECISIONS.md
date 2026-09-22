# Decisions

- Phase 0/1 is a synchronous single-process reference kernel.
- Exactly one CausalIsland is used in the reference kernel.
- No async runtime is required in Phase 0/1.
- Rust Edition 2024, stable compiler pinned at bootstrap.
- Initial crates: tickweave-semantic, tickweave-kernel, tickweave-testkit.
- Unsafe code is forbidden in initial crates.
- Canonical Encoding v1 is custom semantic encoding, not Serde/bincode/protobuf.
- BLAKE3-256 with versioned TickWeave domains is the semantic hash primitive.
- Resource absence is versioned; revisions never reset on delete/recreate.
- Write access implies transactional read capability to the same key.
- Write preconditions are ExactRevision or Current.
- Stable semantic IDs use schema-defined EmissionPath, never scheduling counters.
- Cell, authority, CausalIsland, and Worker are distinct concepts.
- Phase 0/1 does not implement distributed authority or safe-time protocols.
