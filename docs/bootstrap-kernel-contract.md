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
RulesetId ResolutionPoint EmissionPath Revision ResourceKey ResourceEntry
OperationSchemaId EventSchemaId Operation AccessPlan WriteAccess
WritePrecondition MutationIntent AtomicIntent SimulationEvent CauseRef
OriginRef TransactionReceipt
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
