# TickWeave

TickWeave is an experimental distributed simulation runtime for a Minecraft-compatible world.

The project is not a cluster of conventional Minecraft servers. Its target architecture is a single logical world implemented as a deterministic distributed state machine whose simulation can be partitioned, migrated, split, merged, slept, and resumed without making the compute topology part of game semantics.

## Status

Architecture/specification phase. No compatibility or production-readiness claims are made yet.

## Goals

- Present one logical world to Minecraft clients.
- Scale simulation horizontally without a world-wide tick barrier.
- Preserve observable simulation results across arbitrary Cell/Worker placement changes.
- Keep exactly one authoritative writer for mutable world state.
- Make Workers disposable and recoverable from durable state.
- Allow Cells to split, merge, migrate, sleep, and wake without stopping unrelated simulation.
- Terminate Minecraft protocol sessions in independently scalable Gateways.
- Make deterministic replay and fault injection first-class correctness tools.
- Reduce idle-world compute toward zero.

## Non-goals

- Bit-identical reproduction of Mojang's server implementation.
- Bukkit/Spigot/Paper plugin compatibility as an architectural constraint.
- Exposing Worker, Cell, network, thread, or storage topology to game logic.
- Using Kubernetes, a specific cloud, or a specific message broker as part of the core model.

## Authoritative design documents

Read these before implementation:

1. [Architecture Constitution](docs/architecture-constitution.md)
2. [Tick Semantics Specification](docs/tick-semantics.md)
3. [Intent and Event Semantics Specification](docs/intent-event-semantics.md)
4. [Implementation Plan](docs/implementation-plan.md)

These documents define semantic constraints. Implementation convenience is not a reason to violate them.

## Core idea

```text
Minecraft Clients
       |
       v
Protocol Gateways  <-- horizontally scaled, connection-affine
       |
       v
Canonical Internal Protocol
       |
       v
Distributed Simulation Fabric
  |        |        |
 Cell A   Cell B   Cell C
  \        |       /
   \-- causal sync
       |
       v
Workers (disposable compute)
       |
       +--> checkpoint + WAL --> durable storage
```

A Chunk is the minimum ownership unit. A Cell is a dynamic scheduling/migration unit containing a connected set of Chunks. Cell boundaries are not game semantics.

## Development

This repository uses `AGENTS.md` as the agent execution contract. Codex and other coding agents must also read `.agentic/PROJECT.md` and the authoritative design documents above before substantive implementation.

Long-lived architectural changes should be proposed explicitly rather than silently encoded in implementation.
