---
id: TASK-009
title: Store _assocData on each visEdge in paintAssociationsView
priority: P1
status: open
depends_on: []
prd: PRD-interactive-connector
created: 2026-04-02
---

## Description

When building the visEdges DataSet in `paintAssociationsView()` (GroupGraph.vue), attach `_assocData` to every edge object. This metadata is required by the edge-delete flow so that clicking an edge can identify exactly which association to remove without a reverse-lookup.

## Shape

```js
visEdges.add({
  // ...existing fields...
  _assocData: {
    sourceNodeId: <number>,
    sourceEndpoint: <number>,   // association endpoint (0 for root)
    groupId: <number>,
    targetNodeId: <number>,
    targetEndpoint: <number | undefined>,
  },
})
```

## Acceptance Criteria

- [ ] Every edge in Devices view has `_assocData` set
- [ ] `_assocData.sourceNodeId` matches the node the group belongs to
- [ ] `_assocData.targetNodeId` matches the member node
