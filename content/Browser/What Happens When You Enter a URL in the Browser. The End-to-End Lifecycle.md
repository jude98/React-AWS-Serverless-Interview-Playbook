# What Happens When You Enter a URL in the Browser: The End-to-End Lifecycle

## Key Concepts

> [!summary] The High-Level Request Pipeline
> 
> When a user enters a URL (e.g., `[https://api.example.com/v1/users](https://api.example.com/v1/users)`) into the browser's address bar and hits **Enter**, the browser initiates a multi-layered journey spanning the application layer, OS networking, transport protocols, edge routing, server processing, and client-side rendering:
> 
>   
> 
> $$\text{URL Parsing \& HSTS} \to \text{DNS Resolution} \to \text{TCP Handshake} \to \text{TLS Negotiation} \to \text{HTTP Request/Response} \to \text{Critical Rendering Path (CRP)}$$

> [!abstract] 1. Navigation & URL Parsing
> 
>   
> 
> - **Input Processing**: The browser distinguishes whether the text is a valid search term (passed to default search engine) or a protocol/domain URI.
>     
>       
>     
> - **URL Anatomy**: `[https://api.example.com:443/v1/users?sort=asc#profile](https://api.example.com:443/v1/users?sort=asc#profile)`
>     
>       
>     - **Scheme/Protocol**: `https`
>         
>           
>         
>     - **Host / Domain**: `api.example.com`
>         
>           
>         
>     - **Port**: `:443` (implicit for HTTPS; `:80` for HTTP)
>         
>           
>         
>     - **Path**: `/v1/users`
>         
>           
>         
>     - **Query String**: `?sort=asc`
>         
>           
>         
>     - **Fragment / Hash**: `#profile` (never sent to the server; resolved entirely by the client/DOM).
>         
>           
>         
> - **HSTS (HTTP Strict Transport Security)**: The browser checks its internal preloaded HSTS list. If present, it forces HTTPS immediately, preventing plaintext `http://` downgrade attacks (SSL stripping).
>     
>       
>     

> [!info] 2. DNS Resolution (Resolving Domain to IP)
> 
> The browser translates human-readable hostnames into network routable IP addresses (IPv4/IPv6) via a hierarchical caching lookup:
> 
>   
> 
> 1. **Browser DNS Cache**: Chrome (`chrome://net-internals/#dns`).
>     
>       
>     
> 2. **OS Cache & Hosts File**: Checks local operating system DNS cache and the `/etc/hosts` file.
>     
>       
>     
> 3. **Recursive Resolver (ISP / 8.8.8.8 / 1.1.1.1)**: If uncached, queries recursively:
>     
>       
>     - **Root Nameservers (`.`)**: Directs to the Top-Level Domain (TLD) servers.
>         
>           
>         
>     - **TLD Nameservers (`.com`)**: Directs to the Authoritative nameserver for `example.com`.
>         
>           
>         
>     - **Authoritative Nameservers**: Returns the final `A` (IPv4) or `AAAA` (IPv6) record.
>         
>           
>         

> [!tip] 3. Transport & Security (TCP Handshake & TLS 1.3)
> 
>   
> 
> - **TCP 3-Way Handshake**:
>     
>       
>     - Client sends **SYN** (Synchronize).
>         
>           
>         
>     - Server replies with **SYN-ACK** (Synchronize-Acknowledge).
>         
>           
>         
>     - Client replies with **ACK** (Acknowledge). A reliable, ordered byte stream is established ($1\text{ RTT}$).
>         
>           
>         
> - **TLS 1.3 Cryptographic Handshake**:
>     
>       
>     - **ClientHello**: Supported cipher suites and key share (Diffie-Hellman parameters).
>         
>           
>         
>     - **ServerHello & Certificate**: Server selects cipher suite, sends public key share, and provides its X.509 SSL/TLS certificate.
>         
>           
>         
>     - **Certificate Verification**: Browser validates the certificate against its Root Certificate Authorities (CA store) via CRL/OCSP stapling.
>         
>           
>         
>     - **Session Keys Derived**: Both parties compute symmetric encryption keys. TLS 1.3 establishes encrypted communication in just **$1\text{ RTT}$** (or $0\text{ RTT}$ via session resumption).
>         
>           
>         

> [!danger] 4. HTTP Round-Trip, Gateway & Web Server Handling
> 
>   
> 
> - **HTTP Request**: The client dispatches headers (`Host`, `User-Agent`, `Accept`, `Cookie`, `Authorization`).
>     
>       
>     
> - **Edge Network / CDN (Cloudflare, Fastly)**: Terminates TLS close to the user (Anycast DNS routing), serves static assets from edge cache, or forwards dynamic requests upstream.
>     
>       
>     
> - **Reverse Proxy / Load Balancer (Nginx, HAProxy, AWS ALB)**: Balances traffic, applies rate-limiting, and routes to application server instances.
>     
>       
>     
> - **Server Processing & Response**: The application runs logic, queries the database, and returns an HTTP status code (e.g., `200 OK`) with response headers (`Content-Type`, `Cache-Control`, `Set-Cookie`) and payload bytes (HTML stream).
>     
>       
>     

## The Critical Rendering Path (CRP)

Once the first chunks of the HTML document arrive over the network socket, the browser rendering engine (e.g., Blink) executes the **Critical Rendering Path**:

  

```
HTML Bytes ──> Tokens ──> DOM Tree ──┐
                                     ├──> Render Tree ──> Layout (Reflow) ──> Paint ──> Composite
CSS Bytes  ──> Tokens ──> CSSOM Tree ┘
```

1. **DOM Construction**: HTML parser converts raw bytes $\to$ characters $\to$ tokens $\to$ nodes $\to$ **DOM (Document Object Model)** tree. Parsing is progressive (streaming).
    
      
    
2. **CSSOM Construction**: CSS bytes are parsed into the **CSSOM (CSS Object Model)**. CSS is **render-blocking**: the browser will not render pixels until the CSSOM is constructed to prevent Flash of Unstyled Content (FOUC).
    
      
    
3. **JavaScript Execution**: `<script>` tags without `async` or `defer` **block HTML parsing** because JS can mutate the DOM via `document.write()`.
    
      
    
4. **Render Tree Generation**: The DOM and CSSOM combine into the **Render Tree**. It excludes non-visual nodes (`<head>`, `<meta>`, elements with `display: none`). Note that `visibility: hidden` elements _are_ included because they take up geometric space.
    
      
    
5. **Layout (Reflow)**: Computes the exact physical coordinates and bounding box dimensions for every visible node on the viewport grid.
    
      
    
6. **Paint**: Converts geometric render tree nodes into actual colored pixels on display buffers across multiple visual layers.
    
      
    
7. **Compositing**: The GPU composites distinct layers (e.g., elements with `transform`, `opacity`, or `will-change`) into the final frame displayed on the user's monitor.
    
      
    

## Common Interview Questions

- "Walk me through what happens from typing a URL to seeing pixels on the screen."
    
      
    
- "What is the difference between TCP 3-Way Handshake and the TLS Handshake?"
    
      
    
- "Why is CSS considered render-blocking, while JavaScript is parser-blocking?"
    
      
    
- "What is the difference between `script defer` and `script async`?"
    
      
    
- "What is HSTS, and why is an initial HTTP $\to$ HTTPS redirect vulnerable without it?"
    
      
    
- "What is the difference between Layout (Reflow), Repaint, and Compositing?"
    
      
    
- "How does HTTP/2 and HTTP/3 (QUIC) optimize this initial connection flow compared to HTTP/1.1?"
    
      
    

## Deep Dive & Talking Points

### 1. Modern Transport Protocols: HTTP/1.1 vs HTTP/2 vs HTTP/3

- **HTTP/1.1**: Subject to **Head-of-Line (HoL) Blocking** at the application layer. Browsers maintain up to 6 concurrent TCP connections per origin.
    
      
    
- **HTTP/2**: Introduces binary framing and **multiplexing** over a single TCP connection. However, it still suffers from TCP-level HoL blocking (if one TCP packet drops, all multiplexed streams wait for packet retransmission).
    
      
    
- **HTTP/3 (QUIC)**: Replaces TCP with **UDP**. Integrates the cryptographic handshake directly into the transport layer, eliminating TCP+TLS connection overhead ($1\text{ RTT}$ connection setup) and solving HoL blocking entirely.
    
      
    

### 2. Script Execution: Default vs. `defer` vs. `async`

- `<script src="...">`: Halts HTML parser, fetches over network, executes immediately, then resumes HTML parsing.
    
      
    
