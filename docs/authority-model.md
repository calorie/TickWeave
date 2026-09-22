# Authority, Fencing, Cell, and Entity Residency Model

Status: **Authoritative v0.1**

## 1. Separate the concepts

```text
Resource / Chunk Authority = who may commit writes
Cell                       = placement/migration grouping
CausalIsland               = safe-time synchronization domain
Worker                     = physical compute
```

They MUST NOT be collapsed into one identity.

## 2. Chunk authority

Chunk is the minimum spatial authority unit.

```rust
struct ChunkAuthorityRecord {
    chunk: ChunkKey,
    generation: u64,
    owner: WorkerId,
}

struct ChunkFence {
    chunk: ChunkKey,
    generation: u64,
}
```

Generation is monotonic for that ChunkKey.

A write to a resource anchored to a chunk is accepted only under the current generation. A stale generation is always rejected.

## 3. Cell is not authority

A Cell is an ephemeral runtime grouping of a connected set of Chunks.

Split or merge on the same Worker does not itself change chunk authority generations.

Moving a chunk to another Worker increments that chunk's authority generation as part of handoff.

Implementations may physically compress equal records, but semantics are per ChunkKey.

CellId is never a fencing token.

## 4. Non-spatial authority

```rust
enum AuthorityDomainKey {
    Chunk(ChunkKey),
    Named(NamedAuthorityKey),
}
```

Global/plugin/account-like state must choose a deliberate NamedAuthorityKey/sharding scheme rather than accidentally creating one global lock.

The production control-plane representation is deferred until Phase 4; this semantic distinction is fixed.

## 5. Resource authority mapping

Every authoritative ResourceKey schema defines a deterministic authority-anchor function.

Examples:

- Block / BlockEntity -> containing ChunkKey.
- Scheduled block work -> target ChunkKey.
- Entity component -> entity's committed home ChunkKey.
- Block container -> containing ChunkKey.
- Entity container -> owning entity's committed home ChunkKey.
- WorldState / PluginState -> explicit NamedAuthorityKey.

Authority mapping is runtime/control metadata and does not add Worker/Cell identity to gameplay state.

## 6. Entity home chunk

An entity's home chunk is:

```text
home_chunk(entity, committed_state)
  = chunk_containing(entity.anchor_position)
```

It is a pure derived value, not an independently mutable gameplay field.

All entity components for the next simulation step are anchored to that home chunk.

## 7. Crossing a chunk boundary

If Tick T begins with Entity E anchored to Chunk A and commits a position inside Chunk B:

1. Tick-T transaction is authorized under E's Tick-T authority anchor.
2. Committed position determines next home chunk B.
3. Before E participates in subsequent simulation, runtime residency/authority handoff to B must complete.
4. Same-Worker handoff may be a local fast path.
5. Cross-Worker handoff changes fencing/routing before subsequent authoritative mutation.

EntityId never changes.

Failure during handoff may delay E but cannot duplicate it or permit two current writers.

## 8. Entity directory

`EntityId -> home ChunkKey` may be maintained as a derived routing index.

It is reconstructible and excluded from canonical state hashing.

A stale lookup results in redirect/retry under fencing, never an accepted stale write.

## 9. Chunk migration

```text
A owns chunk C generation G
 -> copy/catch up state
 -> establish cutover boundary
 -> owner=B, generation=G+1 committed
 -> A loses authority
 -> B may commit with (C,G+1)
```

Owner/generation update is strongly consistent control metadata. Data copy may precede cutover.

## 10. Split and merge

Cell split/merge only changes placement grouping.

It must not rewrite gameplay resources or affect canonical state hashes.

If the operation also moves Chunks between Workers, moved Chunks use the ordinary generation transition.

## 11. Phase boundaries

Phase 0/1:
- one in-process authority;
- no distributed ownership implementation;
- semantic ResourceKey types contain no WorkerId/CellId.

Phase 2:
- ChunkKey and deterministic resource-authority mapping.

Phase 3:
- Cells as local placement groups, without changing authority semantics.

Phase 4:
- real Worker ownership, generations, fencing, and control-plane protocol.

## 12. Safety invariant

For every authority domain:

```text
at most one current generation can successfully commit
and
accepted write generation == current authority generation
```
