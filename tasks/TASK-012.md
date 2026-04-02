---
id: TASK-012
title: Wire vis-network manipulation API for drag-to-connect
priority: P1
status: open
depends_on: [TASK-011]
prd: PRD-interactive-connector
created: 2026-04-02
---

## Description

Watch `connectMode` and call `enableConnectMode()` / `disableConnectMode()` to toggle vis-network's built-in manipulation API.

## Implementation

```js
watch: {
  connectMode(val) {
    val ? this.enableConnectMode() : this.disableConnectMode()
  },
},

methods: {
  enableConnectMode() {
    if (!this.network) return
    this.network.setOptions({
      manipulation: {
        enabled: true,
        initiallyActive: true,
        addEdge: (data, callback) => {
          callback(null) // prevent vis adding a real edge
          this.onConnectDrop(data.from, data.to)
        },
      },
    })
  },

  disableConnectMode() {
    if (!this.network) return
    this.network.setOptions({ manipulation: { enabled: false } })
    this.pendingDeleteEdge = null
    this.showDeleteSnackbar = false
  },
}
```

vis-network renders the ghost edge and port handles automatically when manipulation is enabled — no custom canvas code needed.

## Acceptance Criteria

- [ ] Entering connect mode enables vis manipulation with `addEdge` callback
- [ ] Dragging from a node shows vis-network's ghost edge cursor
- [ ] Exiting connect mode disables manipulation cleanly
- [ ] Re-painting the graph (view mode switch, stabilize) does not re-enable manipulation unexpectedly