- `<script async src="...">`: Downloads in parallel with HTML parsing. As soon as it downloads, it **pauses the HTML parser to execute immediately**. Execution order is non-deterministic (whichever file finishes downloading first runs first).
    
      
    
- `<script defer src="...">`: Downloads in parallel with HTML parsing. Does **not** block parsing; waits until the HTML parser finishes completely and executes right before `DOMContentLoaded` in strict document order.
    
      
    

### 3. Layout vs. Paint vs. Composite (Jank Optimization)

- **Reflow (Layout)**: Changes that alter geometry (e.g., `width`, `height`, `margin`, `fontSize`). Heavy CPU calculation that ripples down the DOM tree.
    
      
    
- **Repaint**: Changes that alter appearance without affecting geometry (e.g., `color`, `background-color`, `box-shadow`). Skips layout, runs paint.
    
      
    
- **Composite-Only**: Changes handled directly by the GPU (e.g., `transform: translate3d()`, `opacity`). Bypasses both layout and paint, executing smoothly at 60/120 fps.
    
      
    

## Code Snippets / Examples

### 1. Optimizing Critical Path Loading in HTML

HTML

```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Optimized Page Load</title>

  <!-- 1. Resource Hints: Speed up DNS and TLS handshakes for external APIs -->
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="dns-prefetch" href="https://analytics.example.com">

  <!-- 2. Critical Inlined CSS: Prevents render blocking for above-the-fold content -->
  <style>
    body { margin: 0; font-family: sans-serif; }
    .hero { height: 100vh; background: #111; color: #fff; }
  </style>

  <!-- 3. Non-critical CSS loaded asynchronously -->
  <link rel="preload" href="/styles/non-critical.css" as="style" onload="this.rel='stylesheet'">

  <!-- 4. Defer JavaScript execution until DOM parsing completes -->
  <script src="/scripts/app.js" defer></script>

  <!-- 5. Independent third-party scripts run asynchronously -->
  <script src="https://analytics.example.com/tracker.js" async></script>
</head>
<body>
  <div class="hero">Above the fold content</div>
</body>
</html>
```

### 2. Triggering Layout Thrashing / Forced Synchronous Reflow

```javascript
// ❌ ANTI-PATTERN: Forced Synchronous Layout (Layout Thrashing)
// Alternating reads and writes inside a loop forces the engine to recalculate layout repeatedly
function resizeBad(boxes) {
  for (let i = 0; i < boxes.length; i++) {
    // READ: forces immediate style/layout calculation
    const currentWidth = boxes[i].offsetWidth; 
    // WRITE: invalidates layout
    boxes[i].style.width = (currentWidth + 10) + "px"; 
  }
}

// ✅ FIX: Batch reads, then batch writes
function resizeGood(boxes) {
  // 1. Batch read operations
  const widths = boxes.map(box => box.offsetWidth);

  // 2. Batch write operations
  boxes.forEach((box, i) => {
    box.style.width = (widths[i] + 10) + "px";
  });
}
```

## Comparison Matrix: Script Loading Attributes

|**Attribute**|**Download Phase**|**Blocks HTML Parser?**|**Execution Timing**|**Execution Order Guaranteed?**|**Best Used For**|
|---|---|---|---|---|---|
|**Standard `<script>`**|Blocks parser|**Yes**|Immediately upon download|Yes (document order)|Critical polyfills that must run before any HTML|
|**`<script async>`**|In parallel (background)|Only during execution|Immediately when downloaded|**No** (fastest download executes first)|Independent analytics, ads, tracking pixels|
|**`<script defer>`**|In parallel (background)|**No**|After DOM parsing completes, before `DOMContentLoaded`|**Yes** (document order)|Application bundles, libraries depending on DOM|

## Related Topics

- [[Client-Side Browser Storage. Mechanisms, Architecture & Security|Client-Side Browser Storage: Mechanisms, Architecture & Security]]
    
      
    
- [[V8 Engine Architecture. Parsing, JIT Compilation & Execution Pipeline|V8 Engine Architecture: Parsing, JIT Compilation & Execution Pipeline]]
    
      
    
- [[Asynchronous JavaScript, Event Loop & Concurrency Model]]
    
      
    
- [[DOM Event Propagation. Bubbling, Capturing & Event Delegation|DOM Event Propagation: Bubbling, Capturing & Event Delegation]]
    
      
    

## Tags

#fullstack #interview #networking #browser #critical-rendering-path #dns #tcp #tls #http #rendering

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups