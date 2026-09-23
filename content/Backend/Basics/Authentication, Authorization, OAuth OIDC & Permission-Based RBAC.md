# Authentication, Authorization, OAuth OIDC & Permission-Based RBAC

## Key Concepts

- **Authentication (AuthN) vs. Authorization (AuthZ):** AuthN answers _"Who are you?"_ (verifying identity). AuthZ answers _"What are you allowed to do?"_ (verifying permissions/access).
    
      
    
- **Stateful Sessions:** Client receives an opaque session ID (`sid`) stored in an `HttpOnly` cookie. The server maintains session state in a centralized store (e.g., Redis). Immediate invalidation is straightforward, but it requires distributed storage at scale.
    
      
    
- **Stateless Tokens (JWT):** A JSON Web Token encodes cryptographically signed claims (`header.payload.signature`). The backend verifies the signature using an asymmetric public key or shared secret without querying a central database. Immediate revocation is challenging without a token blocklist.
    
      
    
- **Bearer Token Scheme:** An HTTP authentication scheme (`Authorization: Bearer <token>`) where possession of the token grants access—no proof of key ownership is required by the carrier.
    
      
    
- **OAuth 2.0 vs. OpenID Connect (OIDC):**
    
      
    - **OAuth 2.0** is an **authorization framework** granting delegated access (issues `access_token` scoped for APIs). It does not standardize user identity.
        
          
        
    - **OIDC** is an **identity layer built on top of OAuth 2.0** (issues an `id_token` as a JWT to standardize authentication and profile retrieval).
        
          
        
- **OAuth 1.0 vs. 2.0:** OAuth 1.0 relied on cryptographic request signing (HMAC-SHA1) on every call. OAuth 2.0 delegated transport security to TLS/HTTPS and introduced distinct Authorization Grants (e.g., Authorization Code Flow with PKCE).
    
      
    
- **Role-Based Access Control (RBAC) vs. Permission-Based Access Control (PBAC):** Hardcoding checks like `if (role === 'admin')` causes role explosion and fragile code. Modern enterprise systems decouple users from roles, and map roles to granular permissions (`resource:action`). **Code checks permissions, never roles.**
    
      
    

## Common Interview Questions

- What are the architectural trade-offs between stateful session cookies and stateless JWTs?
    
      
    
- What security flags are mandatory for session cookies, and what attack vectors do they mitigate?
    
      
    
- Why is OAuth 2.0 not an authentication protocol by itself, and how does OpenID Connect solve that?
    
      
    
- What are the major vulnerabilities of OAuth 1.0, and why was OAuth 2.0 designed to replace it?
    
      
    
- How do you implement instant revocation for stateless JWTs when a user logs out or is compromised?
    
      
    
- Why should applications check granular permissions (e.g., `posts:delete`) instead of roles (e.g., `role === 'editor'`) in both frontend and backend architectures?
    
      
    

## Strong Answers / Talking Points

### 1. Stateful Sessions vs. Stateless JWTs

|**Dimension**|**Stateful Sessions (Cookie-Based)**|**Stateless Tokens (JWT / Bearer)**|
|---|---|---|
|**State Location**|Server-side (RAM, Redis, database).|Client-side (in memory, service worker, or secure storage).|
|**Token Format**|Opaque random string (e.g., `sess_98fbc2...`).|Cryptographically signed, structured JSON (`base64Url`).|
|**Revocation**|**Instant.** Delete or expire the key in Redis.|**Difficult.** Valid until expiration (`exp`) unless tracked via an active Redis blocklist.|
|**Scalability**|Requires centralized cache lookups on every incoming request.|Zero DB lookups for validation; verified via CPU signature checks.|
|**Cross-Domain / APIs**|Difficult across disparate domains/native mobile apps due to cookie constraints.|Native-friendly; passed in headers (`Authorization: Bearer <token>`).|

### 2. Cookie Security Flags

When storing session identifiers or refresh tokens in cookies:

  

- **`HttpOnly`:** Disallows client-side JavaScript access via `document.cookie` (neutralizes cross-site scripting / XSS token theft).
    
      
    
- **`Secure`:** Instructs the browser to transmit the cookie over encrypted HTTPS connections only.
    
      
    
- **`SameSite=Strict | Lax | None`:** Prevents cross-site request forgery (CSRF). `Lax` allows safe top-level navigations; `Strict` blocks all cross-site transfers.
    
      
    
- **`Domain` & `Path`:** Restricts the cookie scope to specific origins and URI subtrees.
    
      
    

### 3. OAuth 2.0 vs. OpenID Connect (OIDC) & OAuth 1.0

```
+-------------------------------------------------------------+
|                  OpenID Connect (OIDC)                      |
|  - Authentication (Who are you?)                            |
|  - Issues: id_token (JWT containing user profile & sub)     |
+-------------------------------------------------------------+
                              |
                              v  (Extends)
+-------------------------------------------------------------+
|                       OAuth 2.0                             |
|  - Delegated Authorization (What can the client access?)   |
|  - Issues: access_token & refresh_token                     |
|  - Scopes: read:profile, write:orders                       |
+-------------------------------------------------------------+
```

- **OAuth 1.0 vs. OAuth 2.0:**
    
      
    - _OAuth 1.0:_ Required custom cryptographic signatures (nonce, timestamps, HMAC secret keys) computed for every HTTP request. Extremely difficult for developers to implement; brittle.
        
          
        
    - _OAuth 2.0:_ Offloaded cryptographic transport complexity to **TLS/HTTPS**. Replaced complex signing with bearer tokens and introduced standard flows (notably Authorization Code with PKCE for SPAs and mobile apps).
        
          
        
- **Why OAuth 2.0 is NOT Authentication:** An `access_token` is an opaque artifact meant for an API resource server; it tells the API what scopes the bearer can execute. It conveys no standardized information about who logged in, when, or how. OIDC adds the `id_token` (formatted as a verifiable JWT) to standardize identity assertions.
    
      
    

### 4. Designing Permission-Based Access Control (PBAC / Granular RBAC)

> [!IMPORTANT] Core Architectural Rule
> 
> **Assign roles to users, but assign permissions to roles. Check permissions in code, never roles.**
> 
> Hardcoding `if (user.role === 'admin')` creates tight coupling. If product requirements introduce an `Auditor` or `Manager` role with partial administrative duties, you must track down and update every role check. Checking `if (hasPermission(user, 'invoice:delete'))` requires changing only the database role mapping.
> 
>   

#### Data Model (Relational Schema)

Plaintext

```
Users (1) <---> (N) UserRoles (N) <---> (1) Roles
                                              |
                                             (1)
                                              |
                                             (N)
                                        RolePermissions
                                             (N)
                                              |
                                             (1)
                                         Permissions (e.g., "orders:create", "orders:refund")
```

#### Frontend vs. Backend Enforcement

- **Backend (Source of Truth):**
    
      
    - Extract permissions during token generation and encode them in the JWT payload as a flat array: `permissions: ["posts:read", "posts:write"]`.
        
          
        
    - Use guards or middleware to intercept routes: `@RequirePermission('posts:delete')`.
        
          
        
- **Frontend (User Experience Only):**
    
      
    - The frontend checks permissions solely to toggle UI elements (render/disable buttons, hide menu links).
        
          
        
    - Never rely on the frontend for access security; clients can be manipulated.
        
          
        

## Code Snippets / Examples



```TypeScript
// ============================================================================
// 1. Permission-Based Access Control (PBAC) Engine
// ============================================================================

export type Permission = 
  | "users:read"
  | "users:write"
  | "users:delete"
  | "reports:export";

export interface AuthenticatedUser {
  id: string;
  roles: string[];
  permissions: Permission[]; // Flattened list of resolved permissions
}

// Higher-order authorization guard
export function hasPermission(user: AuthenticatedUser, required: Permission): boolean {
  return user.permissions.includes(required);
}

export function hasAllPermissions(user: AuthenticatedUser, required: Permission[]): boolean {
  return required.every((perm) => user.permissions.includes(perm));
}

// ============================================================================
// 2. Backend Middleware (e.g., Express / Node.js)
// ============================================================================
import { Request, Response, NextFunction } from "express";

export interface AuthenticatedRequest extends Request {
  user?: AuthenticatedUser;
}

export const requirePermission = (permission: Permission) => {
  return (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: "Unauthenticated" });
    }

    if (!hasPermission(req.user, permission)) {
      // 403 Forbidden: Client is authenticated, but lacks the necessary capability
      return res.status(403).json({ 
        error: "Forbidden", 
        missingPermission: permission 
      });
    }

    next();
  };
};

// Route usage
// app.delete('/api/users/:id', requirePermission('users:delete'), deleteUserHandler);

// ============================================================================
// 3. Frontend React Component (Permission Gate UI Pattern)
// ============================================================================
import React from "react";

interface CanProps {
  user: AuthenticatedUser;
  perform: Permission;
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

export const Can: React.FC<CanProps> = ({ user, perform, children, fallback = null }) => {
  return hasPermission(user, perform) ? <>{children}</> : <>{fallback}</>;
};

// UI Usage: Declarative and decoupled from roles
export const UserManagementRow = ({ currentUser, targetUser }: any) => {
  return (
    <div>
      <span>{targetUser.name}</span>
      
      {/* UI adapts based on granular capability, not role names */}
      <Can 
        user={currentUser} 
        perform="users:delete" 
        fallback={<span className="disabled-text">No delete permission</span>}
      >
        <button onClick={() => deleteUser(targetUser.id)}>Delete User</button>
      </Can>
    </div>
  );
};
```

## Related Topics


- [[Storage Strategies for Authorization Tokens. Access vs Refresh Tokens]]

- [[Web Security & Identity Architecture. SOP, XSS, CSRF & Token Lifecycles]]

- [[AWS Identity and Access Management (IAM) - Identities, Policies, Roles & Best Practices]]

- [[AWS API Gateway - Architecture, Security & Limitations]]

- [[Client-Side Browser Storage. Mechanisms, Architecture & Security]]


## Tags

#fullstack #interview #security #authentication #authorization #jwt #oauth #rbac

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups