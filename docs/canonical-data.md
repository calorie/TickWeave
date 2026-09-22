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


## 14. Core semantic ABI v1

The Phase 0/1 kernel uses a generic core ResourceKey rather than freezing Minecraft-specific enum tags too early.

```rust
struct ResourceNamespaceId(u32);
struct ResourceKindId(u32);
struct StateSchemaId(u32);

struct CanonicalBytes(Vec<u8>);

struct ResourceKey {
    namespace: ResourceNamespaceId,
    kind: ResourceKindId,
    key: CanonicalBytes,
}

struct StateValue {
    schema: StateSchemaId,
    bytes: CanonicalBytes,
}
```

Canonical ResourceKey order is lexicographic over the canonical encoding of `(namespace, kind, key)`.

Phase 2 typed Minecraft keys are constructors/adapters over this semantic key space. They may not change the v1 ordering of already-defined keys.

A `CanonicalBytes` value is opaque to the kernel. Its owning schema is responsible for validating canonical payload/state bytes.

## 15. Schema identifiers

The bootstrap semantic ABI uses:

```rust
struct OperationSchemaId {
    namespace: u32,
    operation: u32,
    version: u32,
}

struct EventSchemaId {
    namespace: u32,
    event: u32,
    version: u32,
}
```

Ordering is lexicographic by fields in declaration order.

StateSchemaId is a u32 identifier inside a Ruleset. A future cross-project registry may expand this only under a new ABI version.

## 16. Bootstrap RulesetManifest v1

The Phase 0/1 manifest is canonically encoded in this field order:

```rust
struct RulesetManifestV1 {
    semantics_version: u32,
    canonical_encoding_version: u32,
    numeric_profile_version: u32,
    rng_profile_version: u32,
    max_reactive_rounds: u32,
    max_reactive_events_per_tick: u64,
    operation_schemas: Vec<OperationSchemaId>,
    event_schemas: Vec<EventSchemaId>,
    state_schemas: Vec<StateSchemaId>,
    gameplay_config: CanonicalBytes,
}
```

Schema vectors are sorted canonically and contain no duplicates.

For bootstrap fixtures, all version fields are `1` unless a test explicitly exercises version differences.

## 17. Deterministic RNG v1

The bootstrap RNG host primitive is:

```text
random_u64(
  world_seed,
  intent_id,
  operation_member_index,
  stream_id,
  invocation_index
)
```

where `world_seed` is exactly 32 bytes and integer fields use Canonical Encoding v1.

Compute:

```text
digest = H(
  "rng/v1",
  Canonical(
    world_seed,
    intent_id,
    operation_member_index:u32,
    stream_id:u32,
    invocation_index:u32
  )
)
```

Return the first 8 digest bytes interpreted as little-endian u64.

No mutable RNG cursor exists outside the explicit invocation index.

## 18. Tie-break v1

`world_order_seed` is exactly 32 authoritative bytes.

For an Intent at ResolutionPoint `(tick,wave)`:

```text
tie_break =
  H(
    "intent-order/v1",
    Canonical(
      world_order_seed,
      tick:u64,
      wave:u32,
      intent_id
    )
  )
```

Compare the 32-byte tie-break values lexicographically. IntentId bytes are the collision fallback.

## 19. Identity collision rule

If an already-known IntentId/EventId/ObjectId is encountered again:

- identical canonical semantic content is an idempotent duplicate only where that ID class explicitly allows retry/dedup;
- differing canonical content under the same ID is a RuntimeDeterminismFault;
- duplicate emission of the same newly-generated ID inside one cause is a RuntimeDeterminismFault even if bytes match.

Cryptographic collision is not silently resolved by generating another ID.

## 20. Event admission ledger

Exactly-once SimulationEvent semantics requires durable knowledge of admitted EventIds.

The Phase 0/1 reference kernel retains an unbounded canonical EventAdmissionLedger.

An entry binds EventId to the hash of its canonical event content.

Repeated identical physical delivery is ignored semantically. Same EventId with different content is a RuntimeDeterminismFault.

The ledger is included in the reference replay state and canonical state hash because it can change the result of future duplicate delivery.

Production GC is deferred. A later implementation may delete ledger entries only after a proven transport/durability watermark makes recurrence impossible.

## 21. Overflow

Revision, TickId, WaveId, sequence, and canonical length arithmetic use checked operations.

Semantic counters never wrap.

Exhaustion/overflow is a RuntimeDeterminismFault (or process-fatal equivalent before any affected commit), not modulo arithmetic.
