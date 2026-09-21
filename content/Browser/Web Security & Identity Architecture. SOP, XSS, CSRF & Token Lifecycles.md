
## Key Concepts

> [!summary] Foundations: SOP vs. CORS
> 
>   
> 
> - **Same-Origin Policy (SOP)**: A core browser security mechanism that restricts a document or script loaded by one origin from reading resources or interacting directly with a document from a different origin (`protocol + domain + port`). It prevents `evil.com` from inspecting `bank.com`'s DOM, `localStorage`, or read API responses.
>     
>       
>     
> - **CORS (Cross-Origin Resource Sharing)**: A standard server-driven relaxation of SOP via HTTP headers (`Access-Control-Allow-Origin`, `Access-Control-Allow-Credentials`) that allows designated trusted origins to read server responses.
>     
>       
>     

> [!abstract] Authentication vs. Authorization
> 
>   
> 
> - **Authentication (AuthN)**: _Who are you?_ Validating the caller's identity (passwords, TOTP 2FA, biometric WebAuthn, OAuth SSO).
>     
>       
>     
> - **Authorization (AuthZ)**: _What are you allowed to do?_ Determining permissions and access rights (RBAC: Role-Based Access Control, ABAC: Attribute-Based Access Control). AuthN always precedes AuthZ.
>     
>       
>     

> [!danger] Attack Vectors: XSS vs. CSRF
> 
>   
> 
> - **XSS (Cross-Site Scripting)**: An attacker injects **arbitrary JavaScript** that executes inside the victim's browser context under the trusted origin.
>     
>       
>     - _Types_: Stored (saved in DB), Reflected (reflected in URL/query), DOM-based (injected into client-side DOM sinks).
>         
>           
>         
>     - _Impact_: Session hijacking, exfiltrating tokens/cookies, keylogging, full account takeover.
>         
>           
>         
> - **CSRF (Cross-Site Request Forgery)**: An attacker tricks a victim's authenticated browser into executing an **unwanted state-changing action** on a trusted site where the user is currently logged in.
>     
>       
>     - _Mechanism_: Explains the browser's ambient credential behavior (automatically attaching cookies to cross-site requests). The attacker cannot _read_ the response due to SOP, but the state-changing mutation executes on the server.
>         
>           
>         

> [!tip] Token Topology: Access Tokens vs. Refresh Tokens
> 
>   
> 
> - **Access Token (JWT / Bearer)**: Short-lived (~5–15 minutes). Contains cryptographic claims/scopes. Statelessly validated by microservices via signature verification without constant DB lookups. Kept in **in-memory JavaScript state** to resist XSS.
>     
>       
>     
> - **Refresh Token**: Long-lived (~7–30 days). Opaque string or signed token used exclusively to mint new access tokens. Kept in an **`HttpOnly; Secure; SameSite=Strict` (or `Lax`) cookie** with a restricted `/api/auth/refresh` path to resist both XSS and CSRF.
>     
>       
>     

## Common Interview Questions

- "What is the Same-Origin Policy (SOP), and what does it restrict versus what does it allow (e.g., embedding vs. reading)?"
    
      
    
- "What is the fundamental difference between an XSS attack and a CSRF attack?"
    
      
    
- "Why should you NEVER store Access Tokens in `localStorage` or `sessionStorage`?"
    
      
    
- "How do you protect a modern SPA against CSRF attacks?"
    
      
    
- "Explain the complete token lifecycle: User login $\to$ Token distribution $\to$ API invocation $\to$ Silent token refresh $\to$ Logout."
    
      
    
- "What is Refresh Token Rotation (RTR) and Automatic Reuse Detection?"
    
      
    
- "What is a Content Security Policy (CSP), and how does it mitigate XSS?"
    
      
    

## Deep Dive & Talking Points

### 1. Same-Origin Policy (SOP): What is Blocked vs. Allowed

- **Origin Definition**: `[http://example.com:80/](http://example.com:80/)`
    
      
    - Different protocol (`https`) $\to$ Different origin.
        
          
        
    - Different domain/subdomain (`api.example.com`) $\to$ Different origin.
        
          
        
    - Different port (`:8080`) $\to$ Different origin.
        
          
        
- **Allowed by Default (Cross-Origin Writes / Embeds)**:
    
      
    - Form POST submissions to external origins.
        
          
        
    - Media embeds (`<img src="...">`, `<video>`, `<audio>`).
        
          
        
    - Script executions (`<script src="...">`), stylesheets (`<link rel="stylesheet">`).
        
          
        
- **Blocked by Default (Cross-Origin Reads / Script Access)**:
    
      
    - Reading cross-origin response bodies from `fetch()` / `XMLHttpRequest` (unless allowed by CORS).
        
          
        
    - Reading or manipulating another tab's DOM via `window.opener` or `iframe.contentWindow`.
        
          
        
    - Reading `localStorage`, `sessionStorage`, `IndexedDB`, or cookies belonging to another origin.
        
          
        

