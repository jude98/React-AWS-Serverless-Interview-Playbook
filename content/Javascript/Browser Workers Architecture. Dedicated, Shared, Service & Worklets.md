# Browser Workers Architecture: Dedicated, Shared, Service & Worklets

## Key Concepts

> [!summary] What are Web Workers?
> Modern browsers execute JavaScript on a single **main thread** responsible for executing script, evaluating user events, and running layout, styling, and paint pipelines. **Workers** are background execution contexts that run in parallel on separate operating system threads, enabling true multi-threaded JavaScript without blocking the UI.

> [!abstract] Worker Constraints & Environment Capabilities
> * **No DOM Access**: Workers run in an isolated execution context (`WorkerGlobalScope`) and have **zero access** to `window`, `document`, DOM nodes, or local storage.
> * **Available APIs**: Can access `fetch`, `XMLHttpRequest`, `IndexedDB`, `WebSockets`, `crypto`, `location`, `navigator`, and `setTimeout`/`setInterval`.
> * **Communication**: Primarily communicate with the main thread via message passing (`postMessage` and `onmessage`), serializing data through the **Structured Clone Algorithm** or zero-copy **Transferable Objects**.
> 
> 

> [!info] The 4 Main Types of Workers
> 1. **Dedicated Web Workers (`new Worker()`)**: Linked to a single script/tab instance. Used to offload CPU-heavy computations (image processing, big data parsing, cryptography, complex math).
> 2. **Shared Workers (`new SharedWorker()`)**: Shared across multiple browser contexts (tabs, windows, iframes) of the same origin. Managed via explicit `MessagePort` connections.
> 3. **Service Workers (`navigator.serviceWorker.register()`)**: Event-driven network proxies sitting between the browser, network, and disk cache. Powers PWAs, offline caching, background sync, and push notifications.
> 4. **Worklets (AudioWorklet, PaintWorklet, AnimationWorklet)**: Ultra-lightweight, high-priority render/audio pipeline hooks operating directly in the browser's rendering/audio rendering engines with strict real-time deadlines.
> 
> 

> [!danger] Data Passing Overhead: Structured Cloning vs. Transferables
> * **Structured Clone**: Deep copies objects by default. Passing a 100MB object between threads duplicates it in heap memory and can cause a temporary UI hitch during serialization.
> * **Transferable Objects** (`ArrayBuffer`, `MessagePort`, `ImageBitmap`): Transfers ownership of raw memory instantly ($O(1)$) with zero copy. Once transferred, the buffer becomes **detached** (byte length 0) and completely inaccessible on the sender thread.
> 
> 

---

## Common Interview Questions

* "Compare Dedicated Workers, Shared Workers, and Service Workers in terms of lifecycle, scope, and communication patterns."
* "Why can't Web Workers access the DOM or `localStorage`?"
* "What is the Structured Clone algorithm, and how do Transferable Objects avoid memory cloning bottlenecks?"
* "Walk me through the full Service Worker lifecycle: Registration, Installation, Activation, and Fetch interception."
* "What happens if a new Service Worker script has even a 1-byte difference from the currently active worker?"
* "Explain standard Service Worker caching strategies: Cache-First, Network-First, and Stale-While-Revalidate."
* "How does a Shared Worker know when multiple tabs connect to it, and how does it broadcast a message to all open tabs?"

---

## Deep Dive & Talking Points

### 1. Dedicated Web Workers

* **Spawning**: `const worker = new Worker('worker.js')`.
* **Termination**: Call `worker.terminate()` from the main thread, or `self.close()` inside the worker itself.
* **Use Cases**: Parsing gigabyte JSON/CSV files, Markdown rendering, client-side encryption/hashing, image filtering on canvas data.

### 2. Shared Workers

* **Connection**: Connects across multiple tabs/windows via `new SharedWorker('worker.js')`.
* **Port Management**: Every tab connection triggers the `onconnect` event inside the worker, yielding an individual `MessagePort`.
* **Port Explicit Start**: When using `port.addEventListener('message', ...)`, you **must** call `port.start()` explicitly (unlike `port.onmessage`, which starts the port automatically).
* **Use Cases**: Single WebSocket connection shared across 10 open tabs to save server connections; cross-tab synchronization without `localStorage` events.

### 3. Service Workers (The Programmable Network Proxy)

* **Security Requirement**: Runs exclusively over **HTTPS** (or `localhost` for local development) to prevent Man-in-the-Middle (MITM) attacks, because it can intercept and spoof every network request.
* **Scope**: Controlled by directory placement. A worker registered at `/sw.js` can control the whole origin, whereas `/app/sw.js` only controls paths matching `/app/*`.
* **Lifecycle Phases**:
1. **Registration**: Main thread registers the script.
2. **Install (`install` event)**: Worker installs. Best place to pre-cache static assets via `caches.open()` and `cache.addAll()`. Can call `event.waitUntil()` to delay completion, or `self.skipWaiting()` to bypass waiting for old workers to shut down.
3. **Activate (`activate` event)**: Fires when all tabs running the old service worker are closed (unless `skipWaiting()` was called). Ideal for clearing outdated caches (`caches.delete()`). Call `clients.claim()` to immediately take control of uncontrolled open clients.
4. **Idle / Terminated**: Service workers do not stay alive in memory. The browser terminates idle workers to conserve battery/RAM, spinning them back up on demand when a `fetch`, `push`, or `sync` event arrives.
5. **Fetch (`fetch` event)**: Intercepts network calls via `event.respondWith()`.



---

## Code Snippets / Examples

