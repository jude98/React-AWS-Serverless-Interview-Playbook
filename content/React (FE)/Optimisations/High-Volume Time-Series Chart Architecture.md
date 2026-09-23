# High-Volume Time-Series Chart Architecture

> [!abstract] Architectural Overview
> 
> Rendering 3 days of minute-by-minute data equals **4,320 points per series**. With multiple metrics or zooming, rendering raw SVG DOM nodes causes severe layout thrashing and high INP. The solution requires a **Canvas/WebGL rendering core**, a **hybrid DOM overlay for interactions**, **data downsampling (LTTB/Bucketing)**, and an **OffscreenCanvas / Web Worker pipeline** to keep the React main thread unblocked.
> 
>   

## Key Concepts

- **Scale Math**: $3 \text{ days} \times 24 \text{ hrs} \times 60 \text{ mins} = 4,320 \text{ points}$ per metric. If a screen width is 1440px, there are ~3 data points per horizontal screen pixel, making raw point-rendering visually redundant.
    
      
    
- **Rendering Engine Trade-off**:
    
      
    - **SVG (Recharts, standard D3)**: Retains a DOM node per point/bar. At >1,000 nodes, memory balloons and hover interactions cause layout recalculations.
        
          
        
    - **HTML5 Canvas / WebGL (uPlot, ECharts, PixiJS)**: Single DOM element; draws pixels directly on an immediate-mode bitmap. Renders tens of thousands of points at 60 FPS.
        
          
        
- **Data Downsampling Algorithms**:
    
      
    - **Min-Max / OHLC Bucketing**: Groups points into time buckets (e.g., 15-minute chunks) and extracts Minimum, Maximum, Open, and Close to preserve statistical peaks.
        
          
        
    - **LTTB (Largest Triangle Three Buckets)**: Downsamples time-series data while visually preserving peaks, valleys, and perceived shape far better than basic averaging.
        
          
        
- **Hybrid Component Model**:
    
      
    - **Visual Surface**: Rendered on `<canvas>` for high throughput.
        
          
        
    - **Interactive Surface**: HTML/SVG overlays for tooltips, crosshairs, and legends to preserve accessibility and styling flexibility.
        
          
        
- **OffscreenCanvas & Web Workers**: Offload downsampling math, date formatting, and canvas drawing routines to a background worker thread.
    
      
    

## Common Interview Questions

- Why do SVG-based chart libraries degrade in performance with thousands of points, and how does Canvas or WebGL solve this?
    
      
    
- How do you handle rendering 3 days of minute-level data when the screen only has 1,200 horizontal pixels?
    
      
    
- What is the difference between simple rolling-average downsampling vs. Min-Max or LTTB (Largest Triangle Three Buckets)?
    
      
    
- How do you design an accessible and responsive tooltip system on top of an immediate-mode HTML5 Canvas?
    
      
    
- How would you structure zooming and panning so that the browser does not re-fetch or re-calculate the entire 3-day dataset on every frame?
    
      
    
- What role does `OffscreenCanvas` play in preventing long interaction tasks (INP) during high-frequency data streams?
    
      
    

## Strong Answers / Talking Points

### 1. Component Decomposition (Compound Pattern)

Break the chart into isolated, single-responsibility layers instead of one massive monolithic component:

  

1. **ChartProvider (Context Engine)**:
    
      
    - Holds shared dimensions, coordinate scales (d3-scale `scaleTime`, `scaleLinear`), active bounding box, and zoom/pan viewport domains.
        
          
        
2. **DataCanvas (Visual Base Layer)**:
    
      
    - Contains the raw `<canvas>` element.
        
          
        
    - Listens to data changes and viewport domain changes.
        
          
        
    - Pure drawing operations; never renders HTML children.
        
          
        
3. **Axis & Grid Layer**:
    
      
    - Computes tick values using d3-scale generators.
        
          
        
    - Rendered either directly onto the canvas or via a lightweight SVG overlay for sharp text rendering.
        
          
        
4. **Interaction & Crosshair Layer**:
    
      
    - An invisible HTML layer covering the canvas to capture `onPointerMove`, `onWheel`, and touch gestures.
        
          
        
    - Computes data index via binary search (`d3-bisector`) in $\mathcal{O}(\log N)$ based on pointer X coordinate.
        
          
        
5. **Tooltip & Legend (HTML/React Layer)**:
    
      
    - Completely decoupled from the canvas render loop.
        
          
        
    - Updates using CSS `transform: translate3d()` to avoid triggering browser reflows.
        
          
        

### 2. Handling 3 Days of Minute-by-Minute Data (4,320 Points)

- **Pixel-Matching Downsampling**:
    
      
    - A viewport is typically 1,000px to 1,920px wide. Displaying 4,320 SVG points wastes memory because multiple points occupy the same sub-pixel column.
        
          
        
    - Group data into dynamic buckets matching the client's physical pixel width: $\text{Bucket Size} = \lceil \frac{\text{Total Data Points}}{\text{Chart Pixel Width}} \rceil$.
        
          
        
- **Why Simple Averages Fail (and What to Use Instead)**:
    
      
    - _Simple Average_: Dampens spikes, hiding critical telemetry anomalies (e.g., a CPU spike to 99% that lasted 1 minute gets averaged out).
        
          
        
    - _Min-Max / Range Grouping_: Stores the minimum and maximum of each bucket, drawing a vertical range bar or envelope line to highlight true outliers.
        
          
        
    - _LTTB (Largest Triangle Three Buckets)_: Calculates triangles between adjacent buckets to pick the point maximizing triangular area, retaining visual fidelity.
        
          
        
- **Hierarchical Tiering (Zoom-Driven Level of Detail)**:
    
      
    - **Zoomed Out (Full 3 Days)**: Display pre-aggregated 15-minute or hourly buckets from the API or worker.
        
          
        
    - **Zoomed In (e.g., 2-hour window)**: Slice the local array to reveal the raw 1-minute data points without downsampling.
        
          
        

### 3. State & Event Performance Guardrails

- **Separate Fast State from React State**:
    
      
    - Never put crosshair mouse coordinates `(x, y)` in React root state (`useState`), as this re-renders the entire component tree at 60Hz.
        
          
        
    - Use raw DOM manipulation (`ref.style.transform = ...`) or mutate a `useRef` and draw on a dedicated interaction overlay canvas.
        
          
        
- **Non-Blocking Processing**:
    
      
    - If calculations exceed 16ms, run downsampling inside a Web Worker. Use `OffscreenCanvas` with `transferControlToOffscreen()` so render draws execute outside the main JS thread entirely.
        
          
        

## Code Snippets / Examples

### Min-Max Outlier Preserving Downsampler



```TypeScript
interface DataPoint {
  timestamp: number; // Epoch ms
  value: number;
}

/**
 * Downsamples data into discrete pixel-width buckets while preserving peaks and valleys.
 */
export function bucketMinMaxDownsample(
  data: DataPoint[],
  targetPoints: number
): DataPoint[] {
  if (data.length <= targetPoints || targetPoints <= 0) return data;

  const bucketSize = data.length / targetPoints;
  const downsampled: DataPoint[] = [];

  for (let i = 0; i < targetPoints; i++) {
    const startIdx = Math.floor(i * bucketSize);
    const endIdx = Math.floor((i + 1) * bucketSize);

    let minPoint = data[startIdx];
    let maxPoint = data[startIdx];

    for (let j = startIdx + 1; j < endIdx && j < data.length; j++) {
      const current = data[j];
      if (current.value < minPoint.value) minPoint = current;
      if (current.value > maxPoint.value) maxPoint = current;
    }

    // Preserve temporal order of extremes within bucket
    if (minPoint.timestamp <= maxPoint.timestamp) {
      downsampled.push(minPoint, maxPoint);
    } else {
      downsampled.push(maxPoint, minPoint);
    }
  }

  return downsampled;
}
```

### Decoupled Canvas & Fast Crosshair Component


```TypeScript
import React, { useRef, useEffect, useCallback } from 'react';

interface ChartProps {
  data: DataPoint[];
  width: number;
  height: number;
}

export const HighVolumeChart: React.FC<ChartProps> = ({ data, width, height }) => {
  const canvasRef = useRef<HTMLCanvasElement | null>(null);
  const crosshairRef = useRef<HTMLDivElement | null>(null);

  // 1. Heavy Render Path: Canvas Bitmap Drawing (Runs on data/size changes only)
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    // Handle high-DPI displays (Retina)
    const dpr = window.devicePixelRatio || 1;
    canvas.width = width * dpr;
    canvas.height = height * dpr;
    ctx.scale(dpr, dpr);

    ctx.clearRect(0, 0, width, height);
    ctx.strokeStyle = '#2563eb';
    ctx.lineWidth = 1.5;

    // Simple uniform scale calculation
    const minVal = Math.min(...data.map((d) => d.value));
    const maxVal = Math.max(...data.map((d) => d.value)) || 1;
    const stepX = width / (data.length - 1);

    ctx.beginPath();
    data.forEach((pt, i) => {
      const x = i * stepX;
      const y = height - ((pt.value - minVal) / (maxVal - minVal)) * height;
      if (i === 0) ctx.moveTo(x, y);
      else ctx.lineTo(x, y);
    });
    ctx.stroke();
  }, [data, width, height]);

  // 2. Fast Interaction Path: Bypass React state, mutate transform directly (60fps)
  const handlePointerMove = useCallback((e: React.PointerEvent<HTMLDivElement>) => {
    const rect = e.currentTarget.getBoundingClientRect();
    const x = e.clientX - rect.left;

    if (crosshairRef.current) {
      crosshairRef.current.style.transform = `translate3d(${x}px, 0px, 0)`;
      crosshairRef.current.style.opacity = '1';
    }
  }, []);

  const handlePointerLeave = useCallback(() => {
    if (crosshairRef.current) {
      crosshairRef.current.style.opacity = '0';
    }
  }, []);

  return (
    <div
      className="relative overflow-hidden cursor-crosshair"
      style={{ width, height }}
      onPointerMove={handlePointerMove}
      onPointerLeave={handlePointerLeave}
    >
      {/* Base Canvas */}
      <canvas ref={canvasRef} style={{ width, height }} className="absolute inset-0" />

      {/* GPU-Accelerated Hardware Transformed Crosshair */}
      <div
        ref={crosshairRef}
        className="absolute top-0 left-0 bottom-0 w-[1px] bg-red-500 pointer-events-none opacity-0 will-change-transform"
      />
    </div>
  );
};
```

## Related Topics

- [[Browser Architecture. High-Level Components, Rendering Engines & HTML Parsing|Browser Rendering Engine and Critical Rendering Path]]
    
      
    
- [[Browser Workers Architecture. Dedicated, Shared, Service & Worklets|Web Workers and Off-Main-Thread Processing]]
    
      
    
- [[High-Scale Data Table Architecture Handling Millions of Records|Virtualization and Large Data Rendering]]
    
      
    
- [[Comprehensive Performance Optimization Architecture in React|React Performance Optimization]]
    
      
    
- [[Web Vitals Optimization LCP INP and FCP]]
    
      
    

## Tags

#fullstack #interview #data-visualization #charts #performance #canvas #webgl

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups