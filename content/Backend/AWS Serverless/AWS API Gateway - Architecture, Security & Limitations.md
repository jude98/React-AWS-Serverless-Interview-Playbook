# AWS API Gateway: Architecture, Security & Limitations

## Key Concepts

- **API Gateway Role:** Fully managed reverse proxy and API management service providing entry points for client applications to access backend services (AWS Lambda, EC2, HTTP endpoints, or direct AWS service integrations).
    
      
    
- **Core Flavors:**
    
      
    - **REST APIs (v1):** Feature-rich; supports API keys, usage plans, request/response transformations (VTL mapping templates), caching, and custom authorizers.
        
          
        
    - **HTTP APIs (v2):** Lightweight, lower latency (~60% faster), cheaper (~70% lower cost), native OpenID Connect (OIDC) / JWT validation, but lacks request body transformations and response caching.
        
          
        
    - **WebSocket APIs:** Stateful, bi-directional persistent connections for real-time messaging.
        
          
        
- **Authorization Options:**
    
      
    - **Cognito User Pools Authorizer:** Managed OAuth2/OIDC token verification natively within the gateway without invoking Lambda compute.
        
          
        
    - **Lambda Authorizer:** Custom programmatic authorization returning dynamic IAM policies or simple allow/deny decisions.
        
          
        
    - **AWS IAM Authorization (`AWS_IAM`):** Signature Version 4 (SigV4) request signing; ideal for internal service-to-service or trusted backend callers.
        
          
        
- **Hard Architectural Constraints:** Fixed payload size limit of **10 MB** (request and response), and a maximum integration timeout limit of **29 seconds** (REST APIs default) or **30 seconds** (HTTP APIs).
    
      
    

## Common Interview Questions

- What are the architectural differences and selection criteria between REST APIs and HTTP APIs?
    
      
    
- How do Cognito User Pools Authorizers differ from custom Lambda Authorizers in performance and cost?
    
      
    
- When should you use `AWS_IAM` authentication over JWT/Bearer token authorization?
    
      
    
- How do you bypass API Gateway's 10 MB payload limit for large file uploads?
    
      
    
- How do you handle long-running operations when API Gateway times out at 29–30 seconds?
    
      
    
- How does API Gateway handle throttling, rate limiting, and DDoS mitigation?
    
      
    

## Strong Answers / Talking Points

### 1. Authentication & Authorization Comparison

|**Feature**|**Cognito Authorizer**|**Lambda Authorizer**|**IAM Authorization (AWS_IAM)**|
|---|---|---|---|
|**Mechanism**|Validates JWT (`id_token` or `access_token`) issued by Cognito User Pools.|Executes custom Lambda logic to inspect headers/tokens and yield an IAM Policy or simple Boolean.|Client signs the HTTP request using AWS credentials via SigV4.|
|**Latency / Cost**|Zero compute cost; fast, managed hardware/service-level evaluation.|Incurs Lambda execution duration and potential cold starts; mitigable via policy caching (TTL up to 3600s).|Zero compute cost; evaluated natively by AWS IAM infrastructure.|
|**Best For**|Standard B2C/B2B user authentication with OAuth2/OIDC claims.|Custom auth schemes (e.g., HMAC, legacy tokens, external providers like Auth0/Okta with custom claims).|Microservice-to-microservice, internal VPC workloads, or trusted AWS CLI/SDK clients.|

### 2. Hard Limitations & Workarounds

- **Payload Limit (10 MB):**
    
      
    - _Limitation:_ Request and response bodies cannot exceed 10 MB.
        
          
        
    - _Workaround:_ Use the **S3 Pre-signed URL pattern**. The client requests an upload URL via API Gateway $\rightarrow$ Lambda generates a pre-signed S3 URL $\rightarrow$ Client uploads directly to S3.
        
          
        
