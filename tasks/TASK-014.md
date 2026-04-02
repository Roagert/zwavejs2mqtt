---
id: TASK-014
title: Implement edge-click delete with confirmation snackbar
priority: P1
status: open
depends_on: [TASK-009, TASK-012]
prd: PRD-interactive-connector
created: 2026-04-02
---

## Description

In `createNetwork()`, add a `selectEdge` handler. When connect mode is active and the selected edge has `_assocData`, show a confirmation snackbar with a 5-second window before removing the association.

## Data additions

```js
pendingDeleteEdge: null,
showDeleteSnackbar: false,
```

## Event handler (inside createNetwork)

```js
this.network.on('selectEdge', (params) => {
  if (!this.connectMode) return
  if (!params.edges.length) return
  const edgeData = visEdges.get(params.edges[0])
  if (!edgeData?._assocData) return
  this.pendingDeleteEdge = edgeData._assocData
  this.showDeleteSnackbar = true
})

this.network.on('deselectEdge', () => {
  if (!this.showDeleteSnackbar) return
  this.pendingDeleteEdge = null
  this.showDeleteSnackbar = false
})
```

## Snackbar template

```html
<v-snackbar v-model="showDeleteSnackbar" :timeout="5000" color="error">
  Remove association
  <strong>{{ getNodeName(pendingDeleteEdge?.sourceNodeId) }}</strong>
  → <strong>{{ getNodeName(pendingDeleteEdge?.targetNodeId) }}</strong>?
  <template #actions>
    <v-btn variant="text" @click="confirmDeleteEdge">Remove</v-btn>
    <v-btn variant="text" @click="showDeleteSnackbar = false">Cancel</v-btn>
  </template>
</v-snackbar>
```

## confirmDeleteEdge method

```js
async confirmDeleteEdge() {
  if (!this.pendingDeleteEdge) return
  const d = this.pendingDeleteEdge
  this.showDeleteSnackbar = false
  this.pendingDeleteEdge = null
  const source = { nodeId: d.sourceNodeId, endpoint: d.sourceEndpoint ?? undefined }
  const target = { nodeId: d.targetNodeId, endpoint: d.targetEndpoint ?? undefined }
  await this.app.apiRequest('removeAssociations', [source, d.groupId, [target]])
  const fresh = await this.app.apiRequest('getAssociations', [d.sourceNodeId, false])
  this.associationsMap[d.sourceNodeId] = fresh.result
  this.setAssociations(this.associationsMap)
  this.paintGraph()
}
```

## Acceptance Criteria

- [ ] Clicking an edge in connect mode shows the delete snackbar
- [ ] Snackbar names source and target nodes
- [ ] Confirming removes the association and repaints the graph
- [ ] Cancelling / snackbar timeout deselects edge with no change
- [ ] Clicking edge outside connect mode does not show snackbar
