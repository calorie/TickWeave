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
| fixed-size array | elements in order, no length prefix |
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

A key that has never been materialized is implicitly `ResourceEntry { revision: 0, value: None }` and need not be stored.

Absence after the first accepted write/delete is versioned and retained logically.

Rules:

- a key on which an accepted transaction invokes write/delete increments revision exactly once, even if the final value bytes equal the prior value;
- multiple writes/deletes to the same key inside one AtomicIntent still increment once;
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


## 22. Generic semantic addresses

Phase 0/1 avoids freezing Minecraft-specific address enum tags.

```rust
struct SemanticAddress {
    namespace: u32,
    kind: u32,
    key: CanonicalBytes,
}

struct OriginRef {
    source: SemanticAddress,
    stream: u32,
    sequence: u64,
}
```

Canonical address order is lexicographic over `(namespace, kind, key)`.

Typed Player/Entity/Block/System/EventTarget wrappers introduced later construct SemanticAddress values; they do not alter core encoding.

## 23. CauseRef and intent identity tags v1

```rust
enum CauseRef {
    ExternalInput(InputId),          // tag 0
    SimulationEvent(EventId),       // tag 1
    Scheduled(ScheduleId),          // tag 2
    Intent(IntentId),               // tag 3
    AtomicIntent(AtomicIntentId),   // tag 4
    System(SemanticAddress),        // tag 5
}

enum AnyIntentId {
    Mutation(IntentId),             // tag 0
    Atomic(AtomicIntentId),         // tag 1
}
```

The comments define fixed Canonical Encoding v1 enum tags.

All fixed-size 16/32-byte ID newtypes encode as their raw bytes with no length prefix.

## 24. Operation and event envelope v1

```rust
struct Operation {
    schema: OperationSchemaId,
    payload: CanonicalBytes,
}

struct MutationIntent {
    id: IntentId,
    world: WorldId,
    point: ResolutionPoint,
    cause: CauseRef,
    origin: OriginRef,
    operation: Operation,
}

struct AtomicIntent {
    id: AtomicIntentId,
    world: WorldId,
    point: ResolutionPoint,
    cause: CauseRef,
    origin: OriginRef,
    operations: Vec<Operation>, // canonical validation requires non-empty
}

struct SimulationEvent {
    id: EventId,
    world: WorldId,
    cause: CauseRef,
    source: SemanticAddress,
    target: SemanticAddress,
    deliver_at: ResolutionPoint,
    schema: EventSchemaId,
    payload: CanonicalBytes,
}
```

Generic Guard values are not fields of the v1 Intent envelope. Schema-specific predicates are encoded in operation payloads and reflected in declared transactional reads/write preconditions. Host libraries may provide typed guard helpers.

## 25. EventDraft v1

A resolver emits an EventDraft, not a pre-built EventId:

```rust
struct EventDraft {
    emission_path: EmissionPath,
    source: SemanticAddress,
    target: SemanticAddress,
    deliver_at: ResolutionPoint,
    schema: EventSchemaId,
    payload: CanonicalBytes,
}
```

The runtime sets `cause` from the accepted parent transaction and derives EventId from the parent cause/transaction identity plus the deterministic emission path.

An EventDraft targeting the same Tick must satisfy `deliver_at.wave >= current_wave + 1`.

## 26. Reference WorldSnapshotV1

The Phase 0/1 canonical state-hash payload is exactly:

```rust
struct WorldSnapshotV1 {
    world: WorldId,
    ruleset: RulesetId,
    world_seed: [u8; 32],
    world_order_seed: [u8; 32],
    state_tick: TickId,
    resources: Vec<(ResourceKey, ResourceEntry)>,
    pending_events: Vec<SimulationEvent>,
    event_admission_ledger: Vec<(EventId, [u8; 32])>,
}
```

Canonical validation requires:

- resources sorted by ResourceKey with no duplicate key;
- pending_events sorted by `(deliver_at.tick, deliver_at.wave, EventId)`;
- ledger sorted by EventId with no duplicate ID.

`state_tick = T` means the snapshot is S[T], before executing Tick T.

After committing Tick T, the resulting snapshot has `state_tick = T + 1`.

The reference state hash is:

```text
H("state/v1", Canonical(WorldSnapshotV1))
```

For Phase 0/1, future scheduled semantic work is represented by pending SimulationEvents or explicit ResourceEntries; there is no additional hidden scheduled-work collection.

