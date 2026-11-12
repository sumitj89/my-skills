---
name: designer-fabric-canvas
description: Fabric.js canvas, layers, hooks, and editor undo/redo architecture. Use when touching canvas objects, layer management (addToContentsLayer, LayerUtils), canvas event hooks (useFabricCanvas, useFabricCanvasHandler, useActiveFabricObject), custom fabric classes (PatternImage, fabricCustom.js), or editor history (saveCurrentState, muteUndoRedoStack).
---

# Designer Fabric Canvas

Single source of truth for working with Fabric.js, canvas layers, custom hooks, and the undo/redo history engine in the Designer editor (`react-fabric-page` + `fabricCustom.js`).

---

## 1. Undo/Redo & Event Architecture

The editor manages multiple history stacks (`urStack`, `urNestedStack`, `urPathStack`, `urFreeTransformStack`, `urEraserStack`) on the `CanvasEditor` instance (`editor`).

### Auto-Saving Canvas Events in `RibbonController`
`RibbonController.js` subscribes to canvas events via `useFabricCanvasHandler` and automatically saves editor state:

| Event Name | When It Fires | `RibbonController` Action |
| :--- | :--- | :--- |
| **`object:added`** | New object placed on canvas/layer | Calls `editor.saveCurrentState()` |
| **`object:modified`** | Object **scaled**, **rotated**, **dragged/moved**, or updated via controls (mouse up), or fired manually via `canvas.fire('object:modified', { target })` or `activeObject.fireModified()` | Calls `editor.saveCurrentState()` |
| **`object:removed`** | Object deleted from layer/canvas | Calls `editor.saveCurrentState()` |
| **`freeTransform:pointsModified`** | Free-transform corner/perspective pins moved | Calls `editor.saveCurrentState()` |

### The Core Rule: Prevent Duplicate Undo Steps
Because the events above already trigger an automatic `saveCurrentState()`, **never call `editor.saveCurrentState()` after firing an auto-saving event**. Doing both pushes two identical snapshots onto the undo stack, forcing the user to press Ctrl+Z twice.

#### 1. Standard Object Addition / Transformation (Let Event Save)
Add the object to the layer and let the event trigger the save automatically:
```javascript
// Adding an object: object:added auto-saves
await canvas.addToContentsLayer({ obj: newObject, layerId });
canvas.fire('object:added', { target: newObject });
canvas.requestRenderAll();
// Do NOT call editor.saveCurrentState() here!

// Modifying an object: object:modified auto-saves
activeObject.set('opacity', 0.8);
activeObject.fireModified(); // Fires canvas 'object:modified'
canvas.requestRenderAll();
// Do NOT call editor.saveCurrentState() here!
```

#### 2. Manual Save (When No Auto-Saving Event Fires)
Only call `editor.saveCurrentState()` manually when an action modifies properties, adjusts layering, or updates state without firing an auto-saving event:
```javascript
target.set({ customProperty: 'value' });
target.setCoords();
editor.saveCurrentState(); // Manually commit the step
canvas.requestRenderAll();
```

#### 3. Batch Operations (Grouping into a Single Undo Step)
When adding or modifying multiple objects that would trigger multiple events in sequence, mute the editor to prevent intermediate auto-saves, then commit exactly one clean undo step:
```javascript
editor.mute(); // Suppresses RibbonController auto-save during batch
try {
  for (const item of items) {
    await canvas.addToContentsLayer({ obj: item, layerId });
  }
} finally {
  editor.unmute();
}

// Commit exactly ONE clean undo step for the entire batch
editor.saveCurrentState();
canvas.requestRenderAll();
```

#### 4. Silent Helper Objects
For temporary UI helpers, markers, indicators, or rulers that should never enter the undo/redo stack:
- Use `canvas.addSilently(...objects)` and `canvas.removeSilently(...objects)`.

---

## 2. Core React Fabric Hooks

All primary hooks are exported from `react-fabric-page`:

```javascript
import {
  ActiveObject,
  fabric,
  useActiveFabricObject,
  useFabricCanvas,
  useFabricCanvasHandler,
  useFabricEditor
} from 'react-fabric-page';
```

### `useFabricCanvas()`
Returns the active Fabric `canvas` instance from React context.
```javascript
const canvas = useFabricCanvas();
```

### `useFabricEditor()`
Returns the `CanvasEditor` instance (`editor`), providing access to `saveCurrentState()`, `mute()`, `unmute()`, `deleteObjects()`, `copyObjects()`.
```javascript
const editor = useFabricEditor();
```

