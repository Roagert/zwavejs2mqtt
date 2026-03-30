---
id: TASK-006
title: Fix loadAllAssociations stuck loading state on exception
priority: P1
status: done
depends_on: []
created: 2026-03-30
---

## Description
`loadAllAssociations()` sets `this.loading = true` at the top but relies on reaching `this.loading = false` at the bottom. If anything between them throws (e.g. `clearAssociations()`, `nodes.filter()`), the loading spinner never clears and the UI hangs forever.

## Root Cause
`GroupGraph.vue:838-876` — no try/finally around the function body.

## Acceptance Criteria
- [ ] `this.loading` is always set to false when `loadAllAssociations()` exits, regardless of how it exits
- [ ] `this.loadProgress` is always cleared on exit

## Implementation Notes
Wrap the function body in try/finally:
```js
async loadAllAssociations() {
    this.loading = true
    try {
        ...
    } finally {
        this.loading = false
        this.loadProgress = ''
    }
}
```

## Execution Log
| Iteration | Outcome | Notes |
|-----------|---------|-------|
