---
id: TASK-007
title: Fix removeMember stuck loading spinner on API exception
priority: P1
status: done
depends_on: []
created: 2026-03-30
---

## Description

`removeMember()` sets `this.removeLoading[key] = true` before the API call. If `apiRequest` throws (network error, timeout), the line that resets it to `false` is never reached. The remove button shows a loading spinner forever.

## Root Cause

`GroupGraph.vue:1555-1558` — no try/catch around the API await.

## Acceptance Criteria

- [ ] `removeLoading[key]` is always reset to false when `removeMember()` exits
- [ ] On API error, button returns to normal state (not stuck loading)

## Implementation Notes

```js
async removeMember(group, member) {
    ...
    this.removeLoading = { ...this.removeLoading, [key]: true }
    try {
        const resp = await this.app.apiRequest(...)
        this.removeLoading = { ...this.removeLoading, [key]: false }
        if (resp.success) { ... }
    } catch {
        this.removeLoading = { ...this.removeLoading, [key]: false }
    }
}
```

Or use try/finally for the reset.

## Execution Log

| Iteration | Outcome | Notes |
|-----------|---------|-------|
