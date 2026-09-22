# Project-specific context

## Project

TickWeave is a deterministic distributed simulation runtime intended to present a single Minecraft-compatible logical world while dynamically distributing simulation across Workers.

The architecture specifications under `docs/` are authoritative. Implementation convenience must not silently change their semantics.

## Build / test / lint / typecheck

No implementation toolchain has been committed yet.

When Rust workspace code is introduced, record the exact verified commands here.

## Dependency / toolchain

- Core/Data Plane implementation language: Rust.
- Infrastructure/orchestrator products are adapters, not architectural dependencies.
- Minecraft wire-protocol handling belongs in Gateways, not in simulation modules.
- Plugin execution, when introduced, should use a capability-based sandbox model; WASM is the intended direction.

## Architecture invariants

- Topology independence / Partition Invariance: Worker count, thread count, Cell grouping, Cell boundaries, placement, migration, split, and merge must not change committed simulation results for the same normalized input/event log and world-rules version.
- Global TickId namespace, but no world-wide tick completion barrier.
- A Chunk is the minimum spatial authority unit. A Cell is a dynamic placement/migration grouping and is not itself a fencing identity.
- CausalIsland is a safe-time/synchronization domain separate from Cell and Worker.
- Mutable authoritative state has exactly one writer authority, fenced by ownership epoch.
- Simulation code never observes Worker IDs, Cell IDs, process IDs, thread IDs, network topology, or storage topology.
- Simulation mutation is declarative: immutable/provisional state -> intents -> deterministic resolution -> commit.
- Cross-domain causal work is represented as SimulationEvent, not synchronous gameplay RPC.
- Workers are disposable compute and own no unique durable state.
- Gateway processes own transport/protocol session state; Player game state belongs to simulation.
- Gateway scale-in is drain-only for established TCP sessions. Catastrophic Gateway loss may require client reconnect.
- Persistent world state uses checkpoint + finite WAL semantics.
- Deterministic replay and cross-topology state-hash equivalence are required correctness mechanisms.
- No general optimistic rollback in the initial runtime model.

## Source of truth

Read in this order before substantive architecture or runtime changes:

1. `docs/architecture-constitution.md`
2. `docs/tick-semantics.md`
3. `docs/causal-frontier.md`
4. `docs/intent-event-semantics.md`
5. `docs/canonical-data.md`
6. `docs/authority-model.md`
7. `docs/bootstrap-kernel-contract.md`
8. `docs/implementation-plan.md`

If an implementation request conflicts with a MUST/SHALL invariant in those documents, do not silently reinterpret the invariant. Surface the design conflict.

## Generated code / source of truth

None yet.

## External constraints

- Minecraft protocol compatibility is an edge concern handled by Gateways.
- Internal simulation protocol and storage formats are versioned independently from Minecraft protocol versions.


## Bootstrap implementation status

Phase 0/1 is implementation-ready on the bootstrap architecture branch.

Codex should use `docs/bootstrap-kernel-contract.md` as the concrete execution contract. Phase 0/1 intentionally uses one CausalIsland and one in-process authority while preserving topology-independent semantic types.
