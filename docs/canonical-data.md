# Canonical Data, Hashing, Revision, and Identity Semantics

Status: **Authoritative v0.1**

## 1. Purpose

Canonical representation is part of TickWeave semantics. Rust memory layout, Serde format, database encoding, protobuf field order, and network frame encoding are not authoritative by themselves.

Phase 0 implements this specification directly.

## 2. Canonical Encoding v1

All authoritative canonical values implement an explicit `CanonicalEncode` contract.

Primitive encoding:

| Type | Encoding |
|---|---|
| `u8` | one byte |
| `u16/u32/u64/u128` | fixed-width little-endian |
| signed integers | fixed-width two's-complement little-endian |
| `bool` | exactly `0x00` or `0x01` |
| enum tag | `u32` little-endian followed by variant fields |
| `Option<T>` | one-byte tag 0/1, then value when present |
| bytes | `u64` byte length, then raw bytes |
| UTF-8 text | `u64` byte length, then exact UTF-8 bytes |
| sequence | `u64` element count, then elements |
| struct | fields in schema-defined order, no field names |

Variable-length integer encodings are not used in Canonical Encoding v1.

Floating-point values are forbidden in authoritative canonical state.

Strings are byte-exact UTF-8. Canonical encoding performs no Unicode normalization. Semantic schemas that need normalized identifiers must normalize before constructing the canonical value.

## 3. Maps and sets

Canonical maps are sorted by the canonical encoded bytes of their keys, lexicographically ascending.

Duplicate canonical keys are invalid.

Canonical sets use the same canonical-byte ordering.

Semantic behavior MUST NOT depend on hash-map iteration order.

## 4. Decoder rule

A CanonicalDecode implementation rejects invalid tags, duplicate or unsorted map/set keys, invalid UTF-8 for text, trailing bytes for a complete value, and schema-invalid representations.

For every accepted complete value:

```text
encode(decode(bytes)) == bytes
```

## 5. Hash primitive

Semantic hashes use BLAKE3-256.

```text
H(domain, payload) =
  BLAKE3(
    "TickWeave\0"
    || domain
    || "\0"
    || payload
  )
```

Domain strings are ASCII and versioned, for example:

```text
state/v1
resource-key/v1
intent/v1
event/v1
entity/v1
ruleset/v1
intent-order/v1
```

Changing canonical encoding or hash framing requires a new version/domain.

## 6. Resource entry and revision

Every authoritative ResourceKey has logical history:

```rust
struct ResourceEntry {
    revision: Revision,
    value: Option<StateValue>,
}
```

Absence is versioned.

Rules:

- a successful transaction that semantically changes the key increments revision exactly once;
- multiple writes to the same key inside one AtomicIntent still increment once;
- rejection increments nothing;
- deletion produces `value = None` and increments revision;
- recreation increments revision again;
- revision MUST NOT reset on delete/recreate.

```text
absent rev=10
create -> present rev=11
delete -> absent rev=12
create -> present rev=13
```

Storage may compact tombstones only if it preserves equivalent monotonic version history sufficient to reject stale operations.

## 7. Write implies read capability

Declaring write access to ResourceKey K implicitly grants transactional read access to K.

Access-plan normalization removes read entries already covered by writes.

A resolver still may not access any unrelated undeclared ResourceKey.

## 8. Write precondition model

```rust
struct WriteAccess {
    key: ResourceKey,
    precondition: WritePrecondition,
}

enum WritePrecondition {
    ExactRevision(Revision),
    Current,
}
```

`ExactRevision(R)` requires the key's transaction-entry provisional revision to equal R.

`Current` has no revision precondition and resolves against the latest provisional value produced by earlier canonical transactions.

All conflicting transactions are semantically executed in canonical serial order.

For duplicate write keys inside one AtomicIntent:

- Current + Current -> Current;
- ExactRevision(R) + Current -> ExactRevision(R);
- ExactRevision(R) + ExactRevision(R) -> ExactRevision(R);
- different ExactRevision values -> invalid Intent.

Revision increments occur only at whole-transaction commit, not between AtomicIntent members.

## 9. Stable semantic IDs

Simulation-generated IDs are deterministic hashes, not random UUIDs.

```text
IntentId = H("intent/v1", Canonical(cause_id, emission_path))
EventId  = H("event/v1",  Canonical(cause_id, emission_path))
EntityId = H("entity/v1", Canonical(creating_intent_id, spawn_path))
```

No semantic ID may include WorkerId, CellId, process ID, thread ID, wall clock, or system RNG.

## 10. EmissionPath

Parallel execution must not allocate semantic IDs using an atomic counter or physical append order.

```rust
struct EmissionPath(Vec<u32>);
```

Path components identify schema-defined semantic emission sites and deterministic ordinals.

For repeated collections, first establish canonical semantic order, then assign ordinals.

Within AtomicIntent event emission, the path includes at least the operation member index and operation-local emission path.

Duplicate IDs generated within one cause are a RuntimeDeterminismFault.

## 11. RulesetId

```rust
struct RulesetId([u8; 32]);
```

RulesetId hashes a canonical RulesetManifest containing at least:

- simulation semantics version;
- canonical encoding version;
- authoritative numeric profile version;
- deterministic RNG algorithm/version;
- reactive bounds;
- enabled Operation/Event schema IDs and versions;
- gameplay rule configuration affecting authoritative results.

Thread count, Cell size, scheduler thresholds, compression, logging, and node price are excluded.

## 12. Canonical state hash

For a logical committed cut, reference state hashing includes:

- WorldId;
- RulesetId;
- authoritative seeds;
- logical cut descriptor;
- all authoritative ResourceKey entries, including revisions and retained absence/version metadata;
- pending authoritative scheduled work;
- admitted future SimulationEvents required for replay.

Entries are sorted by canonical key/ID bytes.

Excluded:

- CellId and Cell membership;
- WorkerId;
- owner placement;
- CausalIsland identity;
- transport/routing state;
- halo/ghost caches;
- derived indexes;
- metrics/logging;
- OS/process/thread state.

Distributed implementations may use Merkle/incremental hashes only if the resulting logical root is equivalent to the reference canonical hash.

## 13. Phase 0 conformance vectors

Phase 0 must commit golden byte/hash vectors covering all primitive/container rules, ResourceKey ordering, ResourceEntry tombstones, stable ID derivation, RulesetId, and one complete state hash.

These vectors are compatibility fixtures and must not be rewritten merely because implementation changes.
