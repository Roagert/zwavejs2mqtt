---
id: TASK-010
title: Add lockedSource / lockedTarget props to DialogAssociation
priority: P1
status: open
depends_on: []
prd: PRD-interactive-connector
created: 2026-04-02
---

## Description

DialogAssociation.vue lets the user pick source and target freely. Add optional `lockedSource` and `lockedTarget` props so the connector builder can pre-fill and lock both fields when opening the dialog from a canvas drag.

## Props to add

```js
props: {
  lockedSource: { type: Object, default: null }, // { nodeId, name }
  lockedTarget: { type: Object, default: null }, // { nodeId, name }
}
```

## Behaviour changes

- When `lockedSource` is set: source node field shows the node name, is disabled, source endpoint defaults to 0 and is hidden
- When `lockedTarget` is set: target node combobox shows the node name and is disabled
- Validation (`allowedAssociation`) still runs normally with the locked values
- Emitted `add` payload is unchanged

## Acceptance Criteria

- [ ] Passing `lockedSource` pre-fills and disables the source field
- [ ] Passing `lockedTarget` pre-fills and disables the target field
- [ ] Association validation still fires when dialog opens
- [ ] Normal (unlocked) dialog usage is unaffected
