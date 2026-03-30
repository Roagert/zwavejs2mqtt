---
id: TASK-004
title: Fix priorityEdges corruption when paintLocationsView calls parseNode
priority: P1
status: done
depends_on: []
created: 2026-03-30
---

## Description
`paintLocationsView()` calls `this.parseNode(node, [])` for every node. `parseNode` internally calls `parseRouteStats` which writes orphaned edge objects into `this.priorityEdges`. These objects are never added to any vis-network DataSet, so if a live `updateNode` event fires while the Locations view is active, the save/restore of `priorityEdges` in the update handler will restore these corrupted entries back into the routes graph when the user switches back.

The update handler at line ~797 already documents this pattern:
```js
// save/restore priorityEdges because parseNode calls
// parseRouteStats which overwrites entries with edge
// objects that are never added to the DataSet
const saved = { ...this.priorityEdges }
```
`paintLocationsView` must do the same.

## Acceptance Criteria
- [ ] `this.priorityEdges` is unchanged after `paintLocationsView()` runs
- [ ] Switching Routes → Locations → Routes produces a correct routes graph

## Implementation Notes
Wrap the `parseNode` loop in `paintLocationsView` with a save/restore:
```js
const savedEdges = { ...this.priorityEdges }
for (const node of this.allNodes) {
    const { node: entity } = this.parseNode(node, [])
    visNodes.add(entity)
    ...
}
this.priorityEdges = savedEdges
```
File: `src/components/custom/ZwaveGraph.vue` in `paintLocationsView`.

## Execution Log
| Iteration | Outcome | Notes |
|-----------|---------|-------|
