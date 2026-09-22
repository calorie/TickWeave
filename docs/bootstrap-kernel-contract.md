# Bootstrap Kernel Implementation Contract

Status: **Implementation-ready v0.1**

This is the execution contract for Codex to implement Phase 0 and Phase 1 without additional product/design decisions.

## 1. Scope

Implement only the deterministic single-process reference kernel.

Do not implement yet: Minecraft protocol, Gateway, networking/RPC, async distributed transport, Worker control plane, Cell placement, multi-island safe-time protocol, production WAL/database, cloud/orchestration, or real Minecraft gameplay semantics.

The output is the semantic oracle against which later distributed implementations are compared.

## 2. Rust workspace

Create an Edition 2024 workspace using resolver 3:

```text
crates/
  tickweave-semantic/
  tickweave-kernel/
  tickweave-testkit/
```

### tickweave-semantic

Own canonical encode/decode, semantic IDs/hashing, ResolutionPoint, ResourceKey/ResourceEntry, access types, schema IDs, EmissionPath, RulesetId, and state-hash primitives.

It must not depend on runtime/network/orchestrator concepts.

### tickweave-kernel

Own schema registry, Intent types, guards, access normalization, provisional reference store, conflict graph, happens-before DAG, canonical ordering, transaction execution, revisions/tombstones, event materialization/dedup, single-CausalIsland Wave execution, deterministic RNG, receipts/errors.

### tickweave-testkit

Own tiny non-Minecraft schemas, property generators, replay harness, state-hash comparison helpers, and physical-order perturbation helpers.

## 3. Toolchain policy

- stable Rust, pinned in `rust-toolchain.toml` when bootstrap starts;
- Edition 2024;
- `#![forbid(unsafe_code)]` in all initial crates;
- no MSRV promise beyond the pinned toolchain yet;
- Serde may be used for diagnostics/test fixtures but not as Canonical Encoding v1;
- no async runtime in Phase 0/1;
- minimal dependencies; BLAKE3 is required, property-test libraries are allowed.

## 4. Required semantic types

Implement strong newtypes for at least:

```text
WorldId ObjectId IntentId AtomicIntentId EventId InputId ScheduleId
RulesetId ResolutionPoint EmissionPath Revision CanonicalBytes
SemanticAddress ResourceKey StateValue ResourceEntry
OperationSchemaId EventSchemaId StateSchemaId Operation AccessPlan WriteAccess
WritePrecondition MutationIntent AtomicIntent EventDraft SimulationEvent CauseRef
OriginRef AnyIntentId TransactionReceipt WorldSnapshotV1
```

Minecraft-specific ResourceKey variants may remain test-oriented until Phase 2, but canonical extension/ordering must be explicit.

## 5. Reference store

Use a simple deterministic in-memory reference store based on ordered collections.

It stores ResourceEntry including absent/tombstone revision state.

Derived indexes are excluded from canonical state.

Correctness takes precedence over performance.

## 6. Operation schema contract

Every schema provides payload validation, a complete access plan, and deterministic resolution through TransactionContext.

Resolvers cannot expand access sets.

Write access implies read access to that same key.

No wall clock, system RNG, thread identity, network, filesystem, WorkerId, or CellId capability is exposed.

## 7. Reference resolution

For each ResolutionPoint:

1. validate admitted intents;
2. derive and normalize access plans;
3. build conflicts on same ResourceKey with at least one writer;
4. split connected components;
5. build happens-before constraints;
6. deterministic topological sort with specified BLAKE3 tie-break;
7. execute against provisional state;
8. deterministically accept/reject;
9. buffer events for accepted transactions;
10. materialize accepted events;
11. complete the Wave;
12. repeat reactive Waves as required by test schemas;
13. commit reference state;
14. compute canonical state hash.

Independent components may execute physically in any order; tests prove equality.

## 8. AtomicIntent normalization

For duplicate write keys:

- Current + Current -> Current;
- ExactRevision(R) + Current -> ExactRevision(R);
- ExactRevision(R) + ExactRevision(R) -> ExactRevision(R);
- different ExactRevision values -> invalid Intent.

One AtomicIntent increments a changed key's revision once at transaction commit.

Members see preceding private member writes.

Any member rejection rejects all private writes and buffered events.

## 9. Required test schemas

The testkit must represent operations equivalent to:

- CreateValue;
- DeleteValue;
- PutExact(expected revision);
- AddCurrent(signed delta);
- TransferExact(from/to, all-or-nothing);
- EmitOnSuccess;
- SpawnStableObject using deterministic EmissionPath.

Names may differ; semantics may not.

## 10. Required tests

### Canonical data
- golden primitive/container bytes;
- canonical map/set order and invalid-order rejection;
- ResourceKey order;
- domain-separated hash vectors;
- stable ID vectors;
- RulesetId vector;
- complete reference state-hash vector.

### Revision
- create/delete/recreate never resets revision;
- stale ExactRevision fails after ABA-shaped delete/recreate;
- AtomicIntent multiple writes increment once.

### Determinism
- randomized physical input/storage iteration does not change canonical result after the admitted set is fixed;
- independent component execution order does not change hash;
- repeated replay yields byte-identical receipts/events/hash;
- stable IDs do not depend on scheduling.

### Conflicts
- write/write and write/read conflict;
- read/read does not;
- ExactRevision yields deterministic first-success winner;
- Current operations compose in canonical order.

