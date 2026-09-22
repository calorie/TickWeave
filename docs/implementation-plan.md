# Implementation Plan

Status: **Planning v0.1**

This plan orders work so semantic correctness is executable before distributed infrastructure is introduced.

## Principle

Do not start with Kubernetes, autoscaling, Minecraft packet coverage, or production storage.

First create a small deterministic kernel whose reference semantics can be exhaustively tested. Distribution is then an optimization/preservation problem: every distributed implementation must remain equivalent to the reference kernel.

## Phase 0 — Specification and conformance skeleton

Deliver:

- Rust workspace skeleton.
- Canonical serialization/hashing primitives.
- Stable ID primitives.
- TickId / ResolutionPoint.
- ResourceKey model.
- Reference state hash format.
- Property-test harness.
- A tiny model world used only for kernel conformance.

Acceptance:

- Same canonical state serializes/hashes identically across repeated runs.
- No runtime/topology identity appears in semantic types.

## Phase 1 — Single-process reference transaction kernel

Implement:

- OperationSchema registry.
- AccessPlan.
- MutationIntent.
- AtomicIntent.
- guards and revisions.
- conflict graph.
- happens-before DAG.
- deterministic tie-break/topological ordering.
- provisional transaction state.
- SimulationEvent admission/dedup.
- Wave execution.
- deterministic RNG.

Use a deliberately small set of test operations before Minecraft behavior.

Acceptance:

- Exact deterministic replay.
- CAS conflict tests.
- Sequential operation tests.
- atomic multi-resource transfer tests.
- event retry/dedup tests.
- reference state hashes stable across thread counts.

## Phase 2 — Spatial world model

Implement authoritative primitives:

- World / Dimension.
- Chunk identity and ownership keys.
- Block state.
- Entity existence/components.
- Containers.
- scheduled objects.
- minimal spatial index as derived state.

Define exact authoritative numeric encodings before real movement/physics.

Acceptance:

- movement and resource transfer semantics do not reference Cell.
- deleting an entity conflicts with component access.
- derived indexes can be discarded/rebuilt without changing state hash.

## Phase 3 — Cell runtime in one process

Introduce dynamic Cells as runtime grouping only.

Implement:

- Chunk -> Cell assignment.
- local causal dependency tracking.
- split/merge operations.
- multi-thread scheduling.
- local event fast path that is equivalent to serialized event transport.

Acceptance:

- randomized split/merge schedules produce identical committed state hashes.
- randomized thread scheduling produces identical hashes.
- same test workload with 1 Cell and many Cells is equivalent.

## Phase 4 — Authority and multi-Worker transport

Implement:

- Worker identity as runtime-only type.
- Cell authority assignment.
- OwnershipEpoch fencing.
- versioned internal transport.
- at-least-once physical event delivery with EventId dedup.
- control-plane prototype.

Acceptance:

- stale epoch writes always fail.
- duplicate/reordered physical messages do not alter semantic results.
- network partition tests never produce two accepted writers for one authority epoch.

Before implementation, model-check the ownership/migration protocol (TLA+ or equivalent is recommended).

## Phase 5 — Durability and recovery

Implement:

- canonical WAL records.
- periodic checkpoint.
- durable resolution decisions.
- idempotent replay.
- Worker crash recovery.

Acceptance:

- kill -9/fault injection at every transaction stage.
- recovery never duplicates committed mutations/events.
- recovery state hash equals uninterrupted reference execution.

## Phase 6 — Migration, split, merge across Workers

Implement:

- background state copy/catch-up.
- boundary cutover.
- epoch transition.
- routing update.
- source cleanup.
- cross-authority AtomicIntent protocol.

Acceptance:

- arbitrary migration/split/merge schedules preserve Partition Invariance.
- migration during heavy conflict workload causes neither duplicate nor missing semantic writes.
- unrelated simulation continues while a Cell migrates.

## Phase 7 — Canonical internal protocol and Gateway

Implement:

- canonical input intents.
- canonical simulation output/events.
- one Minecraft protocol version initially.
- session sequencing.
- protocol-version translation boundary.
- connection-affine Gateway pool.
- drain-only scale-in.

Acceptance:

- Worker migration does not disconnect clients.
- Gateway rolling drain preserves existing sessions.
- catastrophic Gateway loss is handled as an allowed reconnect case, with simulation Player state preserved.

## Phase 8 — Placement and capacity control

Separate:

1. Placement Scheduler: maps Cells to existing Worker capacity.
2. Capacity Controller: changes Worker/Node capacity.

Inputs may include:

- tick/wave duration;
- CPU and memory;
- entity/chunk counts;
- mutation counts;
- cross-Cell traffic;
- halo/event bytes;
- migration cost;
- predicted movement;
- node price/capability.

Scale-out should follow attempts to use existing capacity via scheduling/repartition where appropriate.

No core algorithm may require Kubernetes. Add providers/adapters separately.

## Phase 9 — Cold/sleep and generation services

Implement:

- Cell sleep/wake.
- deterministic scheduled-work preservation.
- proven fast-forward functions for selected subsystems.
- separate deterministic world-generation service/cache.

Acceptance:

- sleep/wake path equals ordinary reference simulation.
- cold regions consume no active simulation CPU.
- regenerated immutable base terrain plus mutations equals stored authoritative view.

## Phase 10 — Minecraft semantics expansion

Only after the kernel invariants are enforced by tests, expand:

- blocks;
- fluids;
- redstone;
- entities;
- combat;
- inventories;
- portals;
- dimensions;
- generation;
- client-version coverage.

Each subsystem must state ResourceKeys, operation schemas, events, deterministic RNG streams, and conflict rules.

## Continuous verification

Every phase should maintain these test families:

- deterministic replay;
- same workload across different OS thread counts;
- same workload across randomized Cell partitions;
- randomized split/merge/migration schedules;
- duplicate/reordered/lost physical message injection;
- Worker crash injection;
- canonical state-hash comparison;
- model-based transaction tests.

## Codex task decomposition

Prefer implementation tasks that preserve a single authoritative specification boundary.

Good early tasks:

- implement canonical IDs and encoding;
- implement ResourceKey ordering;
- implement reference conflict graph;
- implement canonical topological resolver;
- implement provisional transaction store;
- implement deterministic RNG;
- implement event dedup/admission;
- implement state hash harness.

Do not ask Codex to invent unresolved Minecraft semantics while building infrastructure. When a required semantic decision is absent from the authoritative specs, record the gap and propose the smallest explicit decision.