- **Integration Timeout (29–30 seconds):**
    
      
    - _Limitation:_ Default integration timeout is 29 seconds for REST APIs and 30 seconds for HTTP APIs.
        
          
        
    - _Workaround:_ Implement the **Asynchronous Polling (Job Queue) Pattern**. API Gateway triggers a Lambda that enqueues work to SQS and immediately returns HTTP `202 Accepted` with a `jobId`. A worker Lambda processes the queue asynchronously, and the client checks status via a separate `GET /jobs/{id}` route or WebSocket push.
        
          
        
- **Throttling & Quotas:**
    
      
    - Regional token-bucket algorithm: Standard limit is 10,000 requests per second (RPS) with a 5,000 burst quota per account (expandable).
        
          
        
    - Breaching limits triggers HTTP `429 Too Many Requests`.
        
          
        

## Code Snippets / Examples

### Custom Token Lambda Authorizer (TypeScript / Node.js)

```TypeScript
import { APIGatewayAuthorizerResult, APIGatewayTokenAuthorizerEvent } from "aws-lambda";
import * as jwt from "jsonwebtoken";

interface DecodedToken {
  sub: string;
  role: string;
}

export const handler = async (
  event: APIGatewayTokenAuthorizerEvent
): Promise<APIGatewayAuthorizerResult> => {
  const token = event.authorizationToken?.replace("Bearer ", "");

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as DecodedToken;
    const effect = decoded.role === "admin" ? "Allow" : "Deny";

    return {
      principalId: decoded.sub,
      policyDocument: {
        Version: "2012-10-17",
        Statement: [
          {
            Action: "execute-api:Invoke",
            Effect: effect,
            Resource: event.methodArn,
          },
        ],
      },
      // Pass contextual attributes to the backend Lambda integration
      context: {
        userId: decoded.sub,
        role: decoded.role,
      },
    };
  } catch {
    throw new Error("Unauthorized"); // Triggers HTTP 401
  }
};
```

### AWS SAM: REST API with Cognito Authorizer & IAM-Protected Route

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: API Gateway with Cognito and IAM Authorization

Resources:
  SecuredApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Auth:
        DefaultAuthorizer: CognitoAuth
        Authorizers:
          CognitoAuth:
            UserPoolArn: !GetAtt UserPool.Arn
            Identity:
              Header: Authorization

  # Public Route with Cognito Validation
  GetUserProfileFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: profile.handler
      Runtime: nodejs20.x
      Events:
        GetProfile:
          Type: Api
          Properties:
            RestApiId: !Ref SecuredApi
            Path: /profile
            Method: GET

  # Internal Microservice Route protected via IAM SigV4
  InternalAdminFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: admin.handler
      Runtime: nodejs20.x
      Events:
        AdminTask:
          Type: Api
          Properties:
            RestApiId: !Ref SecuredApi
            Path: /internal/sync
            Method: POST
            Auth:
              Authorizer: AWS_IAM

  UserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: production-users
```

## Related Topics

- [[AWS Lambda Core Architecture & Execution Model|AWS-Lambda-Core-Architecture]]
    
      
    
- [[Authentication, Authorization, OAuth OIDC & Permission-Based RBAC|Authentication-OAuth2-OIDC-and-JWT]]
    
      
    
- [[Authentication, Authorization, OAuth OIDC & Permission-Based RBAC|AWS-Cognito-User-Pools-vs-Identity-Pools]]
    
      
    
- [[AWS Identity and Access Management (IAM) - Identities, Policies, Roles & Best Practices|AWS-IAM-Policies-Roles-and-SigV4]]
    
      
    
- [[AWS Step Functions - Workflow Types, State Machine Patterns & Integration|Asynchronous-Workflows-with-SQS-and-Step-Functions]]
    
      
    

## Tags

#fullstack #interview #aws #api-gateway #cognito #security #iam

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups
    
      
    

To see a hands-on walkthrough showing how Cognito user pools plug directly into API Gateway routes to validate tokens, check out this guide on how to [authorize API calls with AWS Cognito and API Gateway](https://www.youtube.com/watch?v=N3FZWVF97n4&utm_source=gemini). It demonstrates configuring Cognito authorizers and handling authenticated user tokens.