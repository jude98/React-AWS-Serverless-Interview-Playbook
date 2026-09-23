# HTTP Fundamentals

## Key Concepts

- **Hypertext Transfer Protocol (HTTP):** An Application Layer (Layer 7) request-response protocol designed to transfer hypermedia documents across distributed systems.

- **Stateless Nature:** Each HTTP request is executed independently; the server retains no implicit session context or memory of previous requests from the same client.

- **State Management Workarounds:** Because HTTP is stateless, state is simulated at the application layer via cookies, session identifiers (`Set-Cookie`), JWTs, and distributed caches (e.g., Redis).

- **HTTP/1.1 Core Traits:** Introduced persistent TCP connections (`Keep-Alive`), chunked transfer encoding, and mandatory `Host` headers. However, it relies on human-readable plaintext framing and suffers from application-layer Head-of-Line (HoL) blocking.

- **HTTP/2 Core Traits:** Introduced binary framing, request/response multiplexing over a single TCP connection, header compression via HPACK, and Server Push.

- **The Core Upgrade:** HTTP/2 eliminates application-layer HoL blocking by interleaving independent streams over a single connection, though transport-layer HoL blocking persists due to TCP.

## Common Interview Questions

- What is HTTP, and what does it actually mean when we say HTTP is a "stateless" protocol?

- If HTTP is stateless, how do modern web applications handle user authentication, shopping carts, and persistent sessions?

- What are the major limitations of HTTP/1.1 that motivated the creation of HTTP/2?

- How does HTTP/2 multiplexing work under the hood using frames and streams?

- What is HPACK, and why couldn't HTTP/2 use standard gzip compression for headers?

- What is Head-of-Line (HoL) blocking in HTTP/1.1 versus HTTP/2, and why did this eventually drive the development of HTTP/3?

## Strong Answers / Talking Points

### 1. HTTP Foundations & The Stateless Architecture

- **Why HTTP was designed stateless:**

    - _Simplicity & Scalability:_ Servers do not need to allocate memory or synchronize session states across distributed clusters. Any server behind a load balancer can handle any incoming request.

    - _Fault Tolerance:_ If an application server crashes between requests, no session state is lost on the server side; the next request can route to an entirely different node seamlessly.

- **How state is maintained in production:**

    - **Cookies & Server-Side Sessions:** Server sets a cryptographically random session ID via the `Set-Cookie` response header; client sends it back via the `Cookie` request header. State lives in a shared store (Redis).

    - **Stateless Tokens (JWT):** The client stores a signed payload. The server validates the cryptographic signature without querying a shared session store.

### 2. HTTP/1.1 vs. HTTP/2 Architecture Comparison

|**Feature**|**HTTP/1.1**|**HTTP/2**|
|---|---|---|
|**Protocol Format**|Plaintext (ASCII / human-readable). Prone to parsing errors and whitespace overhead.|**Binary framing layer** (parses into lightweight Frames and Streams).|
|**Connection Model**|Sequential requests per connection (1 request active per TCP socket at a time).|Fully **multiplexed** over a **single TCP connection**.|
|**Head-of-Line (HoL) Blocking**|**Application-Layer HoL:** A slow or stalled response blocks all subsequent queued responses on that TCP socket.|Solved at the application layer via interleaved frames. _(TCP-level HoL still exists if packets drop)._|
|**Header Handling**|Redundant plaintext headers sent on every request (often 500B–2KB per round-trip).|**HPACK compression**: Uses a static/dynamic lookup table to transmit only header diffs.|
|**Resource Optimization**|Required anti-patterns: domain sharding (opening 6 TCP sockets per domain), image spriting, inlining assets.|Anti-patterns deprecated. All assets stream concurrently over a single socket.|
|**Server Capabilities**|Strict request-then-response cycle.|**Server Push** (server can proactively push static assets to client cache; largely superseded in practice by `103 Early Hints`).|

### 3. Deep Dive: How Multiplexing Works in HTTP/2

> [!NOTE] The Binary Framing Breakdown
> 
> HTTP/2 breaks all communication down into binary-encoded messages:
> 
>   
> 
> - **Frame:** The smallest unit of communication (e.g., `HEADERS` frame for metadata, `DATA` frame for payload).
>     
>       
>     
> - **Stream:** A bidirectional flow of bytes within an established TCP connection, carrying a unique stream ID.
>     
>       
>     
> - **Message:** A complete sequence of frames that map to an HTTP request or response.
>     
>       
>     

- Rather than waiting for Request 1 to complete before dispatching Request 2, HTTP/2 cuts both payloads into frames with distinct `Stream ID` tags (e.g., Stream 1, Stream 3).

- Frames from different streams are interleaved across the single TCP pipe and reassembled by the receiver based on their IDs, eliminating application-layer serialization delays.

### 4. HPACK and Security Considerations

- HTTP/1.1 allowed body compression (`gzip`, `brotli`), but headers remained raw plaintext.

- Naive header compression using `gzip` caused the **CRIME / BREACH** attacks: attackers could deduce secret tokens (cookies) by observing changes in compressed payload lengths when injecting known plaintext.

- **HPACK** was specifically built to avoid this vulnerability: it separates literal values, maintains static and dynamic indexing tables between client and server, and uses Huffman coding without cross-compressing untrusted inputs against secrets.

## Code Snippets / Examples

```typescript
import http2 from "node:http2";
import fs from "node:fs";

// ============================================================================
// Native HTTP/2 Multiplexed Server in Node.js
// Note: Browsers mandate TLS (ALPN negotiation) for HTTP/2
// ============================================================================
const server = http2.createSecureServer({
  key: fs.readFileSync("localhost-privkey.pem"),
  cert: fs.readFileSync("localhost-cert.pem"),
});

server.on("stream", (stream, headers) => {
  const path = headers[":path"];
  const method = headers[":method"];

  console.log(`Incoming ${method} on Stream ID: ${stream.id} for ${path}`);

  if (path === "/") {
    // Respond to Stream 1
    stream.respond({
      ":status": 200,
      "content-type": "text/html; charset=utf-8",
    });
    stream.end("<h1>Served over HTTP/2 Binary Framing</h1>");
  } else if (path === "/api/data") {
    // Concurrently handle another stream on the exact same underlying TCP connection
    stream.respond({
      ":status": 200,
      "content-type": "application/json",
      "cache-control": "no-cache",
    });
    stream.end(JSON.stringify({ status: "success", multiplexed: true }));
  } else {
    stream.respond({ ":status": 404 });
    stream.end("Not Found");
  }
});

server.listen(8443, () => {
  console.log("HTTP/2 server listening on port 8443");
});
```

## Related Topics

- [[TCP vs. UDP|TCP vs UDP]]

- [[Network Protocols. Transport, Security & Application Layers|HTTP-3 and QUIC]]

- [[API Paradigms (REST, GraphQL, gRPC), OpenAPI & Production API Design|RESTful API Design & Idempotency]]

- [[Web Security & Identity Architecture. SOP, XSS, CSRF & Token Lifecycles|Web Security: Cookies, CSRF, and CORS]]

- [[Network Protocols. Transport, Security & Application Layers|TLS Handshake & ALPN (Application-Layer Protocol Negotiation)]]

## Tags

#fullstack #interview #networking #system-design #protocols #http #web-architecture

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups