---
id: TASK-001
title: Fix stale nodesMap indices after removeNode splice
priority: P0
status: done
depends_on: []
created: 2026-03-30
---

## Description
`removeNode()` in `base.js` calls `this.nodes.splice(index, 1)` but never updates `nodesMap`. Every node that was stored at an index greater than the removed node now has a stale map entry pointing to the wrong array position. All subsequent `getNode()` calls for those nodes silently return the wrong node object.

## Root Cause
`base.js:344-350`
```js
removeNode(n) {
    const index = this.nodesMap.get(n.id)
    if (index >= 0) {
        this.nodesMap.delete(n.id)
        this.nodes.splice(index, 1)  // ← shifts all subsequent indices by -1
        // nodesMap entries for ids at indices > removed are now off by 1
    }
}
```

## Acceptance Criteria
- [ ] After removing node at index 2 of a 5-node array, nodes at former indices 3 and 4 are still reachable via `getNode()` and return correct data
- [ ] `nodesMap.size` equals `nodes.length` after removal
- [ ] All map entries point to the correct array index after any removal

## Implementation Notes
After the splice, iterate over all nodesMap entries and decrement every index > removed index by 1:
```js
for (const [id, idx] of this.nodesMap.entries()) {
    if (idx > index) this.nodesMap.set(id, idx - 1)
}
```
File: `src/stores/base.js` around line 344.

## Execution Log
| Iteration | Outcome | Notes |
|-----------|---------|-------|