### `useFabricCanvasHandler(dataOrCallback, dependencies = [])`
Subscribes to canvas events for the lifetime of the component and cleans them up via `canvas.off(handler)` on unmount.

- Can take an event mapping object or a callback `(canvas) => ({ ... })`.
- **Always pass closed-over props/state in the `dependencies` array** to avoid stale closures.

```javascript
useFabricCanvasHandler({
  'selection:created': (e) => handleSelection(e.target),
  'selection:updated': (e) => handleSelection(e.target),
  'selection:cleared': () => clearSelection()
}, [handleSelection, clearSelection]);
```

### `useActiveFabricObject(Wrapper = ActiveObject)`
Returns `[activeObject, update]` where `activeObject` is an instance of `ActiveObject` wrapping the current selection.

- Works transparently with both a single object and multi-selections (`fabric.ActiveSelection`).
- **Key Methods**:
  - `activeObject.get(key)`: Gets property value across selection; returns `null` if objects have differing values.
  - `activeObject.set(key, value)`: Sets property on all selected objects.
  - `activeObject.getTargetObjects()`: Returns array of underlying fabric objects.
  - `activeObject.fireModified()`: Fires canvas `object:modified` event (which triggers auto-save).
  - `activeObject.isEmpty()` / `activeObject.size()`: Selection state checks.
  - `update()`: Manually triggers re-sync with canvas selection.

```javascript
const [activeObject, updateActiveObject] = useActiveFabricObject();

const handleScaleChange = (scale) => {
  activeObject.set('scaleX', scale);
  activeObject.set('scaleY', scale);
  activeObject.fireModified();
  canvas.requestRenderAll();
};
```

---

## 3. Layer Architecture & Object Insertion

In Designer V2, **`canvas._objects` is an array of `fabric.Layer` instances**, not raw drawing objects.

### Layer Hierarchy
```
Canvas
 ├── fabric.Layer (id: 'background')
 ├── fabric.Layer (id: 'contents' / custom layer ID)  <-- Design objects live here
 ├── fabric.Layer (id: 'white', 'varnish', etc.)     <-- Special layers
 ├── fabric.Layer (layerType: 'smart-guide')
 └── fabric.Layer (layerType: 'grid')
```

### Never Use Raw `canvas.add(obj)` for Design Objects
Raw `canvas.add(obj)` pushes the object directly onto `canvas._objects`, placing it outside the layer tree and breaking stacking order, visibility toggles, and export.

### Always Add Objects to a Layer
Use `canvas.addToContentsLayer`:
```javascript
await canvas.addToContentsLayer({
  obj: fabricObjectOrArray,
  layerId: currentLayerId || 'contents', // defaults to 'contents'
  designerObjectType: 'clipart',         // optional metadata
  index: targetIndex                     // optional z-index inside layer
});
```
Or insert directly into an existing layer:
```javascript
const layer = canvas.getLayerById(layerId);
if (layer) {
  layer.add(obj); // or layer.insertAt(obj, index);
  obj.layer = layer;
}
```

### Layer Types
- **Printable Layers**: User content layers (`isPrintableLayer(layer)`).
- **System / Non-Printable Layers**: `background`, `grid`, `smart-guide`, `guide`, `bleedLayer`, `cropMark`, `cutting-layer`, `frame-guide`.
- **Creating Layers**: Use `LayerUtils.addNewLayer({ canvas, layerType, width, height, margins })`.

---

## 4. Custom Fabric Classes & Extensions (`fabricCustom.js`)

`designer_ui/src/App/modules/Editor/fabricCustom.js` extends native Fabric classes.

### `fabric.PatternImage`
A custom image class (`fabric.util.createClass(fabric.Image)`) supporting tiled fill patterns, color replacement, and white ink handling (`applyWhite`).

- **Instantiation**:
  ```javascript
  fabric.PatternImage.fromURL(url, (patternImg, isError) => {
    if (!isError && patternImg) {
      // Use patternImg as pattern fill or canvas object
    }
  }, { crossOrigin: 'anonymous' });
  ```

### Essential Fabric Object Mutation Rules
1. **Coordinates**: Always call `obj.setCoords()` after modifying `left`, `top`, `scaleX`, `scaleY`, `angle`, `width`, or `height`.
2. **Repainting**: Always call `canvas.requestRenderAll()` (or `canvas.renderAll()`) after changes.
3. **Dirty Flags**: If caching is enabled on groups or custom shapes, call `obj.markDirty()` to invalidate the cached canvas.
4. **Group Coordinates**: To get absolute canvas coordinates of an object inside a group/layer, use `getAbsoluteTransform(obj)` from `objectUtils.js`.
