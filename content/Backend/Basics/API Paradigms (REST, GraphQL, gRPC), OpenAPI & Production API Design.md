
## Key Concepts

- **REST (Representational State Transfer):** An architectural style defined by Roy Fielding relying on stateless communication, standard HTTP methods, and resource-oriented URIs (`/resources/:id`).
    
      
    
- **GraphQL:** A query language and runtime created by Meta; enables clients to request exact data shapes in a single round-trip over a single HTTP endpoint (usually `POST /graphql`), eliminating over-fetching and under-fetching.
    
      
    
- **gRPC:** A high-performance RPC framework by Google built on **HTTP/2 transport** and **Protocol Buffers (Protobuf)**; provides strict binary serialization, strongly typed contracts, and bidirectional streaming for microservices.
    
      
    
- **OpenAPI Specification (OAS):** A vendor-neutral, machine-readable format (YAML/JSON) for describing RESTful APIs; enables automated client SDK generation, mock servers, and interactive documentation (Swagger UI).
    
      
    
- **The 6 REST Architectural Constraints:** Client-Server, Stateless, Cacheable, Uniform Interface, Layered System, and Code on Demand (optional).
    
      
    
- **API Response Best Practices:** Consistent envelopes, precise HTTP status codes, structured error formats (RFC 7807 / RFC 9457), deterministic pagination, metadata, and idempotency guarantees.
    
      
    

## Common Interview Questions

- What are Roy Fielding's 6 constraints of REST, and what does it take for an API to be truly "RESTful" (Richardson Maturity Model Level 3 / HATEOAS)?
    
      
    
- How do REST, GraphQL, and gRPC compare in terms of network efficiency, serialization overhead, and tooling?
    
      
    
- How does GraphQL solve over-fetching and under-fetching, and what are its operational trade-offs (e.g., the $N+1$ problem, HTTP caching difficulties)?
    
      
    
- Why is gRPC faster than REST over JSON, and why is it difficult to consume directly from a browser?
    
      
    
- What should a production-ready JSON API response contain for both success and error cases?
    
      
    
- What are the trade-offs between Offset-based and Cursor-based pagination?
    
      
    

## Strong Answers / Talking Points

### 1. REST, GraphQL, and gRPC Comparison

|**Feature**|**REST**|**GraphQL**|**gRPC**|
|---|---|---|---|
|**Data Format**|Typically JSON (can be XML, text).|JSON.|**Protobuf** (compact binary).|
|**Transport**|HTTP/1.1 or HTTP/2.|Typically HTTP/1.1 or HTTP/2 (`POST`).|**HTTP/2 only** (multiplexing, binary framing).|
|**Contract / Schema**|Optional (OpenAPI / Swagger).|Mandatory (GraphQL SDL Schema).|Mandatory (`.proto` interface file).|
|**Over/Under-Fetching**|Common problem (fixed endpoint responses).|**Eliminated** (client defines response query).|Avoided via custom RPC message structures.|
|**Network Caching**|**Trivial & Native** via HTTP headers (`ETag`, `max-age`).|Complex; requests usually share a single `POST` endpoint.|Handled at application layer; no standard HTTP intermediary caching.|
|**Streaming**|Server-Sent Events (SSE), WebSockets.|Subscriptions (via WebSockets/SSE).|**Native bidirectional streaming** over HTTP/2.|
|**Best Used For**|Public APIs, CRUD apps, standard web clients.|Complex frontends with varied UI views, mobile apps.|Internal service-to-service communication (microservices).|

### 2. The 6 REST Constraints

1. **Client-Server Architecture:** Separation of user interface concerns from data storage concerns.
    
      
    
2. **Statelessness:** No client context is stored on the server between requests; every request must contain all necessary session and authentication credentials.
    
      
    
3. **Cacheability:** Responses must explicitly declare whether they are cacheable to prevent clients from reusing stale data (`Cache-Control`).
    
      
    
4. **Uniform Interface:**
    
      
    - Resource identification in requests (URIs: `/users/42`).
        
          
        
    - Resource manipulation through representations (sending JSON to modify state).
        
          
        
    - Self-descriptive messages (proper `Content-Type` headers).
        
          
        
    - **HATEOAS** (Hypermedia as the Engine of Application State): Providing hypermedia links pointing to related actions.
        
          
        
5. **Layered System:** The client cannot tell whether it is connected directly to the end server or to an intermediate proxy, CDN, or load balancer.
    
      
    
6. **Code on Demand (Optional):** Server can extend client functionality by transferring executable code (e.g., JavaScript).
    
      
    

### 3. What to Include When Designing an API Response

When a client requests an API endpoint, returning bare objects or raw arrays is fragile. High-reliability backends adhere to explicit standards:

  

> [!IMPORTANT] Production API Response Checklist
> 
>   
> 
> 1. **Accurate HTTP Status Code:** Never return `200 OK` with `{ success: false, error: "..." }`. Use proper 4xx/5xx codes.
>     
>       
>     
> 2. **Consistent Top-Level Envelope:** Separate data, metadata, and error details predictably.
>     
>       
>     
> 3. **Pagination Metadata:** Provide cursor tokens or total counts, current limits, and navigation markers.
>     
>       
>     
> 4. **RFC 9457 (formerly RFC 7807) Problem Details:** Standardize machine-readable errors (`type`, `title`, `status`, `detail`, `instance`).
>     
>       
>     
> 5. **Correlation / Request ID:** Return an `X-Request-ID` or include a `traceId` in error payloads to link frontend errors directly to backend distributed traces.
>     
>       
>     
> 6. **Deprecation Headers:** Use `Deprecation` and `Sunset` headers when deprecating fields.
>     
>       
>     

### 4. What is OpenAPI (OAS)?

- A standard specification (using YAML or JSON) for describing REST APIs independent of programming language.
    
      
    
- **Why it matters:**
    
      
    - **Contract-First Development:** Frontend and backend teams can agree on API shapes before writing code.
        
          
        
    - **Codegen:** Generates typed TypeScript fetch clients, server boilerplate, and validation schemas automatically.
        
          
        
    - **Documentation & Mocking:** Generates Swagger/Redoc UI documentation and allows mock servers (e.g., Prism) to run against the spec during frontend development.
        
          
        

## Code Snippets / Examples

```TypeScript
// ============================================================================
// 1. Standard Production API Envelopes
// ============================================================================

// Standard Success Envelope (with Cursor Pagination)
export interface ApiResponse<T> {
  success: true;
  data: T;
  meta?: {
    pageInfo?: {
      hasNextPage: boolean;
      nextCursor: string | null;
      limit: number;
    };
    timestamp: string;
  };
}

// RFC 9457 Standardized Error Envelope (Problem Details for HTTP APIs)
export interface ApiErrorResponse {
  type: string;        // URI reference identifying problem type (e.g., "https://api.example.com/errors/not-found")
  title: string;       // Short, human-readable summary
  status: number;      // Matches HTTP status code (e.g., 404, 422)
  detail: string;      // Human-readable explanation specific to this occurrence
  instance?: string;   // URI reference identifying specific request occurrence
  invalidParams?: Array<{ name: string; reason: string }>; // Validation breakdown
  traceId: string;     // Distributed tracing correlation ID
}

// ============================================================================
// 2. Express Controller Implementing Production Response Standards
// ============================================================================
import { Request, Response } from "express";

export async function getUsersHandler(req: Request, res: Response) {
  const requestId = (req.headers["x-request-id"] as string) || crypto.randomUUID();
  const limit = Math.min(parseInt(req.query.limit as string) || 20, 100);
  const cursor = req.query.cursor as string | undefined;

  try {
    const { users, nextCursor, hasMore } = await fetchUsersByCursor({ cursor, limit });

    // Consistent Envelope + Cache Control
    const responsePayload: ApiResponse<typeof users> = {
      success: true,
      data: users,
      meta: {
        pageInfo: {
          hasNextPage: hasMore,
          nextCursor: nextCursor ?? null,
          limit,
        },
        timestamp: new Date().toISOString(),
      },
    };

    res.setHeader("X-Request-ID", requestId);
    res.setHeader("Cache-Control", "private, no-cache");
    return res.status(200).json(responsePayload);

  } catch (error: any) {
    // Standard RFC Error formatting
    const errorPayload: ApiErrorResponse = {
      type: "https://api.example.com/errors/internal-error",
      title: "Internal Server Error",
      status: 500,
      detail: "An unexpected error occurred while querying the user store.",
      instance: req.originalUrl,
      traceId: requestId,
    };

    res.setHeader("X-Request-ID", requestId);
    return res.status(500).json(errorPayload);
  }
}

// Mock query function for context
async function fetchUsersByCursor(opts: { cursor?: string; limit: number }) {
  return { users: [{ id: "usr_1", name: "Alice" }], nextCursor: "usr_1", hasMore: false };
}
```

## Related Topics

- [[HTTP Fundamentals & HTTP-1.1 vs HTTP-2]]
    
      
    
- [[HTTP Methods, CORS, Status Codes & Caching]]
    
      
    
- [[Clean Architecture, Directory Structure & DTOs]]
    
      
    
- [[WebSockets vs WebRTC vs Server-Sent Events (SSE)]]
    
      
    
- [[Database Indexing & Cursor vs Offset Pagination]]
    
      
    

## Tags

#fullstack #interview #api-design #rest #graphql #grpc #openapi #system-design

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups