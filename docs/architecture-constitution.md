# Architecture Constitution

Status: **Authoritative v0.1**

This document defines architectural decisions that implementations must preserve. Detailed tick and transaction semantics live in the linked specifications.

## 1. System identity

TickWeave is a distributed simulation runtime, not a federation of conventional Minecraft servers.

A client should observe one logical world and one logical server session even when world computation moves across processes or machines.

## 2. Compatibility target

TickWeave targets:

- Minecraft wire-protocol compatibility at the Gateway boundary; and
- compatible player-observable gameplay semantics implemented by the runtime.

TickWeave does **not** require bit-identical reproduction of Mojang server internals, iteration accidents, thread scheduling, or undocumented implementation order.

## 3. Partition Invariance

For the same:

- initial authoritative state;
- normalized input/event log; and
- world-rules version,

the committed simulation result MUST be independent of:

- Worker count;
- OS thread count;
- Cell count;
- Cell boundaries;
- Cell placement;
- migration schedule;
- split schedule; and
- merge schedule.

Formally, for any two legal topologies A and B:

```text
Hash(State_A[T]) == Hash(State_B[T])
```

for each comparable committed TickId.

This is the project's central correctness property.

## 4. World partitioning

### Chunk

A Chunk is the minimum spatial ownership unit.

For every authoritative Chunk at any instant there MUST be exactly one active write authority.

### Cell

A Cell is a dynamic scheduling, causal-coordination, persistence, and migration grouping over a connected set of Chunks.

Cells MAY split, merge, migrate, sleep, and wake.

Cell shape is not restricted to rectangles or a quadtree.

Cell boundaries MUST NOT be visible to gameplay logic.

## 5. Authority and fencing

Every authoritative Cell assignment has:

```text
CellId
OwnershipEpoch
OwnerWorkerId
```

All authoritative writes are accepted only under the current epoch. A stale Worker cannot write after authority has moved.

Single-writer authority is logical; computation within an authority MAY use parallel evaluation so long as commit semantics remain deterministic.

## 6. Time

The World has a common `TickId` namespace with a nominal 20 Hz real-time pace.

There is no world-wide completion barrier.

Each causal domain advances its own commit frontier subject to dependencies defined by the Tick Semantics Specification.

Correctness has priority over wall-clock progress. An overloaded domain lags; it does not skip semantic ticks.

## 7. Simulation mutation

Gameplay code MUST NOT mutate shared authoritative state directly.

The semantic path is:

```text
immutable/provisional state
        -> deterministic evaluation
        -> MutationIntent / AtomicIntent
        -> deterministic conflict resolution
        -> durable commit
        -> observable output
```

Game code MUST NOT use thread timing, network arrival order, mutex acquisition order, Worker identity, or Cell identity to define gameplay results.

## 8. Cross-domain interaction

Cross-domain causal work is represented by versioned `SimulationEvent` values with logical delivery points.

Gameplay modules MUST NOT perform synchronous Worker-to-Worker RPC as part of simulation semantics.

Physical local fast paths are allowed only if they are semantically equivalent to event transport.

## 9. Determinism

After external input has been normalized and assigned a logical admission point, execution MUST be deterministic.

Authoritative randomness is derived from stable semantic keys. It MUST NOT depend on Worker, Cell, process, thread, wall clock, or network timing.

Stable object identity MUST survive migration.

## 10. Gateway architecture

Minecraft TCP/protocol connections terminate at Gateways.

Gateways translate version-specific wire packets to/from a canonical internal protocol.

Simulation state such as position, inventory, health, effects, XP, and abilities belongs to the simulation fabric, not to the Gateway.

Gateways horizontally autoscale using edge-load signals such as:

- established connection count;
- packets per second;
- RX/TX bandwidth;
- codec CPU;
- event-loop lag.

Existing TCP sessions are connection-affine.

Normal scale-in is drain-only:

```text
ACTIVE -> DRAINING -> zero sessions -> terminate
```

Routine Gateway scale-out, scale-in, and rolling replacement should not disconnect established clients.

Catastrophic Gateway process/node loss MAY require client reconnect in v0.x. Transparent replication of live TCP transport state is not a core requirement.

## 11. Workers

Workers are disposable compute.

No Worker may be the sole durable holder of world state.

Killing a Worker must not corrupt authoritative world state. Affected Cells are reconstructed from durable state and reassigned.

## 12. Persistence

The core persistence model is:

```text
periodic checkpoint + finite WAL
```

A committed externally observable state transition must satisfy its configured durability policy before publication.

Storage engine choice is an implementation adapter, not gameplay semantics.

## 13. Control plane

Control-plane metadata requiring strong consistency includes authority assignment and fencing epochs.

The control plane MUST NOT be required in the steady-state per-tick data path.

If the control plane is temporarily unavailable, existing simulation should continue where authority is already unambiguous, while operations such as migration, split, merge, or scale decisions may pause.

## 14. Orchestration independence

Kubernetes, Agones, cloud VM APIs, bare metal managers, and similar systems are optional adapters.

The core runtime interacts through abstractions such as a future `NodeProvider`; it does not encode Pod semantics into simulation.

## 15. World generation and cold state

World generation is conceptually separable from active simulation.

Generated terrain should be treated as reproducible base data where practical, with authoritative mutations layered over it.

Inactive regions may sleep with zero active simulation compute.

Fast-forwarding a sleeping subsystem is allowed only when its result is exactly equivalent to the specified ordinary tick semantics.

## 16. Plugins

Bukkit/Paper API compatibility is not a core constraint.

Plugin execution must not gain implicit global mutable state, unrestricted threads, blocking filesystem/network I/O, or topology visibility.

The intended direction is a capability-based sandbox, such as WASM, with deterministic/scheduled host APIs.

## 17. Verification

Correctness work is part of the product.

The runtime must support:

- deterministic replay;
- canonical state hashing;
- cross-topology equivalence tests;
- migration/split/merge fault injection;
- duplicate/reordered physical message tests; and
- model checking for authority/fencing protocols where practical.

## 18. Change rule

Changes that invalidate Partition Invariance, single-writer fencing, deterministic resolution, or the no-global-barrier model are architecture changes, not implementation refactors. They require an explicit specification/ADR decision before implementation.
