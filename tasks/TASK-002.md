---
id: TASK-002
title: Fix fragile undefined >= 0 guard in removeNode
priority: P1
status: done
depends_on: [TASK-001]
created: 2026-03-30
---

## Description
`removeNode()` guards with `if (index >= 0)` where `index = this.nodesMap.get(n.id)`. If `n.id` is not in the map, `get()` returns `undefined`. In JavaScript `undefined >= 0` evaluates to `false`, so the block is skipped — which is the intended behaviour, but it's accidentally correct rather than explicitly correct. The intent should be expressed clearly.

## Root Cause
`base.js:347` — `if (index >= 0)` should be `if (index !== undefined)`

## Acceptance Criteria
- [ ] Guard uses explicit `!== undefined` check
- [ ] Behaviour is unchanged: unknown node IDs are silently ignored

## Implementation Notes
Single line change in `src/stores/base.js`:
```js
if (index !== undefined) {
```

## Execution Log
| Iteration | Outcome | Notes |
|-----------|---------|-------|
