# Layer Architecture & Object Insertion

Single source of truth for canvas layer hierarchy, printable vs system layers, and adding objects safely.

---

## 1. Canvas Layer Architecture

In Designer V2, **`canvas._objects` is an array of `fabric.Layer` instances**, not raw drawing objects.

### Layer Tree Representation
```
Canvas (_objects)
 ├── fabric.Layer (id: 'background')
 ├── fabric.Layer (id: 'contents' / custom user layer)  <-- User design objects live here
 ├── fabric.Layer (id: 'white', 'varnish', 'primer')   <-- Special ink layers
 ├── fabric.Layer (layerType: 'cropMark')              <-- System: Crop marks
 ├── fabric.Layer (layerType: 'smart-guide')           <-- System: Snapping guides
 └── fabric.Layer (layerType: 'grid')                  <-- System: Alignment grid
```

---

## 2. Core Rule: Never Use Raw `canvas.add(obj)` for Design Objects

Raw `canvas.add(obj)` pushes the object directly into `canvas._objects`. This bypasses the layer engine entirely, placing the object outside the layer tree and causing:
- Inability to hide or lock the object via the layer panel.
- Broken z-index stacking and rendering behind or above system guides.
- Corrupted export and print output generation.

---

## 3. Adding Objects to Layers

### Primary Insertion Pattern: `canvas.addToContentsLayer`
Always use `canvas.addToContentsLayer`:

```javascript
await canvas.addToContentsLayer({
  obj: fabricObjectOrArray,
  layerId: currentLayerId || 'contents', // Defaults to 'contents'
  designerObjectType: Consts.OBJECT_TYPE_CLIPART, // Domain metadata
  index: targetZIndex, // Optional z-index inside the layer
  shouldEndSvgEditing: true
});
canvas.requestRenderAll();
```

### Direct Layer Insertion
If you already hold a reference to the target `fabric.Layer`:
```javascript
const layer = canvas.getLayerById(layerId);
if (layer) {
  layer.add(obj); // or layer.insertAt(obj, index);
  obj.layer = layer;
  canvas.requestRenderAll();
}
```

---

## 4. Layer Types & Management

### Layer Categories
* **Printable Layers**: User design content layers that are sent to the RIP/printer software (`Utils.isPrintableLayer(layer)`).
* **System / Non-Printable Layers**: Overlays maintained by the editor for alignment and framing:
  - `background` (canvas backdrop)
  - `smart-guide` (dynamic alignment snapping guides)
  - `grid` (measurement grid lines)
  - `cropMark` (print cutting guides)
  - `bleedLayer` (bleed safety margins)
  - `frameGuide` (jig/device frame boundaries)

### Creating New Layers (`LayerUtils`)
Use `LayerUtils.addNewLayer` from `src/App/Utils.js` to create layers with proper clip paths and system index placement:

```javascript
import { LayerUtils } from 'src/App/Utils';

const newLayer = LayerUtils.addNewLayer({
  canvas,
  layerType: 'contents',
  width: canvas.pageWidth,
  height: canvas.pageHeight,
  margins: { ml: 0, mt: 0, mr: 0, mb: 0 }
});
```

`LayerUtils.addNewLayer` automatically inserts the new layer below `smart-guide` and above `background`, preserving critical system layer ordering.
