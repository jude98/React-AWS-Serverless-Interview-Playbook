# The Browser Rendering Pipeline: Reflow, Repaint, and Composite

## Key Concepts

> [!summary] The Core Rendering Pipeline
> Once the browser constructs the **DOM** (from HTML) and the **CSSOM** (from CSS), it combines them to convert code into physical display pixels through five sequential stages:
> 
> $$\text{DOM + CSSOM} \longrightarrow \text{Render Tree} \longrightarrow \text{Layout (Reflow)} \longrightarrow \text{Paint (Repaint)} \longrightarrow \text{Composite}$$
> 
> 

> [!abstract] Pipeline Stage Definitions
> 1. **Render Tree**: A tree of visual elements combining DOM structure with calculated CSS styles. Omits non-visual nodes (`<head>`, `<script>`, `display: none`), but retains elements with `visibility: hidden` (since they occupy physical space).
> 2. **Reflow (Layout)**: The browser calculates the exact geometric position, bounding box coordinates, and dimensions for every visible element on the page relative to the viewport.
> 3. **Paint (Repaint)**: The browser fills in visual pixels (colors, borders, shadows, backgrounds, text rasterization) into drawing command bitmaps (layers).
> 4. **Composite**: The browser groups painted surfaces into GPU memory textures (**compositing layers**) and composites them onto the screen using the **Compositor Thread** and GPU.
> 
> 

> [!danger] Reflow vs. Repaint vs. Composite Triggers
> * **Reflow always triggers Repaint and Composite**: Changing an element's geometry (`width`, `height`, `margin`, `top`, `left`, `fontSize`) forces recalculation of surrounding elements, followed by repainting and recompositing.
> * **Repaint triggers Composite (skips Reflow)**: Changing visual appearance without altering geometry (`color`, `background-color`, `box-shadow`, `visibility`) skips layout recalculation but re-rasterizes the pixels.
> * **Composite-Only (skips Reflow AND Repaint)**: Changing properties handled directly by the GPU (`transform`, `opacity`, `filter`) bypasses both Layout and Paint completely.
> 
> 

---

## Common Interview Questions

* "Walk me through the pipeline from DOM/CSSOM creation to pixels on screen."
* "What is the concrete difference between Reflow and Repaint?"
* "Why is animating `transform: translate()` significantly better for performance than animating `top` / `left` or `width`?"
* "What is Layout Thrashing (Forced Synchronous Layout), and how does it cause frame drops?"
* "What are compositing layers, and how does `will-change` affect GPU acceleration?"
* "Can you list CSS properties that trigger Reflow, properties that trigger Repaint, and properties that trigger only Compositing?"

---

## Deep Dive & Talking Points

### 1. Reflow (Layout) Mechanics & Ripple Effects

* Reflow is the most computationally expensive phase in the pipeline because elements exist within a **flow-based layout model**.
* Changing the dimensions of a single parent element or changing font size can alter line wrapping, causing a cascade that invalidates and forces the recalculation of geometry for child, sibling, and ancestor nodes throughout the document tree.
* Common triggers:
* Resizing the browser window or rotating a mobile device.
* Adding, deleting, or reordering DOM nodes.
* Changing geometric CSS properties: `width`, `height`, `padding`, `margin`, `border-width`, `font-size`, `display`, `position`, `top`, `left`.
* Querying layout metrics via JavaScript (reading properties like `offsetWidth`, `clientHeight`, `scrollTop`, or calling `getBoundingClientRect()`).

### 2. Repaint (Rasterization) Mechanics

* Once the browser knows the exact coordinates and dimensions from the Reflow stage, it records drawing operations (e.g., "draw a black rectangle at $(x, y)$", "render text glyphs with font X").
* It rasterizes vectors and text into raw pixel bitmaps.
* Triggered whenever an element's appearance changes without changing its size, shape, or position on the page (e.g., `background-color`, `color`, `border-style`, `box-shadow`, `outline`).
* Less expensive than Reflow, but still executed on the **CPU main thread**, competing with JavaScript execution.

### 3. Why `transform` is Better than `width` / `left` (Compositing on the GPU)

When animating an element moving across the screen:

#### Case A: Animating `left` or `width` (Slow Path - Main Thread Bound)

```css
/* Triggers: Reflow -> Repaint -> Composite on EVERY single frame (60Hz = every 16.6ms) */
.box {
  transition: left 0.3s;
  left: 200px;
}
```

1. **CPU Main Thread Overload**: On every frame, the JavaScript thread must pause, recalculate the bounding box coordinates (Reflow), and re-render the pixels of that element and the background pixels it exposed (Repaint).
2. **Main Thread Contention**: If any JavaScript task runs for longer than 16ms (or 8.3ms on 120Hz displays), the frame drops, causing visible jank and stutter.

#### Case B: Animating `transform: translateX()` (Fast Path - GPU Accelerated)

```css
/* Triggers: Composite ONLY (Offloaded to GPU) */
.box {
  transition: transform 0.3s;
  transform: translateX(200px);
}
```

1. **Dedicated Compositing Layer**: The element is promoted to its own separate layer texture in GPU VRAM (equivalent to a separate transparent sheet).
2. **Zero Layout, Zero Repaint**: The bitmap image of the element is painted **once** during layer initialization. During the animation, neither its geometry nor its rasterized bitmap pixels change.
3. **Off-Main-Thread Execution**: The **Compositor Thread** instructs the GPU hardware to directly transform and reposition the existing texture layer in 3D/2D space.
4. **Jank-Free Animations**: Even if the main JavaScript thread is completely blocked by heavy computations or synchronous scripts, the compositor thread continues animating the layer smoothly at native display refresh rates (60–120fps).

### 4. Layout Thrashing (Forced Synchronous Layout)

* Normally, browsers **batch** DOM style mutations and defer calculating reflow until the end of the current microtask/frame tick.
* **Layout Thrashing** occurs when code interleaves **style writes** with **style reads** inside a loop.
* Reading an offset property (`el.offsetWidth`) forces the browser to prematurely execute a synchronous Reflow right in the middle of JavaScript execution to return an up-to-date coordinate value.

---

## Code Snippets / Examples

### 1. The Layout Thrashing Anti-Pattern & The Batching Solution

```javascript
// ❌ ANTI-PATTERN: Forced Synchronous Layout (Layout Thrashing)
// Reads and writes alternate inside a loop, forcing N synchronous reflows!
function badResize(elements) {
  for (let i = 0; i < elements.length; i++) {
    // READ: Browser is forced to flush pending styles and calculate reflow NOW!
    const width = elements[i].offsetWidth;

    // WRITE: Invalidates the layout again immediately
    elements[i].style.width = (width + 10) + "px";
  }
}

// ✅ FIX: Batch all reads first, then batch all writes
function goodResize(elements) {
  // Step 1: Read all metrics simultaneously (Single reflow calculation)
  const newWidths = elements.map(el => el.offsetWidth + 10);

  // Step 2: Write all style mutations (Browser batches them for the next frame)
  elements.forEach((el, index) => {
    el.style.width = newWidths[index] + "px";
  });
}
```

---

### 2. Promoting Elements with `will-change` (Use Sparingly)

```css
/* Tells the browser engine to promote the element to its own GPU layer in advance */
.modal-overlay {
  will-change: transform, opacity;
}

/*
⚠️ WARNING ON will-change:
- Do NOT apply will-change to hundreds of elements simultaneously!
- Each layer consumes dedicated GPU VRAM (video memory).
- Excessive layer creation causes memory thrashing, device battery drain,
  and can crash mobile browser tabs.
- Always remove will-change when animations or interactions finish.
*/
```

---

### 3. Comparing CSS Property Pipelines

```css
/* 1. REFLOW PIPELINE (Most Expensive) */
/* Recalculates geometry -> Repaints pixels -> Composites layers */
.reflow-example {
  width: 300px;
  height: 200px;
  margin-top: 20px;
  font-size: 1.2rem;
}

/* 2. REPAINT PIPELINE (Medium Cost) */
/* Skips geometry calculation -> Repaints pixels -> Composites layers */
.repaint-example {
  color: #333333;
  background-color: #f4f4f4;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

/* 3. COMPOSITE-ONLY PIPELINE (Cheapest / Peak Performance) */
/* Skips Reflow AND Repaint -> GPU translates pre-painted bitmap textures */
.composite-example {
  transform: translate3d(50px, 0, 0) scale(1.1);
  opacity: 0.9;
}
```

---

## Comparison Matrix: Reflow vs. Repaint vs. Composite

| Characteristic | Reflow (Layout) | Repaint (Paint) | Composite |
| --- | --- | --- | --- |
| **Execution Thread** | **Main CPU Thread** | **Main CPU Thread** | **Compositor Thread + GPU** |
| **What It Calculates** | Positions, sizes, geometry of nodes | Colors, fills, borders, text pixels | Layer stacking, alpha blending, 2D/3D matrix transforms |
| **Relative Cost** | **Highest (Heavy)** | Medium | **Lowest (Ultra-fast)** |
| **Triggers Downstream Stages?** | Yes $\to$ triggers Repaint + Composite | Yes $\to$ triggers Composite | **No** (runs standalone on GPU) |
| **Typical CSS Properties** | `width`, `height`, `padding`, `margin`, `top`, `left`, `fontSize`, `display` | `color`, `background`, `box-shadow`, `border-color`, `visibility` | `transform`, `opacity`, `filter` |
| **JS Read Triggers** | `offsetWidth`, `clientHeight`, `scrollTop`, `getBoundingClientRect()` | None directly | None directly |

---

## Related Topics

* [[What Happens When You Enter a URL in the Browser. The End-to-End Lifecycle|What Happens When You Enter a URL in the Browser: The End-to-End Lifecycle]]
* [[Browser Architecture. High-Level Components, Rendering Engines & HTML Parsing|Browser Architecture: High-Level Components, Rendering Engines & HTML Parsing]]
* [[V8 Engine Architecture. Parsing, JIT Compilation & Execution Pipeline|V8 Engine Architecture: Parsing, JIT Compilation & Execution Pipeline]]
* [[DOM Event Propagation. Bubbling, Capturing & Event Delegation|DOM Event Propagation: Bubbling, Capturing & Event Delegation]]

---

## Tags

#fullstack #interview #browser-internals #rendering-pipeline #reflow #repaint #compositing #gpu-acceleration #layout-thrashing #web-performance

---

## Revision Checklist

* [ ] Can explain in 60 seconds
* [ ] Can explain trade-offs
* [ ] Can give a real project example
* [ ] Can answer common follow-ups