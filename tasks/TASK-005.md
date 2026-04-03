---
id: TASK-005
title: Remove dead v-if="false" alert markup in ZwaveGraph
priority: P2
status: done
depends_on: []
created: 2026-03-30
---

## Description

After removing the Associations mode, the warning alert that said "no association data loaded" was changed to `v-if="false"` to silence it. This is dead markup — an element that can never render. It should be deleted entirely.

## Root Cause

`ZwaveGraph.vue` — alert block with `v-if="false"` that was previously checking `!hasAssociationData`.

## Acceptance Criteria

- [ ] The `v-alert` block with `v-if="false"` is removed from the template
- [ ] No visible change to UI

## Implementation Notes

Find and remove the `<v-alert v-if="false" ...>` block in the template section of `src/components/custom/ZwaveGraph.vue`.

## Execution Log

| Iteration | Outcome | Notes |
|-----------|---------|-------|
