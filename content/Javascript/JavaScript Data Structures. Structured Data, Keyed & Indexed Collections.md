# JavaScript Data Structures: Structured Data, Keyed & Indexed Collections

## Key Concepts

> [!summary] Structured Data (JSON)
> 
> JSON (JavaScript Object Notation) is a lightweight, language-agnostic, text-based data interchange format. It supports only a subset of JavaScript primitives (`string`, `number`, `boolean`, `null`) along with plain `Object` and `Array` structures. It does not support `undefined`, functions, symbols, BigInt, or circular references.
> 
>   

> [!abstract] Keyed Collections: `Map` vs. Plain `Object`
> 
>   
> 
> - `Map`: Direct key-value collection where keys can be **any type** (primitives, objects, functions), insertion order is strictly guaranteed, size is retrieved in $O(1)$ via `.size`, and built-in iterable protocol is supported.
>     
>       
>     
> - Plain `Object`: Keys must be `string` or `symbol`, prototypes introduce default fallback keys, and size must be calculated manually (`Object.keys(obj).length`).
>     
>       
>     

> [!info] Sets: Unique Collections
> 
>   
> 
> - `Set`: Collection of unique values (both primitive and object references). Value equality is evaluated via the `SameValueZero` algorithm (meaning `NaN === NaN`, and `+0 === -0`).
>     
>       
>     

> [!danger] Garbage Collection & Weak Collections (`WeakMap`, `WeakSet`)
> 
>   
> 
> - In `Map` and `Set`, references to keys/values are held **strongly**, preventing garbage collection (GC) even if no other references exist in the application.
>     
>       
>     
> - In `WeakMap` and `WeakSet`, entries hold **weak references** to their object keys/members. If an object has no other strong references, it is eligible for garbage collection, automatically removing the entry.
>     
>       
>     
> - Weak collections are **not iterable**, have no `.size` property, and accept **only objects** (and non-registered symbols) as keys/elements.
>     
>       
>     

> [!tip] Indexed Collections: Standard `Array` vs. `TypedArray`
> 
>   
> 
> - Standard `Array`: Dynamic, resizable, heterogeneous lists (can hold mixed types). Engines optimize them under the hood (e.g., packed vs. holey elements, SMI vs. double elements).
>     
>       
>     
> - `TypedArray` (`Int8Array`, `Uint8Array`, `Float64Array`, etc.): Fixed-length, contiguous raw memory views over an underlying `ArrayBuffer`. Designed for high-performance binary data manipulation (WebGL, Canvas, Web Workers, WebAssembly, file streams).
>     
>       
>     

## Common Interview Questions

- "What is the difference between `Map` and a plain JavaScript object? When would you choose one over the other?"

- "What are `WeakMap` and `WeakSet` used for in real-world applications? Why can't you iterate over them?"

- "What types are dropped or mutated when passing an object through `JSON.stringify()`?"

- "How does a `TypedArray` differ from a standard JavaScript `Array` in terms of memory layout and performance?"

- "How does `Set` evaluate uniqueness? Does it treat `{ id: 1 }` and `{ id: 1 }` as duplicates?"

- "What happens when you try to stringify an object containing circular references or a `BigInt`?"

## Strong Answers / Talking Points

### 1. `Map` vs. Plain `Object`: Decision Matrix

- **Use `Map` when**:

    - Keys are unknown until runtime, or keys are complex types (objects, DOM nodes, functions).

    - You need frequent additions and removals (engines optimize `Map` for frequent mutations).

    - Preserving insertion order is critical for iteration.

    - You need quick access to total entry count via `.size`.

- **Use Plain `Object` when**:

    - You have fixed, known schemas/shapes (allowing V8 hidden classes and inline caching to optimize property access).

    - You need direct JSON serialization (`JSON.stringify` works natively on plain objects, not `Map`).

    - You need simple record-like key-value structures.

### 2. Weak Collections (`WeakMap`, `WeakSet`) & Memory Leaks

- **Core Mechanism**: They do not prevent the Garbage Collector from freeing their keys.

- **Why Non-Iterable**: Because garbage collection is non-deterministic (depends on engine heuristics), exposing iteration, keys, or `.size` would yield non-deterministic results across runs.

- **Primary Use Cases**:

    - **DOM Node Metadata**: Storing listener state, analytics tags, or cache data tied to DOM elements. When the DOM element is removed from the tree, the associated cache entry is cleaned up automatically without explicit teardown code.

    - **Private Data / Encapsulation**: Storing private instance fields in legacy/pre-ES2022 code without exposing properties on the instance.

