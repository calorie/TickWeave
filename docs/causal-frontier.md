# Causal Frontier and Island Semantics

Status: **Authoritative v0.1**

This document closes the safe-time gap created by combining independent progress, no global barrier, and no general rollback.

## 1. Terms

A **CausalIsland** is a synchronization domain for simulation progress. It is not a placement unit and is not gameplay state.

A **Cell** is a compute placement/migration grouping.

A **Chunk authority** is a fencing domain.

These concepts are intentionally separate.

CausalIsland identity is runtime metadata and is excluded from canonical world state hashes.

## 2. Safety rule

A CausalIsland may close and commit a ResolutionPoint only when it has proof that no still-legal inbound SimulationEvent can later appear at that point or any earlier point.

Therefore this is forbidden:

```text
Island A commits Tick 1000
later:
Island B sends an event whose deliver_at is Tick 998
```

No implementation may repair this by silently dropping the event or by general rollback.

## 3. Registered inbound dependencies

An island only waits for explicitly registered inbound dependencies.

For every registered inbound dependency, the receiver tracks a closure/watermark indicating the greatest ResolutionPoint for which that sender can no longer introduce work.

A point is safe to close when every registered dependency relevant to that point has advanced its closure beyond that point and all locally admitted work is quiescent.

This is conservative parallel discrete-event simulation semantics scoped to actual dependencies, not a world-wide tick barrier.

## 4. Creating a new remote dependency

A previously unrelated sender MUST NOT send work into the receiver's already-closed past.

Before a new remote dependency becomes usable, the sender obtains an admission reservation from the receiver.

Conceptually:

```rust
struct AdmissionReservation {
    id: AdmissionReservationId,
    receiver_scope: CausalScope,
    deliver_at: ResolutionPoint,
}
```

Protocol:

1. Sender requests a future admission point.
2. Receiver chooses a legal `deliver_at` strictly after its closed frontier and records the reservation.
3. Receiver promises not to close `deliver_at` until the reservation is satisfied or durably cancelled.
4. Sender emits the event referencing the reservation.
5. Event arrival or cancellation satisfies the reservation.

A reservation and its selected `deliver_at` are admission metadata and MUST be recorded in the replay/admission log when they can affect future simulation.

They are not authoritative gameplay ResourceKeys.

## 5. Determinism boundary

The deterministic simulation contract begins after inputs/events and their logical admission points are fixed.

Real-time coordination can cause two live executions to choose different future admission points. Such executions have different normalized admission logs and are therefore not the same deterministic replay input.

Given the same admission log, CausalIsland placement, Worker placement, transport timing, and thread scheduling MUST NOT change committed results.

## 6. Local spatial interaction

Gameplay code does not observe CausalIslands.

A local interaction that may cross an island boundary must have a registered dependency before it becomes eligible for the relevant ResolutionPoint.

The runtime may maintain conservative interaction envelopes/guard bands so neighboring active spatial domains are registered before entities, fluids, redstone, or other local propagation can cross.

If a dependency was not registered in time, the runtime MUST schedule the new causal work at a legal future point; it must never inject it into a closed past.

Exact envelope tuning is a performance policy. Safety is not.

## 7. Same-Tick cross-island propagation

Registered neighboring islands may exchange events across successive Waves of the same Tick.

For Wave W:

- events emitted during W are eligible no earlier than W+1;
- the receiver does not close W+1 until registered inbound dependencies for W+1 are closed;
- physical message arrival order has no semantic meaning.

This allows redstone/neighbor-style propagation without a world-wide barrier.

## 8. Rendezvous merge

When islands become strongly or persistently coupled, the runtime may merge them.

The merge uses a future rendezvous point R that is legal for both islands.

```text
A frontier ----\
                > rendezvous R -> merged island
B frontier ----/
```

Before R, existing registered-event semantics remain valid.

At R, both islands have closed all earlier work, outstanding reservations that target earlier points are satisfied, and the merged island begins with one synchronization frontier.

The choice of R is admission/control metadata and is recorded for replay when relevant.

## 9. Split

An island may split only at a closed Tick boundary where the proposed sides have no unresolved cross-split reservation/event whose delivery is at or before that boundary.

After split, each new island starts from the same committed boundary.

Future interaction between them uses the ordinary dependency/reservation rules.

## 10. WorldPacer

WorldPacer limits how far active simulation may advance relative to wall-clock logical time. It is not a safety proof.

Safe-time comes from dependency closure and reservations, not from assuming another island will eventually catch up.

## 11. Sleeping regions

Sleeping state has no independently advancing frontier.

Before a live island can deliver causal work into a sleeping region, the target is woken and a legal future admission point is established.

Wake/catch-up must obey the normal deterministic rules.

## 12. Phase 0/1 reference kernel

The initial reference kernel SHALL use exactly one CausalIsland.

Therefore Phase 0/1 requires no distributed safe-time protocol, admission reservation network protocol, or island merge implementation.

However, the semantic APIs MUST NOT encode assumptions that every future implementation has one island.

The single-island reference result is the oracle that later multi-island implementations must reproduce for the same normalized admission log.

## 13. Non-negotiable invariant

```text
No committed point may later receive newly admitted causal work.
```

Any implementation that cannot prove this must stop progress rather than guess, drop work, or rewrite history.
