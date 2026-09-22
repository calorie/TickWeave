# Bootstrap deterministic kernel

## Goal

Implement TickWeave Phase 0 and Phase 1 exactly as defined by `docs/bootstrap-kernel-contract.md`.

## Required reading

1. `docs/architecture-constitution.md`
2. `docs/tick-semantics.md`
3. `docs/causal-frontier.md`
4. `docs/intent-event-semantics.md`
5. `docs/canonical-data.md`
6. `docs/authority-model.md`
7. `docs/bootstrap-kernel-contract.md`
8. `docs/implementation-plan.md`

## Acceptance

All Definition of Done items in the bootstrap contract must pass.

No Minecraft protocol, networking, Cell scheduler, Worker control plane, or production persistence is in scope.

## Design escalation rule

If code requires a semantic decision not answered by the authoritative documents, stop that narrow implementation path, record the gap in DECISIONS.md, and propose the smallest spec amendment. Continue independent work that is not blocked.
