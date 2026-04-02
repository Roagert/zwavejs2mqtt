---
id: TASK-013
title: Implement onConnectDrop — open DialogAssociation from drag result
priority: P1
status: open
depends_on: [TASK-010, TASK-012]
prd: PRD-interactive-connector
created: 2026-04-02
---

## Description

Implement `onConnectDrop(fromVisId, toVisId)` which is called by the vis manipulation `addEdge` callback. It validates the drag and opens DialogAssociation pre-filled with both nodes locked.

## Implementation

```js
onConnectDrop(fromVisId, toVisId) {
  // Guard: same node
  if (fromVisId === toVisId) return

  // Guard: either side is a group diamond, not a device node
  if (
    typeof fromVisId === 'string' && !fromVisId.startsWith('node_') ||
    typeof toVisId   === 'string' && !toVisId.startsWith('node_')
  ) return

  const sourceId = parseInt(String(fromVisId).replace('node_', ''))
  const targetId = parseInt(String(toVisId).replace('node_', ''))

  const sourceNode = this.nodes[this.nodesMap.get(sourceId)]
  const targetNode = this.nodes[this.nodesMap.get(targetId)]
  if (!sourceNode || !targetNode) return

  this.connectorSource = sourceNode
  this.connectorTarget = targetNode
  this.showConnectorDialog = true
},
```

Add to data:

```js
connectorSource: null,
connectorTarget: null,
showConnectorDialog: false,
```

Add DialogAssociation instance in template wired to these data properties with `lockedSource` and `lockedTarget`. On dialog `add` event call `addMember()` using the selected group and `connectorTarget.id`.

## Acceptance Criteria

- [ ] Dropping on same node does nothing
- [ ] Dropping on a group diamond does nothing
- [ ] Dropping on a valid target opens DialogAssociation with source and target pre-filled and locked
- [ ] Confirming creates the association and repaints the graph
- [ ] Cancelling leaves no association and no ghost edge