## 27. Event content binding

The EventAdmissionLedger value for EventId E is:

```text
H("event-content/v1", Canonical(SimulationEvent without id))
```

On duplicate delivery:

- same EventId + same content hash -> idempotent duplicate;
- same EventId + different content hash -> RuntimeDeterminismFault.


## 28. Intent derivation v1

When a semantic cause emits transaction drafts:

```text
IntentId =
  H(
    "intent/v1",
    Canonical(cause_ref, emission_path)
  )

AtomicIntentId =
  H(
    "atomic-intent/v1",
    Canonical(cause_ref, emission_path)
  )
```

When an accepted transaction emits an EventDraft:

```text
parent_cause =
  CauseRef::Intent(parent IntentId)
  or
  CauseRef::AtomicIntent(parent AtomicIntentId)

EventId =
  H(
    "event/v1",
    Canonical(parent_cause, emission_path)
  )
```

The materialized SimulationEvent stores that `parent_cause` in its `cause` field.

Event evaluation creates Intent/AtomicIntent drafts whose CauseRef is `SimulationEvent(event.id)`.

## 29. Happens-before v1

Within one ResolutionPoint, the bootstrap happens-before graph has one cross-transaction ordering rule:

```text
if
  A.origin.source == B.origin.source
  and A.origin.stream == B.origin.stream
  and A.origin.sequence < B.origin.sequence
then
  A happens-before B
```

Only transactions in the same conflict component need the edge represented explicitly.

Transactions with equal origin sequence are concurrent and use the deterministic tie-break.

AtomicIntent member vector order is internal transaction order, not separate graph nodes.

Cross-Wave causality is represented by ResolutionPoint ordering rather than an extra same-Wave edge.

## 30. Bootstrap reactive limits

For RulesetManifestV1:

- `max_reactive_rounds == 0` means no configured round limit;
- `max_reactive_events_per_tick == 0` means no configured event-count limit.

The Phase 0/1 bootstrap Ruleset uses zero for both fields.

Finite overflow/carry semantics are intentionally deferred to a later ruleset version before untrusted gameplay/plugin execution. Phase 0/1 test schemas must terminate.

## 31. Bootstrap test schema IDs

Use this reserved test namespace:

```text
TEST_NAMESPACE = 0xFFFF0001
```

Operation IDs:

```text
1 CreateValue
2 DeleteValue
3 PutExact
4 AddCurrent
5 TransferExact
6 EmitOnSuccess
7 SpawnStableObject
```

Event IDs:

```text
1 TestEvent
```

State schema IDs:

```text
1 SignedI64
2 Bytes
```

All versions are 1.

These identifiers are conformance fixtures, not Minecraft production schema allocation.


## 32. Transaction receipt v1

Bootstrap conformance receipts use:

```rust
struct SemanticErrorCode(u32);

enum RejectReason {
    GuardFailed,                         // tag 0
    RevisionMismatch,                   // tag 1
    ResourceMissing,                    // tag 2
    ResourceAlreadyExists,              // tag 3
    SemanticRuleViolation(SemanticErrorCode), // tag 4
}

enum TransactionStatus {
    Accepted,                           // tag 0
    Rejected(RejectReason),             // tag 1
}

struct ResourceDelta {
    key: ResourceKey,
    before: ResourceEntry,
    after: ResourceEntry,
}

struct TransactionReceipt {
    intent: AnyIntentId,
    status: TransactionStatus,
    state_delta_hash: [u8; 32],
    emitted_event_ids: Vec<EventId>,
}
```

For receipt construction:

- ResourceDelta entries are sorted by ResourceKey;
- `state_delta_hash = H("delta/v1", Canonical(Vec<ResourceDelta>))`;
- emitted_event_ids are sorted by EventId;
- rejected transactions use the hash of an empty delta vector and emit no EventIds.

Receipts are conformance artifacts, not authoritative world resources.

## 33. State schema validation

The bootstrap registry includes a deterministic state-schema validator:

```rust
trait StateSchema {
    fn validate(bytes: &CanonicalBytes) -> Result<(), SchemaError>;
}
```

Every StateValue written by an accepted transaction must reference a StateSchemaId present in the active RulesetManifest and pass that schema's canonical validation.

Invalid state bytes are a RuntimeDeterminismFault and abort the current Tick before commit.
