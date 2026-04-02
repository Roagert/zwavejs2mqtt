---
id: TASK-011
title: Add Connect Mode toggle button to GroupGraph toolbar
priority: P1
status: open
depends_on: []
prd: PRD-interactive-connector
created: 2026-04-02
---

## Description

Add a `connectMode` toggle to the GroupGraph toolbar and the data/watcher plumbing around it.

## Changes

**Data:**

```js
connectMode: false,
```

**Watcher:**

```js
viewMode(val) {
  if (val !== 'associations') this.connectMode = false
}
```

**Toolbar button** (alongside existing Load / Live / Demo buttons):

```html
<v-btn
  :color="connectMode ? 'primary' : undefined"
  :disabled="viewMode !== 'associations'"
  icon="add_link"
  size="small"
  variant="tonal"
  title="Connect Mode — drag to create associations"
  @click="connectMode = !connectMode"
/>
```

**Hint banner** shown while connect mode is active:

```html
<v-alert v-if="connectMode" type="info" variant="tonal" density="compact" closable>
  Drag from a node to another node to create an association.
  Click an edge to delete it.
</v-alert>
```

## Acceptance Criteria

- [ ] Button appears in toolbar, disabled outside Devices view
- [ ] Toggling button flips `connectMode`
- [ ] Switching view mode while connect mode is active sets `connectMode = false`
- [ ] Hint banner visible while connect mode is on, dismissible
