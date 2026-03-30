---
id: TASK-008
title: Sync association changes back to Pinia store after add/remove
priority: P1
status: done
depends_on: []
created: 2026-03-30
---

## Description
`addMember()` and `removeMember()` both update `this.associationsMap[nodeId]` locally after a successful API operation but never call `this.setAssociations(this.associationsMap)` to persist the change to the Pinia store. When the user navigates away and returns, `mounted()` loads from the stale store — the added/removed association is gone from the view until a manual Refresh.

## Root Cause
`GroupGraph.vue:1546` and `GroupGraph.vue:1562` — local map updated, store not synced.

## Acceptance Criteria
- [ ] After `addMember` succeeds, Pinia `associationsMap` reflects the new member
- [ ] After `removeMember` succeeds, Pinia `associationsMap` reflects the removal
- [ ] Navigating away and back shows the updated associations without a Refresh

## Implementation Notes
After updating `this.associationsMap[this.managePanelNode.id]` in both methods, add:
```js
this.setAssociations(this.associationsMap)
```

## Execution Log
| Iteration | Outcome | Notes |
|-----------|---------|-------|
