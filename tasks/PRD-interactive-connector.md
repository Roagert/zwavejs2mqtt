---
id: PRD-interactive-connector
title: Interactive Association Connector Builder
status: approved
created: 2026-04-02
---

## Problem

Creating a Z-Wave association today requires 5+ steps through the side panel: click node → wait for panel → expand group → type target in combobox → click Add. There is no spatial relationship between steps — the user has to mentally map graph positions to list entries. Removing a bad association requires the same panel flow.

The graph already shows associations as edges. Users should be able to draw and delete those edges directly on the canvas.

---

## Goals

- Let the user drag from one node to another to create an association
- Let the user select an edge and delete it to remove an association
- Reuse the existing `DialogAssociation` form for group/endpoint selection
- Zero changes to the existing side-panel flow (it stays as-is for power users)

---

## Non-Goals

- Multi-endpoint source drag (covered by side panel)
- Works in Groups / Node→Groups / Shared views — Devices view only
- Mobile touch drag (out of scope for now)

---

## User Journey

```
1. User opens Association Graph → Devices view
2. Clicks "Connect" toggle button in toolbar
3. Graph enters connect mode:
   - Cursor changes to crosshair
   - A hint banner reads "Drag from a node to another to add an association"
   - Existing edges become thinner/dimmed to reduce noise
4. User hovers a source node → a pulsing port handle appears on its edge
5. User drags from the handle toward a target node
6. A dashed ghost edge follows the cursor
   - Valid target nodes highlight green on hover
   - The source node highlights blue
   - Nodes with no available group capacity are dimmed
7. User drops on a target node:
   - DialogAssociation opens pre-filled:
       • Source node locked (read-only)
       • Target node locked (read-only)
       • User picks group and optionally endpoint
   - On confirm: association created, graph repaints, connect mode stays active
   - On cancel: ghost edge removed, no change
8. User drops on empty canvas or the same node: drag cancelled silently
9. User clicks "Connect" toggle again to exit connect mode
```

**Edge deletion flow:**

```
1. User clicks an existing association edge
2. Edge highlights + a small tooltip appears: "Press Delete to remove"
3. User presses Delete (or clicks ✕ in tooltip)
4. Confirmation snackbar with Undo (3 s window)
5. On confirm / timeout: removeAssociations called, edge removed
```

---

## Technical Design

### Connect mode toggle

Add a `connectMode: false` data property. A toolbar button toggles it.

When toggled on, call:

```js
this.network.setOptions({
  manipulation: {
    enabled: true,
    initiallyActive: true,
    addEdge: (data, callback) => {
      callback(null) // prevent vis from drawing a real edge
      this.onConnectDrop(data.from, data.to)
    },
  },
})
```

When toggled off:

```js
this.network.setOptions({ manipulation: { enabled: false } })
```

vis-network's built-in `addEdge` manipulation handles the drag-ghost-edge rendering with no extra canvas code.

### `onConnectDrop(fromVisId, toVisId)`

```
1. Parse actual nodeIds from vis IDs (strip "node_" prefix)
2. Guard: source === target → return
3. Guard: toVisId type === 'group' → return (groups view only)
4. Open DialogAssociation with { sourceNodeId, targetNodeId } pre-filled
5. On dialog confirm: call addMember() with the selected group
```

### Edge click → delete

```js
this.network.on('selectEdge', (params) => {
  if (!this.connectMode) return
  const edgeId = params.edges[0]
  const edgeData = visEdges.get(edgeId)
  if (!edgeData?._assocData) return
  this.pendingDeleteEdge = edgeData._assocData
  this.showDeleteHint = true
})
```

Store `_assocData: { sourceNodeId, sourceEndpoint, groupId, targetNodeId, targetEndpoint }` on each edge when building the DataSet.

`deleteSelectedEdge()` calls `removeMember()` with the stored assocData, then hides the hint.

### Visual feedback during drag

vis-network fires `dragStart` / `dragging` / `dragEnd` — but in manipulation mode it fires `addEdge` events instead. Use the `manipulation.addEdge` callback exclusively; no custom canvas overlay needed.

For node highlighting during drag, override `beforeDrawing` to tint candidate nodes while `connectMode && dragging`.

### Devices view constraint

Connect mode button is disabled (greyed out with tooltip) when `viewMode !== 'associations'`. A watcher clears `connectMode` if the user switches view while it is active.

---

## Acceptance Criteria

- [ ] "Connect" toggle button visible in toolbar, disabled outside Devices view
- [ ] Dragging from node A to node B opens DialogAssociation pre-filled with both nodes
- [ ] Cancelling the dialog leaves no new association and no ghost edge
- [ ] Confirming the dialog creates the association, graph repaints, Pinia store updated
- [ ] Clicking an edge in connect mode shows delete hint
- [ ] Pressing Delete removes the association with a 3-second undo window
- [ ] Switching view mode while connect mode is active automatically exits connect mode
- [ ] Existing side-panel add/remove flow is unaffected

---

## Affected Files

| File | Change |
|------|--------|
| `src/components/custom/GroupGraph.vue` | Connect mode toggle, `onConnectDrop`, edge delete, vis manipulation options |
| `src/components/dialogs/DialogAssociation.vue` | Accept optional `lockedSource` / `lockedTarget` props to pre-fill and lock fields |
| `src/components/custom/GroupGraph.vue` (edge build) | Store `_assocData` on each visEdge during `paintAssociationsView` |