### 2. XSS (Cross-Site Scripting): Mechanics & Defenses

- **Sources and Sinks**:
    
      
    - _Source_: Untrusted input (URL params, search boxes, user profiles).
        
          
        
    - _Sink_: Unsafe DOM insertion points (`element.innerHTML`, `outerHTML`, `document.write`, `eval()`).
        
          
        
- **Defenses**:
    
      
    1. **Context-Aware Output Encoding / Escaping**: Never use `innerHTML`. Use `textContent` or framework templating engines (React's JSX auto-escapes string values).
        
          
        
    2. **Sanitization**: For rich-text HTML rendering, sanitize inputs using trusted libraries like **DOMPurify** before injecting via `dangerouslySetInnerHTML`.
        
          
        
    3. **Content Security Policy (CSP)**: HTTP response header (`Content-Security-Policy: default-src 'self'; script-src 'self' [https://trusted.cdn.com](https://trusted.cdn.com)`) that restricts executable script sources, disallows inline scripts (`<script>`), and disables `eval()`.
        
          
        
    4. **`HttpOnly` Cookies**: Prevents JavaScript from reading session/refresh tokens via `document.cookie`.
        
          
        

### 3. CSRF (Cross-Site Request Forgery): Mechanics & Defenses

- **How it works**:
    
      
    - A user logs into `bank.com` and receives a session cookie.
        
          
        
    - The user visits malicious `evil.com` in another tab.
        
          
        
    - `evil.com` contains `<img src="[https://bank.com/transfer?amount=1000&to=attacker](https://bank.com/transfer?amount=1000&to=attacker)">` or an auto-submitting hidden `<form action="[https://bank.com/transfer](https://bank.com/transfer)" method="POST">`.
        
          
        
    - The browser includes `bank.com` cookies automatically with the request, executing the transfer.
        
          
        
- **Defenses**:
    
      
    1. **`SameSite` Cookie Flag**:
        
          
        - `SameSite=Strict`: Cookies never sent on cross-site requests.
            
              
            
        - `SameSite=Lax`: Cookies withheld on cross-site subrequests (images, forms, AJAX), but sent on top-level safe link navigations (`<a>`).
            
              
            
    2. **Anti-CSRF Tokens (Synchronizer Token Pattern)**: A cryptographically random token generated by the server and tied to the user's session. The client must submit this token inside a custom header (e.g., `X-CSRF-Token`) or hidden form field. Because SOP prevents `evil.com` from _reading_ the token from `bank.com`, the attacker cannot include it.
        
          
        
    3. **Custom Headers**: For modern REST/JSON APIs, requiring headers like `X-Requested-With` or `Content-Type: application/json` triggers a CORS preflight (`OPTIONS`), preventing simple cross-origin form exploits.
        
          
        

## Token Architecture: Authentication & Refresh Flow

```
Client (Browser)                 Auth Server / Backend API
      │                                      │
      │ 1. POST /api/auth/login              │
      │    { username, password }            │
      ├─────────────────────────────────────►│ Validate Credentials
      │                                      │ Generate:
      │                                      │ - Access Token (JWT, 15m)
      │ 2. Response:                         │ - Refresh Token (Opaque, 7d)
      │    - Body: { accessToken }           │
      │    - Header: Set-Cookie:             │
      │        refreshToken=xyz...;          │
      │        HttpOnly; Secure;             │
      │        SameSite=Strict; Path=/refresh│
      │◄─────────────────────────────────────┤
      │                                      │
[Store accessToken in Memory]                │
      │                                      │
      │ 3. GET /api/v1/orders                │
      │    Header: Authorization: Bearer <AT>│
      ├─────────────────────────────────────►│ Validates signature statelessly
      │◄─────────────────────────────────────┤ Returns data
      │                                      │
 [Access Token Expires (401 Error)]          │
      │                                      │
      │ 4. POST /api/auth/refresh            │
      │    (Cookie: refreshToken sent auto)  │
      ├─────────────────────────────────────►│ 1. Validates refresh token in DB
      │                                      │ 2. Checks revocation & reuse
      │                                      │ 3. Rotates refresh token (RTR)
      │ 5. Response:                         │
      │    - Body: { accessToken: new_at }   │
      │    - Header: Set-Cookie: new_rt      │
      │◄─────────────────────────────────────┤
[Store new_at in Memory]                     │
      │                                      │
      │ 6. Retry original failed request     │
      ├─────────────────────────────────────►│
```

## Code Snippets & Implementation Patterns

### 1. Hardening Server Cookies (Express.js Example)

JavaScript

```
import express from "express";

const app = express();

app.post("/api/auth/login", async (req, res) => {
  const { user, password } = req.body;
  // Authenticate user...
  const { accessToken, refreshToken } = generateTokens(user.id);

  // Send Refresh Token as a hardened, isolated HTTP-only cookie
  res.cookie("refreshToken", refreshToken, {
    httpOnly: true,                 // Inaccessible to JavaScript (XSS immune)
    secure: process.env.NODE_ENV === "production", // HTTPS only
    sameSite: "strict",             // CSRF immune
    path: "/api/auth/refresh",      // Cookie sent ONLY to refresh endpoint!
    maxAge: 7 * 24 * 60 * 60 * 1000 // 7 days
  });

  // Access Token sent in JSON body to be held in JS memory
  return res.json({ accessToken });
});
```

### 2. Client-Side Silent Refresh & Axios Interceptor Pattern

JavaScript

```
// client/apiClient.js
import axios from "axios";

// Access token stored ONLY in module memory (closure), NEVER in localStorage!
let inMemoryAccessToken = null;

export const setAccessToken = (token) => {
  inMemoryAccessToken = token;
};

export const apiClient = axios.create({
  baseURL: "/api"
});

// Request Interceptor: Attach bearer token
apiClient.interceptors.request.use((config) => {
  if (inMemoryAccessToken) {
    config.headers.Authorization = `Bearer ${inMemoryAccessToken}`;
  }
  return config;
});

// Response Interceptor: Handle 401 and perform Silent Refresh
let isRefreshing = false;
let failedQueue = [];

const processQueue = (error, token = null) => {
  failedQueue.forEach((prom) => {
    if (error) {
      prom.reject(error);
    } else {
      prom.resolve(token);
    }
  });
  failedQueue = [];
};

apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // Detect expired token
    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        // Queue parallel requests while refresh is in flight
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        }).then((token) => {
          originalRequest.headers.Authorization = `Bearer ${token}`;
          return apiClient(originalRequest);
        });
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        // POST to refresh endpoint; browser sends HttpOnly refreshToken cookie automatically
        const { data } = await axios.post("/api/auth/refresh", {}, { withCredentials: true });
        
        setAccessToken(data.accessToken);
        processQueue(null, data.accessToken);

        originalRequest.headers.Authorization = `Bearer ${data.accessToken}`;
        return apiClient(originalRequest);
      } catch (refreshErr) {
        processQueue(refreshErr, null);
        // Refresh token invalid or revoked -> Redirect to login
        window.location.href = "/login";
        return Promise.reject(refreshErr);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);
```

### 3. XSS Sanitization & DOM Insertion

JavaScript

```
import DOMPurify from "dompurify";

function renderUserComment(rawCommentString) {
  // ❌ VULNERABLE: Direct injection allows <script> or <img onerror="..."> execution
  // document.getElementById("comment").innerHTML = rawCommentString;

  // ✅ DEFENSE OPTION A: Preferred (Pure text interpolation)
  const safeTextNode = document.createTextNode(rawCommentString);
  document.getElementById("comment-safe").appendChild(safeTextNode);

  // ✅ DEFENSE OPTION B: Rich HTML with DOMPurify sanitization
  const cleanHTML = DOMPurify.sanitize(rawCommentString, {
    ALLOWED_TAGS: ["b", "i", "em", "strong", "a"],
    ALLOWED_ATTR: ["href", "title"]
  });
  document.getElementById("comment-rich").innerHTML = cleanHTML;
}
```

## Comparison Matrix: Security Vectors & Tokens

|**Dimension**|**Cross-Site Scripting (XSS)**|**Cross-Site Request Forgery (CSRF)**|
|---|---|---|
|**Origin of Execution**|Injected script runs **inside victim's origin**|Request originates from **external attacker origin**|
|**SOP Impact**|SOP is bypassed (attacker runs in same origin)|Attacker cannot read response due to SOP, but request executes|
|**Root Cause**|Unsanitized inputs rendered into DOM|Ambient cookie inclusion on cross-site requests|
|**Primary Defenses**|Output encoding, DOMPurify, CSP, `HttpOnly`|`SameSite=Strict/Lax`, Anti-CSRF tokens, Custom Headers|

|**Token Type**|**Lifespan**|**Storage Location**|**Vulnerability Target**|**Payload Contents**|
|---|---|---|---|---|
|**Access Token (JWT)**|Short (5–15 min)|**In-Memory (JS variable)**|XSS (mitigated by short TTL and not persisting to disk)|User ID, permissions, roles, expiration (`exp`)|
|**Refresh Token**|Long (7–30 days)|**`HttpOnly` Cookie**|CSRF (mitigated by `SameSite` & path scoping)|Opaque session ID or signed token family ID|

## Related Topics

- [[Client-Side Browser Storage: Mechanisms, Architecture & Security]]
    
      
    
- [[What Happens When You Enter a URL in the Browser: The End-to-End Lifecycle]]
    
      
    
- [[Network Protocols: Transport, Security & Application Layers]]
    
      
    
- [[Browser Architecture: High-Level Components, Rendering Engines & HTML Parsing]]
    
      
    

## Tags

#fullstack #interview #security #sop #cors #xss #csrf #jwt #oauth #authentication #authorization #cookies

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups