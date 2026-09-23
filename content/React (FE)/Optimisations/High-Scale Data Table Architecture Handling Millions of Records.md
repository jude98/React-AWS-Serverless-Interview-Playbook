# High-Scale Data Table Architecture Handling Millions of Records

> [!abstract] Architectural Strategy
> 
> A browser cannot hold or render millions of records simultaneously without exhausting memory (OOM crash) and freezing the main thread. Handling millions of rows requires an **end-to-end windowing pipeline**: **cursor-based pagination/streaming at the API layer**, **sparse chunk caching on the client**, **bi-directional DOM virtualization (rows + columns)**, and **CSS layout isolation**.
> 
>   

## Key Concepts

- **Memory vs. DOM Bound**: 1,000,000 JSON rows consumes ~200MB–500MB of raw JS heap; rendering them directly creates millions of DOM nodes, freezing layout engines.
    
      
    
- **Bi-directional Virtualization (Windowing)**: Only render nodes currently inside the viewport bounding box plus a small overscan buffer (e.g., 20–30 visible rows, 8–10 visible columns).
    
      
    
- **Cursor vs. Offset Pagination**:
    
      
    - `OFFSET / LIMIT`: $\mathcal{O}(N)$ database scan penalty at high pages (`OFFSET 1000000` requires scanning and discarding 1M rows).
        
          
        
    - Keyset / Cursor: $\mathcal{O}(1)$ indexed range scan (`WHERE id > :last_seen_id ORDER BY id ASC LIMIT 50`).
        
          
        
- **Sparse Chunk Array Cache**: Model client memory as an index-sparse array (or bucketed Map of chunks like `Map<chunkIndex, Row[]>`). Fetch and evict chunks dynamically as the virtual scroll position moves.
    
      
    
- **Dynamic Row Heights**: Use element measurement observers (`ResizeObserver` via TanStack Virtual) to recalculate row offsets dynamically without breaking scroll physics.
    
      
    
- **CSS Layout Isolation**: Add `contain: strict` or `content-visibility: auto` to prevent single-cell DOM mutations from triggering reflow across the entire table.
    
      
    

## Common Interview Questions

- Why does standard `OFFSET` pagination fail at scale, and how does cursor-based pagination fix it?
    
      
    
- How do you implement infinite scrolling with jump-to-index (random access) when you don't load all previous rows?
    
      
    
- What are the trade-offs between rendering via standard `<table>`, CSS Grid, or absolute positioning in virtualized tables?
    
      
    
- How do you handle column virtualization for tables with hundreds of columns?
    
      
    
- How do you preserve row selection, sorting, and inline editing state when rows are dynamically mounted and unmounted by the virtualizer?
    
      
    
- How does `ResizeObserver` work with virtualized tables having variable-height rows?
    
      
    

## Strong Answers / Talking Points

### 1. The API & Ingestion Architecture

- **Never Load All Data Upfront**:
    
      
    - Expose API endpoints returning fixed-size pages (e.g., 50–100 rows) with cursor tokens (`next_cursor`, `prev_cursor`).
        
          
        
    - For total counts (e.g., "Page 1 of 20,000"), use approximate database counts (PostgreSQL `pg_class.reltuples` or cached Redis counters) instead of running a locking `SELECT COUNT(*)`.
        
          
        
- **Sparse Chunk Fetching (Random-Access Infinite Scroll)**:
    
      
    - If users drag the scrollbar to jump to row 500,000, calculate `targetPage = Math.floor(rowIndex / pageSize)`.
        
          
        
    - Fetch chunk index `[500000...500100]` on-demand. Store chunks in a bounded LRU cache on the client; evict pages farthest from the current viewport to cap browser memory under 50MB.
        
          
        

### 2. Rendering & DOM Virtualization Strategy

- **Absolute Positioning over Native `<table>` Elements**:
    
      
    - Native HTML `<table>` enforces dynamic table-layout calculations where cells negotiate widths, causing reflow cascades on mount/unmount.
        
          
        
    - Use `div`-based virtual layouts with `position: absolute; transform: translateY(${start}px)` for GPU-composited positioning.
        
          
        
- **Bi-directional Virtualization (Rows + Columns)**:
    
      
    - Tables with 50+ columns suffer from horizontal layout thrashing.
        
          
        
    - Virtualize horizontal columns identically to vertical rows using `estimateSize` for column widths.
        
          
        
- **Sticky Elements**:
    
      
    - Implement sticky headers and frozen columns using CSS `position: sticky; z-index: ...` inside the virtual scroll viewport container.
        
          
        

### 3. State Management & Row Selection

- **Decouple Selection from Row DOM State**:
    
      
    - Unmounting a virtualized row destroys its local DOM state (checkbox toggles, inputs).
        
          
        
    - Store selected item keys in a global `Set<string>` or `Map<string, RowData>` outside the render loop.
        
          
        
    - Look up selection status via $\mathcal{O}(1)$ key checks inside the virtual row renderer: `isSelected = selectedSet.has(row.id)`.
        
          
        

## Code Snippets / Examples

### Virtualized Infinite Table with Cursor Fetching

```TypeScript
import React, { useRef, useEffect } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';
import { useInfiniteQuery } from '@tanstack/react-query';

interface TableRow {
  id: string;
  name: string;
  status: string;
  value: number;
}

const fetchRowBatch = async ({ pageParam = 0 }): Promise<{ rows: TableRow[]; nextCursor: number }> => {
  const res = await fetch(`/api/records?cursor=${pageParam}&limit=50`);
  if (!res.ok) throw new Error('Failed to fetch batch');
  return res.json();
};

export const LargeScaleVirtualTable = ({ approximateTotal = 1_000_000 }: { approximateTotal?: number }) => {
  const parentRef = useRef<HTMLDivElement | null>(null);

  // 1. Infinite query manages chunked server state
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
    queryKey: ['table-records'],
    queryFn: fetchRowBatch,
    initialPageParam: 0,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });

  // Flattened active loaded rows
  const allRows = data ? data.pages.flatMap((page) => page.rows) : [];

  // 2. Virtualizer allocates total virtual height based on total items
  const rowVirtualizer = useVirtualizer({
    count: hasNextPage ? allRows.length + 1 : allRows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 40, // 40px fixed row height
    overscan: 10,           // Render 10 rows above/below viewport
  });

  const virtualItems = rowVirtualizer.getVirtualItems();

  // 3. Trigger next page fetch when scrolling near the end
  useEffect(() => {
    const lastItem = virtualItems[virtualItems.length - 1];
    if (!lastItem) return;

    if (lastItem.index >= allRows.length - 1 && hasNextPage && !isFetchingNextPage) {
      fetchNextPage();
    }
  }, [virtualItems, allRows.length, hasNextPage, isFetchingNextPage, fetchNextPage]);

  return (
    <div
      ref={parentRef}
      className="h-[600px] w-full overflow-auto border border-gray-300 relative"
      style={{ contain: 'strict' }} // CSS containment for rendering isolation
    >
      {/* Total scroll height canvas */}
      <div
        className="w-full relative"
        style={{ height: `${rowVirtualizer.getTotalSize()}px` }}
      >
        {virtualItems.map((virtualRow) => {
          const isLoaderRow = virtualRow.index > allRows.length - 1;
          const row = allRows[virtualRow.index];

          return (
            <div
              key={virtualRow.key}
              className="absolute top-0 left-0 w-full flex items-center px-4 border-b border-gray-100 bg-white"
              style={{
                height: `${virtualRow.size}px`,
                transform: `translateY(${virtualRow.start}px)`,
              }}
            >
              {isLoaderRow ? (
                <span className="text-gray-400 text-sm">Loading more records...</span>
              ) : (
                <div className="grid grid-cols-4 w-full text-sm">
                  <span className="font-mono text-xs text-gray-500">{row.id}</span>
                  <span>{row.name}</span>
                  <span>{row.status}</span>
                  <span className="text-right">${row.value.toFixed(2)}</span>
                </div>
              )}
            </div>
          );
        })}
      </div>
    </div>
  );
};
```

## Related Topics

- [[High-Scale Data Table Architecture Handling Millions of Records|Virtualization and Large Data Rendering]]
    
      
    
- [[High-Scale Data Table Architecture Handling Millions of Records|Database Indexing and Cursor Pagination]]
    
      
    
- [[TanStack Query Server State and Stale While Revalidate Patterns]]
    
      
    
- [[The Browser Rendering Pipeline. Reflow, Repaint, and Composite|Browser Layout Thrashing and CSS Containment]]
    
      
    
- [[Browser Workers Architecture. Dedicated, Shared, Service & Worklets|Web Workers and Off-Main-Thread Processing]]
    
      
    

## Tags

#fullstack #interview #data-table #virtualization #large-scale #cursor-pagination #performance

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups