# Canvas Memory Management & Safari GPU Ceilings

Single source of truth for canvas memory management, Safari memory ceilings, temporary canvas allocation, and deep disposal architecture.

---

## 1. The Mobile Safari GPU Canvas Ceiling

iPad Safari and mobile browsers enforce a strict GPU memory ceiling on all active `<canvas>` elements (~384 MB total across the entire browser tab). 

**The Gotcha**: Browsers **do not** immediately free GPU backing store memory when a JavaScript canvas variable goes out of scope or is garbage collected. The GPU buffer is only released when the `<canvas>` element's dimensions are explicitly **shrunk to 0x0** (`canvas.width = 0; canvas.height = 0`). Failing to do this causes Safari to crash the web process with an out-of-memory reload.

---

## 2. Memory Helpers on `fabric.util` (`fabricMemory.js`)

Installed globally on `fabric.util` at application startup (from `src/App/modules/Editor/fabricMemory.js`):

### 1. Temporary Offscreen Operations: `fabric.util.withTempCanvas()`
**Always** use `withTempCanvas` instead of creating raw `<canvas>` elements for temporary operations (e.g. reading pixel data, calculating contours, downsampling images, rasterizing SVG, reading magic wand masks):

```javascript
// Good - canvas backing store is automatically shrunk to 0x0 upon completion or error
const pixelData = fabric.util.withTempCanvas(width, height, (ctx, tempCanvas) => {
  ctx.drawImage(sourceImage, 0, 0, width, height);
  return ctx.getImageData(0, 0, width, height).data;
});

// Also works seamlessly with async / Promises:
const dataUrl = await fabric.util.withTempCanvas(width, height, async (ctx, tempCanvas) => {
  await renderSomethingAsync(ctx);
  return tempCanvas.toDataURL('image/png');
});

// Bad - leaks GPU backing store in Safari memory until GC!
const tempCanvas = document.createElement('canvas');
tempCanvas.width = width;
tempCanvas.height = height;
const ctx = tempCanvas.getContext('2d');
// ... no releaseCanvas() ...
```

### 2. Immediate Canvas Memory Release: `fabric.util.releaseCanvas()`
Forces immediate zeroing of a canvas's backing store:
```javascript
fabric.util.releaseCanvas(canvasElement);
```

### 3. Creating Canvases to Retain: `fabric.util.drawToNewCanvas()`
Creates a new canvas and draws into it. If the `draw` callback throws an error, the canvas is automatically released:
```javascript
const canvas = fabric.util.drawToNewCanvas(width, height, (ctx, c) => {
  ctx.drawImage(source, 0, 0);
});

// When the caller is permanently done with this canvas:
fabric.util.releaseCanvas(canvas);
```

### 4. Disposing Temporary Fabric Objects: `fabric.util.disposeObjects()`
When creating temporary clones or calculation objects, dispose them to free cache canvases:
```javascript
fabric.util.disposeObjects([tempObject1, tempObject2]);
```

---

## 3. Deep Disposal Architecture (`fabricCustom.js`)

`fabricCustom.js` monkey-patches native Fabric disposal methods so memory leaks are prevented automatically:
- `fabric.Canvas.prototype.dispose` & `StaticCanvas.prototype.dispose`: Automatically shrink upper, lower, and cache canvas elements to `0x0`.
- `fabric.Group.prototype.dispose` (and `fabric.Layer`): Automatically recurses and disposes all nested child objects (`_objects`).
- `fabric.Image.prototype.dispose`: Automatically releases its `_filteredEl` canvas and evicts GPU tile textures.
- `fabric.PatternImage.prototype.dispose`: Automatically releases its source copy canvas (`releaseSourceCopy()`).
- `fabric.Object.prototype._removeCacheCanvas`: Automatically zeroes the cache canvas element.

---

## 4. Cleaning Up Objects in React State (`useDisposeWhenReplaced`)

When holding temporary Fabric objects, swatches, or patterns in React component state, use `useDisposeWhenReplaced` from `src/App/modules/Editor/objectUtils.js` to ensure previous instances are disposed upon replacement or component unmount:

```javascript
import { useDisposeWhenReplaced } from 'src/App/modules/Editor/objectUtils';

function SwatchPreview({ previewObject }) {
  useDisposeWhenReplaced(previewObject);
  // previewObject will have dispose() called when replaced or when unmounting
  return <div ... />;
}
```

---

## 5. `fabric.PatternImage` Class

A custom image class (`fabric.util.createClass(fabric.Image)`) implemented in `fabricCustom.js` supporting tiled fill patterns, color replacement, and white ink handling (`applyWhite`).

```javascript
fabric.PatternImage.fromURL(url, (patternImg, isError) => {
  if (!isError && patternImg) {
    // patternImg automatically tracks source element canvas and releases textures on disposal
  }
}, { crossOrigin: 'anonymous' });
```
