# JavaScript Garbage Collection: Reachability, Mark-and-Sweep & Generational Memory

## Key Concepts

> [!summary] What is Garbage Collection (GC)?
> 
> JavaScript manages memory automatically. Developers do not manually allocate or free heap space via operations like `malloc()` or `free()`. The **Garbage Collector** continuously monitors object allocations in heap memory, identifies objects that are no longer accessible to the program, and reclaims their memory.
> 
>   

> [!abstract] The Reachability Principle
> 
> Memory reclamation in modern JavaScript engines is rooted in the concept of **Reachability**, not simple reference counting:
> 
>   
> 
> - **Roots**: The foundational set of inherently reachable values. These include global variables (`window`, `globalThis`), active call stack frames (local variables and parameters of currently executing functions), and active closures.
>     
>       
>     
> - **Reachable Object**: Any object that is either a Root or can be accessed through a chain of references starting from a Root.
>     
>       
>     
> - **Unreachable Object**: An object that cannot be reached from any Root, even if it has references pointing to other unreachable objects. Unreachable objects are marked for reclamation.
>     
>       
>     

> [!info] The Mark-and-Sweep Algorithm
> 
> The primary algorithm powering modern garbage collection:
> 
>   
> 
> 1. **Mark Phase**: The GC traverses the entire object graph starting from the Roots, marking every encountered object as "active/alive."
>     
>       
>     
> 2. **Sweep Phase**: The GC traverses the heap memory sequentially. Any object not marked as alive is reclaimed, and its memory space is added to the free-memory pool.
>     
>       
>     
> 3. **Compact Phase (Mark-Compact)**: To eliminate memory fragmentation, surviving objects are relocated into contiguous blocks of memory, and references to them are updated.
>     
>       
>     

> [!tip] Generational Hypothesis & V8 Memory Spaces
> 
> V8 divides the heap into two main generations based on the observation that **most objects die young**:
> 
>   
> 
> - **Young Generation (New Space)**: Where new objects are allocated. Small (usually 1–64 MB) and split into two semi-spaces. Managed by a fast, frequent stop-and-copy collector called **Scavenge**.
>     
>       
>     
> - **Old Generation (Old Space)**: Objects that survive multiple Scavenge cycles are promoted to Old Space. Managed by the heavier **Major GC (Mark-Sweep-Compact)**.
>     
>       
>     

## Common Interview Questions

- "How does Garbage Collection work in modern JavaScript engines?"
    
      
    
- "What is the difference between the Reference Counting algorithm and the Mark-and-Sweep algorithm?"
    
      
    
- "What is a circular reference, and why did it cause memory leaks in older browsers?"
    
      
    
- "What defines a 'Root' in JavaScript memory management?"
    
      
    
- "What is the Generational Hypothesis in V8, and how does Scavenger differ from Major GC?"
    
      
    
- "Can an object have multiple incoming references and still be garbage collected?"
    
      
    
- "How do `WeakMap` and `WeakSet` relate to garbage collection and memory leak prevention?"
    
      
    

## Strong Answers / Talking Points

### 1. Reachability vs. Reference Counting (The Circular Reference Trap)

- **Reference Counting (Legacy / Obsolete)**:
    
      
    - Tracked how many pointers referenced an object. If `count === 0`, free the memory.
        
          
        
    - **Fatal Flaw**: If Object A references Object B, and Object B references Object A (`a.b = b; b.a = a;`), their reference counts remain at least `1`. Even if both objects are disconnected from the application, reference counting could never collect them, creating permanent memory leaks (infamous in early Internet Explorer versions).
        
          
        
- **Mark-and-Sweep (Modern Standard)**:
    
      
    - Completely immune to circular references.
        
          
        
    - The algorithm checks: _Can this island of objects be reached from the Roots?_ If the cycle is disconnected from the root set, the mark phase never visits them, and the sweep phase reclaims the entire disconnected cycle.
        
          
        

### 2. Generational Collection in V8 (New Space vs. Old Space)

- **The Generational Hypothesis**: Empirically, the vast majority of allocations (temporary variables, loop buffers, short-lived promises) become unreachable almost immediately.
    
      
    
- **Young Generation (Scavenge Collection)**:
    
      
    - Uses **Cheney’s copying algorithm** across two semi-spaces: `From-Space` and `To-Space`.
        
          
        
    - When `From-Space` fills, the collector traverses reachable objects, copies them contiguously to `To-Space`, and clears `From-Space` in one operation.
        
          
        
    - Very fast (milliseconds) because it only visits surviving objects, ignoring the dead ones.
        
          
        
- **Old Generation (Mark-Sweep-Compact)**:
    
      
    - If an object survives consecutive Scavenge rounds, it is promoted ("tenured") to Old Space.
        
          
        
    - Scanned less frequently because it holds long-lived state (singletons, global configurations, module caches).
        
          
        

### 3. Mitigating GC Pauses: Orinoco (Concurrent & Incremental GC)

- Historically, GC was a **Stop-the-World** event: JavaScript execution halted completely while the heap was swept, causing noticeable UI jank or dropped frames.
    
      
    
- Modern V8 (project **Orinoco**) introduced:
    
      
    - **Incremental Marking**: Breaks marking down into tiny chunks interleaved with JavaScript execution.
        
          
        
    - **Concurrent Marking/Sweeping**: Worker threads scan and sweep memory in the background without halting the main JavaScript thread.
        
          
        
    - **Parallel Collection**: Multiple threads work together during unavoidable pause phases to minimize stop-the-world time.
        
          
        

### 4. Common Sources of Accidental Retention (Memory Leaks)

- **Forgotten Global Variables**: Unintentional assignment to undeclared variables (`name = 'val'`) anchors objects directly to `window`/`globalThis`.
    
      
    
- **Dangling Event Listeners**: Elements removed from the DOM while an active listener retains a closure pointing to surrounding component state.
    
      
    
- **Forgotten Timers**: `setInterval` callbacks keep all closed-over scope variables alive until `clearInterval` is called.
    
      
    
- **Unbounded Caching**: Using standard `Map` or plain objects as caches without size limits or TTL expiration instead of `WeakMap` or an LRU cache.
    
      
    

## Code Snippets / Examples

### 1. Circular References Reclaimed Under Mark-and-Sweep

```javascript
function allocateFamily() {
  const mother = {};
  const father = {};

  // Circular link between objects
  mother.husband = father;
  father.wife = mother;

  return "done";
}

// When allocateFamily() returns:
// - Call stack frame is popped.
// - Neither 'mother' nor 'father' is reachable from Global Roots.
// - Even though mother and father reference each other, Mark-and-Sweep
//   identifies the entire island as UNREACHABLE and frees both.
allocateFamily();
```

### 2. Unreachable Islands vs. Root-Connected Memory

```javascript
let user = { name: "Alice" }; // 'user' is a Root reference on the global scope
let admin = user;             // Second reference to the same heap object

user = null;  // Object is STILL reachable via 'admin' -> NOT collected
admin = null; // Object is now completely UNREACHABLE from Roots -> Eligible for GC
```

### 3. Preventing Memory Leaks with Weak References

```javascript
// BAD: Strong reference map prevents Garbage Collection
const strongMetadataCache = new Map();

function trackElementBad(domNode) {
  strongMetadataCache.set(domNode, { clicked: 0 });
}

// GOOD: WeakMap holds keys weakly
const weakMetadataCache = new WeakMap();

function trackElementGood(domNode) {
  // Key must be an object. Does NOT prevent GC if domNode is removed from DOM
  weakMetadataCache.set(domNode, { clicked: 0 });
}

let button = document.createElement("button");
trackElementGood(button);

// Later in lifecycle:
button.remove();
button = null; // The associated cache object in weakMetadataCache is automatically collected!
```

### 4. Diagnosing Memory Leaks in Node.js

```javascript
// Exposing GC in Node for memory profiling (run with: node --expose-gc script.js)
if (globalThis.gc) {
  console.log("Memory before:", process.memoryUsage().heapUsed / 1024 / 1024, "MB");
  
  let heavyBuffer = new Array(1e6).fill("leak");
  
  heavyBuffer = null; // Nullify reference to make memory unreachable
  
  globalThis.gc(); // Force full garbage collection pass
  console.log("Memory after GC:", process.memoryUsage().heapUsed / 1024 / 1024, "MB");
}
```

## Comparison Matrix: GC Approaches & Spaces

|**Mechanism / Area**|**Target Scope**|**Algorithm Used**|**Collection Frequency**|**Pause Overhead**|
|---|---|---|---|---|
|**Scavenger (New Space)**|Young, short-lived objects|Semi-space copying (Cheney's)|High (Continuous)|Ultra-low (~1–2ms)|
|**Major GC (Old Space)**|Tenured, long-lived objects|Mark-Sweep-Compact|Low (On-demand)|Low to moderate (Incremental/Concurrent)|
|**Reference Counting (Legacy)**|Entire heap|Pointer tally|Immediate upon unlinking|Zero (but misses circular references)|
|**Mark-and-Sweep (Modern)**|Entire reachable graph|Graph reachability traversal|Periodic / Threshold-based|Mitigated by V8 Orinoco background threads|

## Related Topics

- [[V8 Engine Architecture. Parsing, JIT Compilation & Execution Pipeline|V8 Engine Architecture: Parsing, JIT Compilation & Execution Pipeline]]
    
      
    
- [[JavaScript Closures. Encapsulation, Currying & Output Puzzles|JavaScript Closures: Encapsulation, Currying & Output Puzzles]]
    
      
    
- [[DOM Event Listeners, Browser Memory Management & Teardown Mechanics]]
    
      
    
- [[JavaScript Data Structures. Structured Data, Keyed & Indexed Collections|JavaScript Data Structures: Structured Data, Keyed & Indexed Collections]]
    
      
    

## Tags

#fullstack #interview #javascript #v8 #garbage-collection #mark-and-sweep #memory-leaks #memory-management

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups