---
id: TASK-003
title: Fix wrong property response.message in responseHandler
priority: P1
status: done
depends_on: []
created: 2026-03-30
---

## Description
`responseHandler` in `ConfigApis.js` throws `Error(response.message || 'Authentication failed')`. But `response` here is an Axios response object — the server message lives at `response.data.message`, not `response.message`. `response.message` is always `undefined`, so the thrown error always reads "Authentication failed" regardless of what the server actually returned.

## Root Cause
`ConfigApis.js:16`
```js
throw Error(response.message || 'Authentication failed')
//                ↑ undefined — should be response.data.message
```

## Acceptance Criteria
- [ ] Error message matches `response.data.message` when the server provides one
- [ ] Falls back to `'Authentication failed'` when `response.data.message` is absent

## Implementation Notes
`src/apis/ConfigApis.js` line 16:
```js
throw Error(response.data?.message || 'Authentication failed')
```

## Execution Log
| Iteration | Outcome | Notes |
|-----------|---------|-------|
