# React Fabric Hooks & Event Subscriptions

Single source of truth for consuming Fabric.js canvas, editor instances, canvas event listeners, and selection wrappers in React components.

---

## 1. Core Hooks Overview

All primary canvas hooks are imported from `react-fabric-page`:

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

---

## 2. `useFabricCanvas()`

Returns the active Fabric `canvas` instance from React context.

```javascript
const canvas = useFabricCanvas();

// Always guard against null during initial mount:
if (!canvas) {
  return null;
}
```

* **When to use**: When you need to read canvas dimensions, render (`canvas.requestRenderAll()`), add/remove layers, query layers, or run silent additions.

---

## 3. `useFabricEditor()`

Returns the `CanvasEditor` instance (`editor`).

```javascript
const editor = useFabricEditor();
```

* **Primary Methods**:
  - `editor.saveCurrentState()`: Commits an undo snapshot onto `urStack`.
  - `editor.mute()` / `editor.unmute()`: Suppresses/restores auto-save during batch mutations.
  - `editor.deleteObjects(objects)`: Deletes specified objects with history tracking.
  - `editor.copyObjects(objects)`: Copies objects to internal clipboard.

---

## 4. `useFabricCanvasHandler(dataOrCallback, dependencies = [])`

Subscribes to canvas events during the component's lifetime and automatically unbinds them (`canvas.off(handler)`) when the component unmounts or dependencies change.

### The Stale Closure Rule: Always Specify Dependencies
Because event handlers capture component scope at registration time, **any prop, state variable, or callback referenced inside an event handler must be declared in the `dependencies` array**. Omitting dependencies creates stale closures where handlers read obsolete state.

```javascript
// Good - dependencies array prevents stale closures and re-binds cleanly
const [zoomLevel, setZoomLevel] = useState(1);

useFabricCanvasHandler({
  'mouse:wheel': (opt) => {
    handleCanvasZoom(opt, zoomLevel);
  },
  'selection:created': (e) => {
    handleSelectionChange(e.target);
  },
  'selection:cleared': () => {
    handleSelectionChange(null);
  }
}, [zoomLevel, handleCanvasZoom, handleSelectionChange]);

// Bad - missing dependencies causes handler to see initial zoomLevel forever!
useFabricCanvasHandler({
  'mouse:wheel': (opt) => {
    handleCanvasZoom(opt, zoomLevel); // Stale zoomLevel!
  }
}, []); // Empty deps array!

// Bad - manually calling canvas.on without cleanup leaks memory!
useEffect(() => {
  canvas.on('object:moving', handleMove); // Leaks listener on unmount!
}, [canvas]);
```

---

## 5. `useActiveFabricObject(Wrapper = ActiveObject)`

Returns `[activeObject, updateActiveObject]`. The `activeObject` instance is a unified wrapper that transparently normalizes operations across **single selections** and **multi-selections** (`fabric.ActiveSelection`).

```javascript
const [activeObject, updateActiveObject] = useActiveFabricObject();
```

### Essential `ActiveObject` Methods

| Method | Behavior |
| :--- | :--- |
| **`activeObject.get(key)`** | Returns property value. If multiple objects are selected and have differing values, returns `null` (or empty string for text). |
| **`activeObject.set(key, value)`** | Sets the property across all selected objects uniformly. |
| **`activeObject.getTargetObjects()`** | Returns a flat array of underlying native Fabric objects in the selection. |
| **`activeObject.fireModified()`** | Fires `object:modified` on the canvas for the active target, triggering automatic state save in `RibbonController`. |
| **`activeObject.isEmpty()`** | Returns `true` if no object is currently selected. |
| **`activeObject.size()`** | Returns the count of selected objects. |
| **`updateActiveObject()`** | Forces the wrapper to re-sync with the current canvas selection (use when object properties mutate outside React state). |

### Typical Inspector / Tool Panel Pattern
```javascript
const [activeObject, updateActiveObject] = useActiveFabricObject();
const canvas = useFabricCanvas();

const handleOpacityChange = (newOpacity) => {
  if (activeObject.isEmpty()) return;

  activeObject.set('opacity', newOpacity);
  activeObject.fireModified(); // Triggers auto-save
  canvas.requestRenderAll();
};

const handleCustomExternalChange = () => {
  // If external logic altered canvas objects without standard events:
  updateActiveObject(); // Re-sync React wrapper with canvas selection
};
```
