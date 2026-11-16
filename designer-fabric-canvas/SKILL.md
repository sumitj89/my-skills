---
name: designer-fabric-canvas
description: Fabric.js canvas, layers, hooks, editor architecture, undo/redo, object classification, memory management, and serialization. Use when touching canvas objects, layers (addToContentsLayer, LayerUtils), hooks (useFabricCanvas, useFabricCanvasHandler, useActiveFabricObject), object types (designerObjectType vs obj.type), serialization (additionalProps, cloneFabricObject), memory (withTempCanvas, fabricMemory.js), or editor history (saveCurrentState, mute).
---

# Designer Fabric Canvas

Single source of truth for working with Fabric.js, canvas layers, custom hooks, editor history, domain object types, serialization, and memory management in the Designer editor (`react-fabric-page` + `fabricCustom.js`).

---

## Non-Negotiables Checklist

Apply these rules on every canvas change:

- [ ] **No duplicate undo steps**: Never call `editor.saveCurrentState()` after auto-saving events (`object:added`, `object:modified`, `object:removed`, `activeObject.fireModified()`).
- [ ] **Batch operations muted**: Wrap multi-object additions/modifications in `editor.mute()` ... `editor.unmute()` and commit exactly one `editor.saveCurrentState()`.
- [ ] **Silent UI helpers**: Temporary guides, rulers, and bounding indicators must use `canvas.addSilently(obj)`.
- [ ] **No raw `canvas.add(obj)` for design items**: Design objects must be placed in a layer via `await canvas.addToContentsLayer({ obj, layerId, designerObjectType })`.
- [ ] **Domain object checks use `designerObjectType`**: Check `obj.designerObjectType === Consts.OBJECT_TYPE_*` for business entities (text, clipart, shape, image). Use `obj.type` only for low-level Fabric primitives (`group`, `layer`, `rect`, `path`).
- [ ] **Never clone with native `obj.clone()` directly**: Use `cloneFabricObject(obj)` from `objectUtils.js`, or pass `additionalProps` from `Consts.js`.
- [ ] **Pass `additionalProps` when serializing**: Never call `canvas.toJSON()` or `obj.toObject()` without `additionalProps`.
- [ ] **Declare hook dependencies**: Every state/prop used inside `useFabricCanvasHandler` must be in the `dependencies` array to prevent stale closures.
- [ ] **No unmanaged temporary canvases**: Always use `fabric.util.withTempCanvas(w, h, cb)` instead of `document.createElement('canvas')`.

---

## Detailed Reference Pointers

Consult these dedicated references for architectural deep dives and implementation patterns:

### 1. Fabric Package & Override Lifecycle
* **Pointer:** [`references/fabric-architecture.md`](references/fabric-architecture.md)
* **Triggers:** Where Fabric lives (`react-fabric-page`), local development linking (`scripts/dev-fabric.js` / `FABRIC_PATH`), static prototype monkey-patching in `fabricCustom.js`, runtime editor initialization in `customCallback(editor)`, and where new canvas code belongs.

### 2. Undo/Redo & History Engine
* **Pointer:** [`references/undo-redo.md`](references/undo-redo.md)
* **Triggers:** Managing history stacks (`urStack`, `urNestedStack`), avoiding double undo steps with `RibbonController` auto-save events, batch muting via `editor.mute()`, and silent UI objects.

### 3. React Canvas Hooks & Selection
* **Pointer:** [`references/hooks.md`](references/hooks.md)
* **Triggers:** Using `useFabricCanvas`, `useFabricEditor`, preventing stale closures in `useFabricCanvasHandler`, and handling single vs multi-selections with `useActiveFabricObject`.

### 4. Object Types & Entity Classification
* **Pointer:** [`references/object-types.md`](references/object-types.md)
* **Triggers:** Distinguishing `obj.type` (Fabric primitives) from `obj.designerObjectType` (`Consts.OBJECT_TYPE_*`), auto-assigning domain types via `assignDesignerObjectType`, and unpacking selections via `selectionUtils`.

### 5. Object Serialization & Cloning
* **Pointer:** [`references/serialization-cloning.md`](references/serialization-cloning.md)
* **Triggers:** Preventing custom property loss during cloning (`cloneFabricObject`, `cloneWithNested`) and canvas serialization (`canvas.toJSON(additionalProps)`).

### 6. Layer Hierarchy & Insertion
* **Pointer:** [`references/layers.md`](references/layers.md)
* **Triggers:** Canvas layer tree (`canvas._objects` as `fabric.Layer`), inserting objects via `addToContentsLayer`, and managing printable vs system layers (`LayerUtils`).

### 7. Canvas Memory & GPU Ceilings
* **Pointer:** [`references/canvas-memory.md`](references/canvas-memory.md)
* **Triggers:** Mobile Safari ~384MB canvas ceiling, buffer zeroing via `fabric.util.releaseCanvas`, temporary canvases with `withTempCanvas`, deep disposal architecture, and `fabric.PatternImage`.
