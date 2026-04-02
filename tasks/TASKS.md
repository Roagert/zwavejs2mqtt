# Task Index — zwave-group-graph Bug Fixes

**Created:** 2026-03-30

## P0 — Blocking

- [x] [TASK-001](TASK-001.md) — Fix stale nodesMap indices after removeNode splice

## P1 — Core

- [x] [TASK-002](TASK-002.md) — Fix fragile `undefined >= 0` guard in removeNode
- [x] [TASK-003](TASK-003.md) — Fix wrong property `response.message` in responseHandler
- [x] [TASK-004](TASK-004.md) — Fix priorityEdges corruption when paintLocationsView calls parseNode

## P2 — Enhancement

- [x] [TASK-005](TASK-005.md) — Remove dead `v-if="false"` alert markup in ZwaveGraph

## P1 — Round 2

- [x] [TASK-006](TASK-006.md) — Fix loadAllAssociations stuck loading state on exception
- [x] [TASK-007](TASK-007.md) — Fix removeMember stuck loading spinner on API error
- [x] [TASK-008](TASK-008.md) — Sync association changes back to Pinia store after add/remove

## Feature — Interactive Association Connector ([PRD](PRD-interactive-connector.md))

- [x] [TASK-009](TASK-009.md) — Store _assocData on each visEdge in paintAssociationsView
- [x] [TASK-010](TASK-010.md) — Add lockedSource / lockedTarget props to DialogAssociation
- [x] [TASK-011](TASK-011.md) — Add Connect Mode toggle button to GroupGraph toolbar
- [x] [TASK-012](TASK-012.md) — Wire vis-network manipulation API for drag-to-connect
- [x] [TASK-013](TASK-013.md) — Implement onConnectDrop — open DialogAssociation from drag result
- [x] [TASK-014](TASK-014.md) — Implement edge-click delete with confirmation snackbar
