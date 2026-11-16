# Undo/Redo & History Architecture

Single source of truth for the undo/redo engine, event-driven state snapshots, batch operations, and silent canvas objects.

---

## 1. History Engine Overview

The editor manages multiple undo/redo stacks on the `CanvasEditor` instance (`editor`), exported via `useFabricEditor()`:
- **`urStack`**: Main canvas history stack (records design modifications, additions, deletions).
- **`urNestedStack`**: History for nested groups and compound shapes.
- **`urFreeTransformStack`**: History for perspective and corner pin transforms.
- **`urEraserStack`**: History for freehand eraser strokes.

---

## 2. Auto-Saving Canvas Events in `RibbonController`

`RibbonController.js` subscribes to canvas events via `useFabricCanvasHandler` and automatically triggers an undo snapshot:

| Event Name | When It Fires | `RibbonController` Action |
| :--- | :--- | :--- |
| **`object:added`** | New object placed on canvas/layer | Calls `editor.saveCurrentState()` |
| **`object:modified`** | Object moved, scaled, rotated, or modified via controls (mouse up), or fired manually via `canvas.fire('object:modified', { target })` or `activeObject.fireModified()` | Calls `editor.saveCurrentState()` |
| **`object:removed`** | Object deleted from layer/canvas | Calls `editor.saveCurrentState()` |
| **`freeTransform:pointsModified`** | Free-transform corner/perspective pins moved | Calls `editor.saveCurrentState()` |

---

## 3. The Core Rule: Prevent Duplicate Undo Steps

Because the events above automatically trigger `editor.saveCurrentState()`, **never call `editor.saveCurrentState()` after firing an auto-saving event**. Doing both pushes two identical snapshots onto `urStack`, forcing the user to press Ctrl+Z twice to undo a single user action.

### Scenario A: Standard Addition / Transformation (Let Event Save)
```javascript
// Good - object:added auto-saves
await canvas.addToContentsLayer({ obj: newObject, layerId });
canvas.fire('object:added', { target: newObject });
canvas.requestRenderAll();
// Do NOT call editor.saveCurrentState() here!

// Good - activeObject.fireModified() auto-saves
activeObject.set('opacity', 0.8);
activeObject.fireModified(); // Fires canvas 'object:modified'
canvas.requestRenderAll();
// Do NOT call editor.saveCurrentState() here!

// Bad - triggers two snapshots! (User has to press Ctrl+Z twice)
activeObject.set('opacity', 0.8);
activeObject.fireModified();
editor.saveCurrentState(); // DUPLICATE SNAPSHOT!
```

### Scenario B: Manual Save (When No Auto-Saving Event Fires)
Only call `editor.saveCurrentState()` manually when an action modifies properties, adjusts layer hierarchy, or updates state without firing an auto-saving event:
```javascript
// Good - manual property adjustment without firing object:modified
target.set({ customProperty: 'value' });
target.setCoords();
editor.saveCurrentState(); // Manually commit the step
canvas.requestRenderAll();
```

---

## 4. Batch Operations (Single Undo Step)

When adding or modifying multiple objects in sequence, each operation would normally fire `object:added` or `object:modified`, creating dozens of intermediate undo steps. 

**Rule**: Always wrap batch updates in `editor.mute()` ... `editor.unmute()`, then commit exactly one clean undo step:

```javascript
// Good - batch operation collapsed into a single undo step
editor.mute(); // Suppresses RibbonController auto-save during batch
try {
  for (const item of items) {
    await canvas.addToContentsLayer({ obj: item, layerId });
  }
} finally {
  editor.unmute(); // Always unmute in finally block
}

// Commit exactly ONE clean undo step for the whole batch
editor.saveCurrentState();
canvas.requestRenderAll();

// Bad - creates 50 separate undo steps for 50 items!
for (const item of items) {
  await canvas.addToContentsLayer({ obj: item, layerId });
  canvas.fire('object:added', { target: item }); // Floods urStack
}
```

---

## 5. Silent Helper Objects (Never Enter Undo History)

Temporary UI helpers, crop marks, rulers, guidelines, measurement indicators, or temporary bounding boxes must **never** enter `urStack`.

### Adding and Removing Silently
Use `canvas.addSilently` and `canvas.removeSilently`:
```javascript
// Good - temporary UI helper does not pollute undo history
canvas.addSilently(tempGuideLine);
canvas.requestRenderAll();

// When removing:
canvas.removeSilently(tempGuideLine);
canvas.requestRenderAll();

// Or using objectUtils helper:
import { addObjectToCanvasSilently } from 'src/App/modules/Editor/objectUtils';
addObjectToCanvasSilently(canvas, editor, guideObject);
```
