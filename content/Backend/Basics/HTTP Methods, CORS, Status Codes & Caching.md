# HTTP Methods, CORS, Status Codes & Caching

## Key Concepts

- **HTTP Methods:** Verbs indicating the desired action on a target resource.
    
      
    
- **Safe Methods:** Methods that do not alter server state (read-only): `GET`, `HEAD`, `OPTIONS`.
    
      
    
- **Idempotency:** A method is idempotent if executing it multiple times produces the exact same server state as a single execution ($f(f(x)) = f(x)$).
    
      
    - _Correction:_ **`POST` is NOT idempotent**; repeated calls typically create multiple distinct resources or trigger side effects repeatedly. `PUT`, `DELETE`, and `GET` **are** idempotent.
        
          
        
- **`OPTIONS`:** A preflight/discovery method asking the server which HTTP methods and headers are supported for a specific URL or origin.
    
      
    
- **CORS (Cross-Origin Resource Sharing):** A browser-enforced security sandbox based on the **Same-Origin Policy (SOP)** (Origin = Scheme + Host + Port).
    
      
    - Postman, curl, and backend SDKs do not enforce CORS because they lack a sandbox holding ambient user credentials (cookies, local storage) across origins.
        
          
        
- **Preflight vs. Simple Requests:** Simple requests skip the preflight check; non-simple requests trigger an automatic `OPTIONS` preflight before the actual request executes.
    
      
    
- **HTTP Caching:** Reduces latency and network bandwidth through expiration headers (`Cache-Control`) and revalidation validators (`ETag`, `Last-Modified`).
    
      
    

## Common Interview Questions

- Why is `POST` non-idempotent while `PUT` is idempotent? How does `PATCH` fit in?
    
      
    
- Why does CORS exist in browsers, and why doesn't Postman or curl trigger CORS errors?
    
      
    
- What exact criteria differentiate a "Simple Request" from a "Preflighted Request"?
    
      
    
- What is the operational difference between `301 Moved Permanently` and `302 Found` (or `307 Temporary Redirect`)?
    
      
    
- What are `204 No Content` and `304 Not Modified`, and why do they omit a response body?
    
      
    
- How do `Cache-Control: no-cache`, `no-store`, and `ETag` work together during HTTP revalidation?
    
      
    

## Strong Answers / Talking Points

### 1. HTTP Methods: Idempotency & Semantics

|**Method**|**Safe?**|**Idempotent?**|**Primary Purpose**|
|---|---|---|---|
|**GET**|**Yes**|**Yes**|Retrieve resource representation without side effects.|
|**POST**|No|**No**|Create subordinate resources, submit forms, or trigger state transitions. Multiple requests create multiple resources.|
|**PUT**|No|**Yes**|Replace the target resource entirely with the request payload. Re-executing replaces it with the exact same representation.|
|**PATCH**|No|**No***|Apply partial modifications. (_Note: Not inherently idempotent, e.g., `{ $inc: { views: 1 } }` vs. `{ name: "Bob" }`)._|
|**DELETE**|No|**Yes**|Remove resource. First call deletes it (200/204); subsequent calls leave it deleted (404/204). State remains identical.|
|**OPTIONS**|**Yes**|**Yes**|Inspect communication options/capabilities supported by the server.|

> [!WARNING] Common Interview Trap: POST vs. PUT
> 
>   
> 
> - **`POST /users`**: Creates a new user record. If called 5 times, it generates 5 unique records with different IDs. **Not idempotent.**
>     
>       
>     
> - **`PUT /users/42`**: Replaces user 42 with the incoming representation. If called 5 times, user 42 still has that exact payload. **Idempotent.**
>     
>       
>     

### 2. CORS, Preflight, Simple vs. Non-Simple Requests

#### A. Why Browsers Enforce CORS (And Postman Doesn't)

- **Same-Origin Policy (SOP):** Prevents a malicious site (`evil.com`) running in your browser from reading data returned from `yourbank.com` using your stored session cookies.
    
      
    
- **CORS is a relaxation of SOP:** It allows servers to explicitly declare (`Access-Control-Allow-Origin`) that external frontends are permitted to read their responses.
    
      
    
- **Why Postman/curl don't enforce CORS:** CORS is a **client-side browser guard**, not a server-side firewall. The server processes requests regardless; the browser simply hides the response from JavaScript unless proper headers return. Tools like Postman have no ambient user context (cookies/sessions) to hijack across origins.
    
      
    

#### B. Simple Request vs. Non-Simple Request

A request is **Simple** (no `OPTIONS` preflight sent) ONLY if ALL three conditions hold:

  

1. **Method:** Must be `GET`, `HEAD`, or `POST`.
    
      
    
2. **Headers:** Only safe-listed headers: `Accept`, `Accept-Language`, `Content-Language`, and `Content-Type`.
    
      
    
3. **Content-Type:** Strictly limited to:
    
      
    - `application/x-www-form-urlencoded`
        
          
        
    - `multipart/form-data`
        
          
        
    - `text/plain`
        
          
        

If you use `application/json` (standard in modern APIs), a custom header (`Authorization`, `X-Api-Key`), or methods like `PUT`/`DELETE`/`PATCH`, it is a **Non-Simple Request**. The browser will send an automatic `OPTIONS` preflight first.

  

```
Client (Browser)                           Server
      |                                       |
      |--- 1. OPTIONS /api/data ------------->|  (Preflight: Checks permissions)
      |<-- 2. 204 No Content (CORS Headers) --|  (Access-Control-Allow-Methods, etc.)
      |                                       |
      |--- 3. POST /api/data (Actual Payload)->|  (Executes business logic)
      |<-- 4. 200 OK (Data response) ---------|
```

### 3. HTTP Status Codes Cheatsheet

#### 2xx Success

- **200 OK:** Standard success response with payload.
    
      
    
- **201 Created:** Resource successfully created; usually accompanies `POST`/`PUT` and includes a `Location` header.
    
      
    
- **204 No Content:** Request succeeded, but server deliberately returns no body (common for `DELETE` or empty `PUT`/`OPTIONS`).
    
      
    

#### 3xx Redirection

- **300 Multiple Choices:** Rarely used; server suggests multiple available representations.
    
      
    
- **301 Moved Permanently:** Resource assigned a new permanent URI. Browsers cache this redirect aggressively and historically rewrite `POST` to `GET`.
    
      
    
- **302 Found:** Temporary redirect. Historically rewritten to `GET` by browsers (use `307 Temporary Redirect` to strictly preserve the original method like `POST`).
    
      
    
- **304 Not Modified:** Sent during conditional cache validation (`If-None-Match`). Informs the client its cached copy is fresh; returns **no response body**.
    
      
    

#### 4xx Client Errors

- **400 Bad Request:** Malformed syntax, invalid JSON payload, or schema validation failure.
    
      
    
- **401 Unauthorized:** Missing or invalid authentication token/credentials (actually means _Unauthenticated_).
    
      
    
- **403 Forbidden:** Authenticated, but lacking permission/role to access resource.
    
      
    
- **404 Not Found:** Target URI does not exist.
    
      
    

#### 5xx Server Errors

- **500 Internal Server Error:** Generic unhandled runtime exception on the server.
    
      
    
- **502 Bad Gateway:** Reverse proxy (e.g., NGINX, Cloudflare) received an invalid response from upstream application server.
    
      
    
- **503 Service Unavailable:** Server overloaded or down for maintenance.
    
      
    
- **504 Gateway Timeout:** Upstream application server failed to respond within the reverse proxy's timeout window.
    
      
    

### 4. HTTP Caching: `Cache-Control` & `ETag`

- **`Cache-Control` (Freshness Control):**
    
      
    - `max-age=N`: Cached response is valid for $N$ seconds without contacting the server.
        
          
        
    - `no-cache`: Must validate freshness with the origin server (via `ETag`) before serving cached copy.
        
          
        
    - `no-store`: Never cache anything in memory or disk (sensitive banking/PII data).
        
          
        
    - `public` vs. `private`: `public` allows intermediate CDNs to cache; `private` restricts caching strictly to the end-user browser.
        
          
        
- **`ETag` (Entity Tag - Revalidation Validator):**
    
      
    - A cryptographic hash or version token of the resource representation.
        
          
        
    - **Revalidation Cycle:**
        
          
        1. Server sends: `ETag: "v1-hash-abc"`.
            
              
            
        2. Later, browser sends: `If-None-Match: "v1-hash-abc"`.
            
              
            
        3. If hash matches $\rightarrow$ Server replies with **`304 Not Modified`** (0 byte body, saves bandwidth).
            
              
            
        4. If hash changed $\rightarrow$ Server replies with **`200 OK`** and the new resource body.
            
              
            

## Code Snippets / Examples

```TypeScript
import http from "node:http";
import crypto from "node:crypto";

const server = http.createServer((req, res) => {
  // ==========================================================================
  // 1. Handling CORS & Preflight (OPTIONS)
  // ==========================================================================
  res.setHeader("Access-Control-Allow-Origin", "https://app.example.com");
  res.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
  res.setHeader("Access-Control-Allow-Headers", "Content-Type, Authorization");
  res.setHeader("Access-Control-Max-Age", "86400"); // Cache preflight for 24h

  // Preflight interceptor
  if (req.method === "OPTIONS") {
    res.writeHead(204); // No content needed for preflight acknowledgment
    res.end();
    return;
  }

  // ==========================================================================
  // 2. HTTP Caching with Cache-Control and ETag Revalidation
  // ==========================================================================
  if (req.method === "GET" && req.url === "/api/resource") {
    const payload = JSON.stringify({ data: "high-value-state", version: 42 });
    
    // Generate deterministic hash for entity tag
    const etag = crypto.createHash("md5").update(payload).digest("hex");

    // Check if client provided conditional matching header
    const clientETag = req.headers["if-none-match"];

    if (clientETag === etag) {
      // Resource unchanged: save bandwidth with 304 Not Modified
      res.writeHead(304);
      res.end();
      return;
    }

    // Fresh payload with revalidation requirements
    res.writeHead(200, {
      "Content-Type": "application/json",
      "Cache-Control": "public, no-cache", // Revalidate with server every time
      "ETag": etag,
    });
    res.end(payload);
    return;
  }

  res.writeHead(404);
  res.end();
});

server.listen(3000);
```

## Related Topics

- [[HTTP Fundamentals|HTTP Fundamentals & HTTP-1.1 vs HTTP-2]]
    
      
    
- [[API Paradigms (REST, GraphQL, gRPC), OpenAPI & Production API Design|RESTful API Design & Idempotency]]
    
      
    
- [[Web Security & Identity Architecture. SOP, XSS, CSRF & Token Lifecycles|Web Security: SOP, CORS, CSRF, and XSS]]
    
      
    
- [[Caching Architecture, Eviction Policies & Invalidation Pitfalls|CDN Architecture and Edge Caching]]
    
      
    
- [[TCP vs. UDP|TCP vs UDP]]
    
      
    

## Tags

#fullstack #interview #networking #http #cors #caching #web-security #system-design

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups