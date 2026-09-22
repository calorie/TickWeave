# Tick Semantics Specification

Status: **Authoritative v0.1**

## 1. Logical time

```rust
type TickId = u64;

struct ResolutionPoint {
    tick: TickId,
    wave: u32,
}
```

A World has one TickId namespace. Nominal pace is 20 Hz / 50 ms per tick.

A lightweight WorldPacer defines the maximum real-time TickId active simulation may reach. It never waits for every Cell.

Each causal domain has its own committed frontier.

Example:

```text
WorldPacer = 1000
Cell A committed = 1000
Cell B committed = 999
Cell C committed = 944
Cell D = sleeping
```

This is legal.

An active domain MUST NOT commit beyond the pacer. A lagging domain MUST NOT skip semantic ticks merely to catch up.

## 2. Determinism boundary

Determinism begins after external input is normalized and assigned a logical admission point.

For equal:

- initial authoritative state;
- normalized inputs/events;
- world-rules version,

the committed result must be equal independent of topology and physical scheduling.

Raw TCP packet arrival timing before admission is not itself part of the deterministic contract.

## 3. Tick transition

The abstract transition is:

```text
S[T]
+ Inputs[T]
+ Events[T]
    |
    v
deterministic transition
    |
    v
S[T+1]
```

Simulation code never directly mutates `S[T]`.

## 4. Tick phases

Every tick has these logical phases:

```text
ADMIT
  -> EVALUATE
  -> RESOLVE
  -> REACTIVE (0..N waves)
  -> COMMIT
  -> PUBLISH
```

### ADMIT

Seal normalized external inputs and already-admissible events for Tick T.

After sealing, ordinary late input is assigned to a later legal point; it is never inserted retroactively into a committed past.

### EVALUATE

Read the immutable Tick-T view and compute candidate work such as movement, physics, AI, scheduled work, block logic, and plugin logic.

Direct authoritative writes are forbidden.

Output is declarative Intent data.

### RESOLVE

Resolve conflicting MutationIntent/AtomicIntent values using the canonical rules in `intent-event-semantics.md`.

### REACTIVE

Apply deterministic causal propagation.

Reactive processing consists of numbered Waves.

Work emitted from Wave W becomes eligible no earlier than Wave W+1.

A Wave observes the provisional results of preceding Waves in the same Tick.

### COMMIT

Atomically establish the final Tick-T transition to S[T+1] for the participating causal domain and advance its commit frontier.

### PUBLISH

Generate client-visible output, metrics, and permitted external effects from committed state only.

Uncommitted provisional state must not be exposed as authoritative output.

## 5. Provisional state and waves

Conceptually:

```text
S[T]
 -> Wave 0 resolve
 -> P[T,1]
 -> Wave 1 resolve
 -> P[T,2]
 -> ...
 -> quiescence
 -> durable COMMIT
 -> S[T+1]
```

Wave state is provisional until Tick commit.

## 6. No global barrier

No rule may require every Cell in the World to finish Tick T before an unrelated Cell can progress.

Synchronization is causal and scoped.

A domain waits only for dependencies required to know that its relevant input/event set for the current ResolutionPoint is closed.

Unrelated distant regions do not wait for one another.

## 7. Causal completion watermark

For an admitted dependency, participants exchange logical completion information equivalent to:

```text
PhaseComplete(TickId, Phase/Wave, DependencyDomain)
```

A ResolutionPoint may close only when all dependencies that were admitted for that point are known complete.

The exact physical aggregation structure is an implementation choice, but its result must be deterministic and topology-independent.

## 8. Input admission

Gateway input first carries transport/session sequencing. Simulation ingress converts it into a normalized representation such as:

```rust
struct NormalizedInput {
    id: InputId,
    actor: ObjectId,
    apply_at: ResolutionPoint,
    actor_sequence: u64,
    payload: CanonicalBytes,
}
```

Once `apply_at` is assigned it is immutable.

Physical packet arrival order must not become a conflict-resolution rule.

## 9. Reactive bounds

Reactive propagation normally continues until the admitted causal component is quiescent.

To prevent non-termination/DoS, world rules may specify deterministic limits such as:

```text
MAX_REACTIVE_ROUNDS
MAX_REACTIVE_EVENTS_PER_COMPONENT_PER_TICK
```

If a bound is reached, the canonical remainder is carried to a later legal ResolutionPoint.

Limits are versioned world semantics. They must not vary based on Worker speed, Worker count, or Cell size.

## 10. Same-wave recursion is forbidden

An event produced during Wave W MUST NOT become observable in Wave W.

Minimum reactive delivery is Wave W+1.

This makes causal closure finite and explicit.

## 11. Remote dependency expansion

After a ResolutionPoint begins closing, gameplay logic may not discover an arbitrary previously unrelated remote participant and force it into the current Wave.

Long-range work is scheduled as a future SimulationEvent so the dependency can be admitted before that future point opens.

Local neighborhood interactions may remain same-Tick across successive Waves because their causal neighborhood is admitted.

## 12. Movement and spatial boundaries

Movement is a normal state transition:

```text
position[T]
 -> movement intent
 -> position[T+1]
```

Crossing a Chunk or Cell boundary MUST NOT change its gameplay semantics.

Runtime authority/routing may change as a consequence of committed position, but such placement is not part of the gameplay Intent.

## 13. Entity transfer and identity

Stable EntityId is invariant across Cell/Worker movement.

A logical transfer that changes multiple authoritative resources uses an AtomicIntent so duplication/loss is impossible in committed state.

Physical routing changes are runtime metadata.

## 14. Long-range operations

Teleport, cross-world transfer, administrative remote mutation, and similar non-local operations use asynchronous future admission.

They never inject effects into an already closed ResolutionPoint.

Their final world-state transition may still be atomic across multiple authorities, as specified by the Intent/Event semantics.

## 15. Rollback policy

The initial runtime uses conservative execution.

General optimistic simulation with client-visible rollback is not part of v0.1.

A domain commits only when required causal input is known and the durable decision is established.

## 16. Randomness

Authoritative randomness is a pure deterministic function of stable semantic material, for example:

```text
Random(
  world_seed,
  tick,
  stable_object_id,
  stream_id,
  invocation_index
)
```

Worker ID, Cell ID, thread ID, process ID, wall clock, and network timing are forbidden seed material.

## 17. Durability and publication

Logical order:

```text
finalize deterministic delta
 -> durable decision/WAL as required
 -> advance commit frontier
 -> publish externally observable output
```

After publication, the system must not later claim the committed semantic transition never occurred.

## 18. Failure during a Tick

If a Worker dies before a Tick/ResolutionPoint decision is committed, the work may be deterministically re-executed elsewhere.

If the decision is already durable/committed, recovery applies or resumes from that decision; it does not produce a second semantic execution.

## 19. Migration / split / merge

Authority cutover occurs at a semantic boundary.

Example migration:

```text
Worker A owns Cell C through committed Tick T
 -> durable/caught-up handoff
 -> ownership epoch increments
 -> Worker B owns subsequent work
```

Split and merge use the same principle.

Changing topology must not change state hashes for the same normalized workload.

## 20. Sleeping domains

A sleeping region does not execute empty 20 Hz work.

It stores enough authoritative state to resume deterministically, including relevant scheduled future work and rules version.

A subsystem may fast-forward only when equivalence to ordinary Tick semantics is established.

## 21. Overload semantics

If real-time 20 TPS cannot be maintained:

```text
correctness > real-time progression
```

The affected causal domain lags.

The runtime may increase intra-domain parallelism, repartition, migrate, or scale capacity, but it must not silently drop semantic ticks as an autoscaling technique.

## 22. Conformance property

For a fixed initial state, normalized event log, and world-rules version, all legal executions across different Worker/Cell/thread topologies must produce equal canonical committed state hashes at comparable Tick boundaries.