### Atomicity
- multi-key transfer all-or-nothing;
- late member failure exposes no earlier member write;
- rejected AtomicIntent emits no event.

### Events
- duplicate EventId is semantically admitted once;
- Wave W emission is visible no earlier than W+1;
- duplicate/reordered physical delivery representation leaves result unchanged.

### Fault classes
- undeclared access -> RuntimeDeterminismFault;
- inconsistent AtomicIntent ExactRevision requirements -> invalid;
- duplicate semantic ID under a cause -> determinism fault;
- happens-before cycle -> fault/no commit.

## 11. Verification commands

```text
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace --all-features
```

## 12. Autonomous implementation choices

Codex may choose module/file layout inside the three crates, private helper types, error library, property-test library, graph algorithm internals, test naming, benchmark harness, ordered collection helpers, and non-authoritative diagnostic formats.

## 13. Decisions Codex must not silently change

- canonical encoding bytes;
- hash framing/domains;
- stable semantic ID rules;
- ResourceEntry revision/tombstone behavior;
- conflict definition;
- WritePrecondition behavior;
- canonical ordering/tie-break;
- AtomicIntent all-or-nothing semantics;
- Event Wave rule;
- Partition Invariance/determinism boundary;
- authoritative vs derived state.

## 14. Definition of done

Phase 0/1 is complete when all required checks pass, golden vectors are committed, randomized physical-order tests preserve canonical results, replay reproduces receipts/events/hash, semantic types contain no topology identity, and deferred design gaps are explicitly recorded rather than hidden in code.


## 15. Fault atomicity

Gameplay rejection is local to the rejected transaction and normal execution continues.

A RuntimeDeterminismFault, invalid canonical value, happens-before cycle, ID/content collision, undeclared access, or semantic counter overflow aborts the entire current Tick for the single reference CausalIsland.

All provisional writes/events produced since the beginning of that Tick are discarded and the committed state remains S[T].

The kernel must not partially commit earlier Waves from a faulted Tick.

## 16. Event admission ledger

Phase 0/1 keeps an unbounded EventAdmissionLedger as specified by `canonical-data.md`.

Do not implement dedup GC yet.

The ledger is part of replay/state-hash conformance.

## 17. Canonical observable ordering

Physical execution order of independent conflict components is non-semantic.

When the reference harness emits ordered diagnostic/conformance collections:

- TransactionReceipts are sorted by canonical intent identity bytes;
- newly materialized SimulationEvents are sorted by EventId bytes;
- Resource deltas are sorted by ResourceKey canonical bytes.

Gameplay semantics must not depend on these presentation orders, but conformance outputs use them so replay artifacts are byte-stable.

## 18. Generic bootstrap key/value model

Use the exact generic `ResourceKey`, `StateValue`, schema IDs, RulesetManifestV1, RNG v1, and tie-break v1 definitions from `canonical-data.md`.

Do not invent Minecraft-specific canonical enum discriminants during Phase 0/1.


## 19. Snapshot semantics

Use `WorldSnapshotV1` from `canonical-data.md` as the exact reference state-hash payload.

The kernel starts a Tick from S[T], keeps all Wave effects provisional, and produces S[T+1] only after successful Tick commit.

There is no authoritative intermediate-Wave snapshot.

## 20. Guard representation

Do not add a generic `guards` field to MutationIntent/AtomicIntent/Operation v1.

Typed guard helpers belong to Operation schemas and compile to canonical payload plus declared reads/write preconditions.

This keeps the core envelope stable.


## 21. TransactionContext v1

The semantic context exposes an entry view rather than assuming resources always exist:

```rust
struct ResourceEntryView<'a> {
    revision: Revision,
    value: Option<&'a StateValue>,
}

trait TransactionContext {
    fn entry(&self, key: &ResourceKey) -> ResourceEntryView<'_>;
    fn write(&mut self, key: &ResourceKey, value: StateValue);
    fn delete(&mut self, key: &ResourceKey);
    fn emit(&mut self, draft: EventDraft);
    fn random_u64(
        &self,
        stream_id: u32,
        invocation_index: u32,
    ) -> u64;
}
```

A never-seen key reads as revision 0 / absent.

The context enforces the declared normalized AccessPlan.

Writes/deletes are transaction-local until acceptance.

## 22. Wave evaluation boundary

Evaluation for a Wave reads the immutable provisional state produced by prior Waves.

Evaluation emits deterministic Intent/AtomicIntent drafts; it does not mutate state.

Event-derived transaction IDs use `CauseRef::SimulationEvent(event.id)` plus schema-defined EmissionPath.

Resolution then processes the complete admitted transaction set for that Wave.

## 23. Test Ruleset

Use the reserved schema IDs and bootstrap reactive-limit values defined by `canonical-data.md`.

Do not allocate alternative numeric IDs for the required conformance schemas.


## 24. Resolver outcome

An Operation resolver returns either success or a normal `RejectReason`.

On success, staged writes/deletes/events remain transaction-private until the transaction accepts.

On RejectReason, all private effects of that MutationIntent or AtomicIntent are discarded.

ExactRevision mismatch is checked by the kernel before schema resolution and produces `RejectReason::RevisionMismatch`.

Invalid schema/payload/state bytes are runtime faults, not RejectReason values.

## 25. State-schema registry

The bootstrap kernel registry includes StateSchema validators from the active RulesetManifest.

No StateValue is committed without deterministic schema validation.

The required test StateSchema IDs are fixed by `canonical-data.md`.