### 1. Dedicated Worker: Zero-Copy Transferable Memory

```javascript
// main.js
const worker = new Worker("heavy-processor.js");

// Allocate 32MB of raw binary memory
const rawBuffer = new ArrayBuffer(32 * 1024 * 1024);
console.log("Main thread buffer before transfer:", rawBuffer.byteLength); // 33554432

// Transfer ownership to worker (zero-copy) by passing it in the transfer list (2nd arg)
worker.postMessage({ buffer: rawBuffer }, [rawBuffer]);

// Memory is detached immediately from main thread!
console.log("Main thread buffer after transfer:", rawBuffer.byteLength); // 0

worker.onmessage = (event) => {
  console.log("Processed buffer received back:", event.data.byteLength);
};

```

```javascript
// heavy-processor.js (Dedicated Worker)
self.onmessage = (event) => {
  const { buffer } = event.data;
  const view = new Uint8Array(buffer);

  // Perform heavy CPU manipulation on binary data
  for (let i = 0; i < view.length; i++) {
    view[i] = (view[i] + 1) % 256;
  }

  // Transfer back to main thread
  self.postMessage(buffer, [buffer]);
};

```

---

### 2. Shared Worker: Coordinating State Across Tabs

```javascript
// tab.js (Run from multiple browser tabs on same origin)
const sharedWorker = new SharedWorker("shared-bus.js");
const port = sharedWorker.port;

port.start();

port.onmessage = (event) => {
  console.log("Broadcast from shared worker:", event.data);
};

// Send message to all tabs
function broadcastMessage(text) {
  port.postMessage({ type: "BROADCAST", payload: text });
}

```

```javascript
// shared-bus.js (Shared Worker)
const connectedPorts = new Set();

self.onconnect = (event) => {
  const port = event.ports[0];
  connectedPorts.add(port);

  port.onmessage = (e) => {
    if (e.data.type === "BROADCAST") {
      // Fan-out message to all other open tabs
      connectedPorts.forEach((p) => {
        p.postMessage(e.data.payload);
      });
    }
  };

  port.start();
};

```

---

### 3. Service Worker: Stale-While-Revalidate Caching Strategy

```javascript
// sw.js (Service Worker)
const CACHE_NAME = "v1-app-cache";

// 1. Install Phase: Pre-cache core shell
self.addEventListener("install", (event) => {
  self.skipWaiting(); // Force activate without waiting for tab refresh
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll(["/", "/index.html", "/styles.css", "/app.js"]);
    })
  );
});

// 2. Activate Phase: Clean up stale caches
self.addEventListener("activate", (event) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(
        keys.map((key) => {
          if (key !== CACHE_NAME) return caches.delete(key);
        })
      )
    ).then(() => self.clients.claim())
  );
});

// 3. Fetch Phase: Stale-While-Revalidate
self.addEventListener("fetch", (event) => {
  // Only handle GET requests for caching
  if (event.request.method !== "GET") return;

  event.respondWith(
    caches.open(CACHE_NAME).then(async (cache) => {
      const cachedResponse = await cache.match(event.request);

      // Fetch latest version in background to update cache
      const networkFetch = fetch(event.request)
        .then((networkResponse) => {
          if (networkResponse.status === 200) {
            cache.put(event.request, networkResponse.clone());
          }
          return networkResponse;
        })
        .catch(() => cachedResponse); // Fallback if offline

      // Return cached asset immediately for speed, or wait for network
      return cachedResponse || networkFetch;
    })
  );
});

```

---

## Comparison Matrix: Browser Workers

| Dimension | Dedicated Web Worker | Shared Worker | Service Worker | Audio/Paint Worklet |
| --- | --- | --- | --- | --- |
| **Instantiation** | `new Worker('w.js')` | `new SharedWorker('w.js')` | `navigator.serviceWorker.register()` | `CSS.paintWorklet.addModule()` |
| **Scope / Cardinality** | 1 Tab : 1 Worker | Many Tabs : 1 Worker | Entire Origin / Scope Path | Render/Audio Pipeline Thread |
| **Communication** | Direct `postMessage` | `MessagePort` per tab | `postMessage` / `BroadcastChannel` | Audio/Paint parameter hooks |
| **Persistence / Lifecycle** | Tied to lifecycle of parent tab | Stays alive while $\ge 1$ tab open | Ephemeral; wakes up on events (`fetch`, `sync`) | High-priority synchronous execution |
| **Network Interception** | No | No | **Yes (`fetch` event)** | No |
| **DOM Access** | No | No | No | No |
| **Primary Use Case** | Offloading heavy CPU tasks | Shared WebSocket, multi-tab state sync | PWAs, offline cache, push notifications | 60/120fps canvas paint, low-latency audio |

---

## Related Topics

* [[Asynchronous JavaScript, Event Loop & Concurrency Model]]
* [[Event Loop Starvation. Causes, Mechanics & Mitigation Strategies|Event Loop Starvation: Causes, Mechanics & Mitigation Strategies]]
* [[V8 Engine Architecture. Parsing, JIT Compilation & Execution Pipeline|V8 Engine Architecture: Parsing, JIT Compilation & Execution Pipeline]]
* [[Client-Side Browser Storage. Mechanisms, Architecture & Security|Browser Storage APIs: IndexedDB, Cache API, and WebSockets]]

---

## Tags

#fullstack #interview #javascript #browser #web-workers #service-workers #shared-workers #pwa #performance

---

## Revision Checklist

* [ ] Can explain in 60 seconds
* [ ] Can explain trade-offs
* [ ] Can give a real project example
* [ ] Can answer common follow-ups