### 3. JSON Serialization Edge Cases

- Properties with values of `undefined`, `Function`, or `Symbol` are:

    - **Omitted** entirely when found in objects.

    - Converted to `null` when found in arrays.

- `NaN` and `Infinity` are converted to `null`.

- `Date` objects are converted to ISO strings via `Date.prototype.toJSON()`.

- `BigInt` throws a `TypeError: Do not know how to serialize a BigInt`.

- Circular references throw `TypeError: Converting circular structure to JSON`.

### 4. `ArrayBuffer` and `TypedArray` Mechanics

- An `ArrayBuffer` represents a fixed-length raw binary buffer; you cannot directly access or mutate its byte values.

- A `TypedArray` (or `DataView`) is an indexed view over an `ArrayBuffer` allocating specific byte widths (e.g., `Uint8Array` uses 1 byte per slot, `Float64Array` uses 8 bytes per slot).

- Standard arrays can suffer from de-optimizations when transitioning between dense (continuous indices) and sparse (holey) layouts; `TypedArray` guarantees contiguous, unboxed numeric memory allocations.

## Code Snippets / Examples

### `Map` vs. `WeakMap` Garbage Collection Pattern

```javascript
// Strong reference holding via Map
let user = { id: 101, name: "Alice" };
const userMap = new Map();
userMap.set(user, "User Metadata");

// Even if user reference is cleared, userMap keeps it alive in memory
user = null; // Object is NOT garbage collected!

// Preventing memory leaks with WeakMap
let domElement = document.querySelector("#submit-button");
const clickTracker = new WeakMap();

clickTracker.set(domElement, { clickCount: 0 });

// Later, when domElement is removed from DOM:
domElement.remove();
domElement = null;
// The { clickCount: 0 } entry in WeakMap is eligible for Garbage Collection
```

### JSON Serialization Quirks & Replacer Function

```javascript
const payload = {
  id: 1,
  tag: Symbol("admin"),
  metadata: undefined,
  score: NaN,
  createdAt: new Date(),
  calculate: () => 42,
  items: [undefined, () => {}, 10]
};

console.log(JSON.stringify(payload));
// {"id":1,"score":null,"createdAt":"2026-09-20T...","items":[null,null,10]}
// Notice: tag, metadata, and calculate are dropped; array elements become null

// Handling BigInt via replacer argument
const dataWithBigInt = { id: 1n, name: "Metric" };
const jsonString = JSON.stringify(dataWithBigInt, (key, value) =>
  typeof value === "bigint" ? value.toString() : value
);
console.log(jsonString); // {"id":"1","name":"Metric"}
```

### Memory Buffers and TypedArrays

```javascript
// Allocate 16 bytes of contiguous memory
const buffer = new ArrayBuffer(16);

// Create two distinct views over the same raw buffer
const int32View = new Int32Array(buffer); // 4 bytes per element (length: 4)
const uint8View = new Uint8Array(buffer); // 1 byte per element (length: 16)

int32View[0] = 42;

// The change is visible in the raw byte view
console.log(uint8View[0]); // 42
console.log(int32View.byteLength); // 16
```

## Comparison Matrix: Keyed Collections

|**Collection**|**Key Types Allowed**|**Strong/Weak Keys**|**Iteration Support**|**Has .size**|**Primary Use Case**|
|---|---|---|---|---|---|
|**`Map`**|Any type (primitives + objects)|Strong|Yes (`for...of`, `.entries()`)|Yes|Dynamic dictionary, fast insertions/deletions|
|**`WeakMap`**|Objects & non-registered Symbols|Weak (keys only)|No|No|Metadata on DOM nodes, cache memory leak prevention|
|**`Set`**|Any type (unique values)|Strong|Yes (`for...of`, `.values()`)|Yes|De-duplication, membership testing ($O(1)$)|
|**`WeakSet`**|Objects & non-registered Symbols|Weak|No|No|Object tagging, instance tracking without preventing GC|

## Related Topics

- [[JavaScript Data Types, Objects & Prototypal Inheritance]]

- [[JavaScript Garbage Collection. Reachability, Mark-and-Sweep & Generational Memory|Memory Management and Garbage Collection in V8]]

- [[JavaScript Loops, Iteration Protocols & Data Structure Traversal|Iterables, Iterators, and Generators]]

- [[Client-Side Browser Storage. Mechanisms, Architecture & Security|Streams, Buffers, and Binary Data in Node.js]]

## Tags

#fullstack #interview #javascript #data-structures #map #set #typedarray #json

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups