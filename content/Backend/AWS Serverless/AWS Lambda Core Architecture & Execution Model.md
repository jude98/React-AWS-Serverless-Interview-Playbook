
## Key Concepts

- **Compute Model:** Event-driven, ephemeral compute running within lightweight Firecracker microVMs; scales horizontally by spinning up isolated execution environments per concurrent invocation.
    
      
    
- **Horizontal Scaling & One-at-a-Time Rule:** A single execution environment processes strictly **one request at a time**; $N$ concurrent requests spin up $N$ distinct execution environments.
    
      
    
- **Cold Start:** Latency incurred during initial environment spin-up: downloading code, starting the runtime/container, and executing initialization code outside the handler.
    
      
    
- **Resource Allocation Model:** CPU and network bandwidth scale linearly with configured RAM. Exactly **1,769 MB** yields **1 full vCPU**; **10,240 MB (10 GB)** yields **6 vCPUs**.
    
      
    
- **Billing Model:** Pay-as-you-go based on total invocations ($0.20 per 1M requests) and execution duration measured in gigabyte-seconds (GB-s), billed at 1 ms granularity.
    
      
    

## Common Interview Questions

- How does AWS Lambda achieve horizontal scaling, and what does "one invocation per container" mean for shared state?
    
      
    
- Walk me through the complete execution context lifecycle (`Init`, `Invoke`, `Shutdown`).
    
      
    
- What is a cold start, and how do Provisioned Concurrency vs. SnapStart eliminate/reduce it?
    
      
    
- How does AWS Lambda allocate CPU resources, and why is 1,769 MB considered a magic threshold?
    
      
    
- What are the package size limits for ZIP deployments vs. OCI Container Images, and how do you deploy large bundles?
    
      
    
- What are Lambda Layers, and when would you use an API Gateway Lambda Authorizer?
    
      
    
- What happens when your account reaches its regional concurrency limit (e.g., 1,000 concurrency)?
    
      
    

## Strong Answers / Talking Points

### 1. Execution Context Lifecycle (`Init` → `Invoke` → `Shutdown`)

> [!NOTE]
> 
> The execution environment lifecycle consists of three distinct phases managed by AWS:
> 
>   

1. **Init Phase:**
    
      
    - Downloads function code/container layer from S3/ECR.
        
          
        
    - Boots runtime and runs static/global initialization code outside the handler (DB connections, SDK client initialization).
        
          
        
    - If Init takes longer than 10 seconds, it restarts or times out.
        
          
        
2. **Invoke Phase:**
    
      
    - Runs the handler code with the event payload.
        
          
        
    - Environment freezes immediately after completion until another request arrives.
        
          
        
    - If a warm environment is reused, the Init phase is skipped entirely (warm start).
        
          
        
3. **Shutdown Phase:**
    
      
    - Triggered if no events arrive for a period (idle ~5-15 mins) or during scaling down.
        
          
        
    - Lambda sends a `SIGTERM` signal to the runtime and allows up to 2 seconds (or configured extension duration) for runtime shutdown hooks and cleanup before `SIGKILL`.
        
          
        

### 2. Concurrency, Limits, and Throttling

- **Default Limit:** 1,000 concurrent executions per region across an AWS account (soft limit, extendable).
    
      
    
- **Reserved Concurrency:** Guarantees a function a dedicated slice of capacity while preventing it from exhausting the unreserved account pool (also acts as a circuit breaker/rate limiter).
    
      
    
- **Throttling:** When incoming requests exceed available concurrency:
    
      
    - _Synchronous Invocations:_ Immediately return HTTP `429 Too Many Requests`.
        
          
        
    - _Asynchronous Invocations:_ Lambda retries with exponential backoff for up to 6 hours or pushes to a Dead Letter Queue (DLQ).
        
          
        

### 3. Mitigating Cold Starts: SnapStart vs. Provisioned Concurrency

|**Feature**|**SnapStart**|**Provisioned Concurrency**|
|---|---|---|
|**How it Works**|Takes a Firecracker microVM memory/disk snapshot at initialization time; restores from snapshot on invocation.|Pre-warms and holds allocated execution environments running continuously 24/7.|
|**Supported Runtimes**|Java, Python, .NET (managed runtimes).|All runtimes and custom container images.|
|**Cost**|Free (no extra charge; standard execution pricing applies).|Continuous hourly charge for allocated capacity + invocation costs.|
|**Best For**|Burst-heavy traffic on supported runtimes without ongoing idle cost.|Ultra-low latency SLA requirements across any language/container.|

### 4. Memory, CPU Allocation & Pricing

- **Memory Range:** 128 MB to 10,240 MB (1-MB increments).
    
      
    
- **vCPU Allocation:**
    
      
    - $128 \text{ MB} \approx 0.07 \text{ vCPU}$.
        
          
        
    - $1,769 \text{ MB} = 1.0 \text{ vCPU}$ (multithreading becomes useful beyond this threshold).
        
          
        
    - $10,240 \text{ MB} = 6 \text{ vCPUs}$.
        
          
        
- **Pricing Formulation:**
    
      
    
    $$\text{Total Cost} = (\text{Requests} \times \$0.20 / 10^6) + (\text{Allocated GB} \times \text{Duration in Seconds} \times \text{GB-s Rate})$$
    
    _(Note: Billed at 1 ms granularity; configuring higher memory often reduces duration, potentially yielding equal or lower net costs for compute-heavy tasks via AWS Lambda Power Tuning)._
    
      
    

### 5. Packaging Limits & Container Image Deployment

- **ZIP File Direct Upload:** 50 MB (compressed).
    
      
    
- **ZIP File from S3 / Unzipped Code:** 250 MB total (including layers).
    
      
    
- **Lambda Layers:** Up to 5 layers per function; total unzipped size must remain $\le 250\text{ MB}$. Used for sharing heavy dependencies (e.g., NumPy, common shared SDKs).
    
      
    
- **Container Images (OCI/Docker):** Up to **10 GB** pushed to Amazon Elastic Container Registry (ECR). Ideal for machine learning dependencies, headless Chromium, or large legacy binaries.
    
      
    

### 6. Lambda Authorizers

- An API Gateway plugin executing a Lambda function to control API access.
    
      
    
- **Token-based (`TOKEN`):** Receives a bearer token, verifies it (e.g., validates JWT signature), and returns an IAM policy.
    
      
    
- **Request-parameter-based (`REQUEST`):** Receives headers, query strings, and stage variables to construct dynamic allow/deny policies. Supports built-in policy caching (TTL up to 3600 seconds) to minimize authorization overhead.
    
      
    

## Code Snippets / Examples

### Reusing Execution Context for Warm Starts (TypeScript)


```TypeScript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, GetCommand } from "@aws-sdk/lib-dynamodb";

// INITIALIZATION PHASE: Executed once during cold start, reused across warm invocations
const ddbClient = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(ddbClient);

export const handler = async (event: { userId: string }) => {
  // INVOKE PHASE: Only the handler logic executes on warm invocations
  const command = new GetCommand({
    TableName: process.env.TABLE_NAME,
    Key: { userId: event.userId },
  });

  const response = await docClient.send(command);
  return {
    statusCode: 200,
    body: JSON.stringify(response.Item),
  };
};
```

### AWS SAM Template: Container Image, Layers & SnapStart Configuration

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Production Lambda configurations with SAM

Resources:
  # Pattern 1: Standard ZIP Function with SnapStart and Layer
  StandardZipFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: app.lambda_handler
      Runtime: python3.12
      MemorySize: 1769 # Exactly 1 vCPU
      Timeout: 15
      SnapStart:
        ApplyOn: PublishedVersions
      Layers:
        - !Ref SharedUtilitiesLayer
      AutoPublishAlias: live

  # Shared Dependencies Layer (Max 5 per function, 250MB total)
  SharedUtilitiesLayer:
    Type: AWS::Serverless::LayerVersion
    Properties:
      LayerName: shared-dependencies
      ContentUri: layers/dependencies/
      CompatibleRuntimes:
        - python3.12

  # Pattern 2: Large Bundle via OCI Container Image (Up to 10GB from ECR)
  LargeContainerFunction:
    Type: AWS::Serverless::Function
    Properties:
      PackageType: Image
      MemorySize: 3072
      Timeout: 30
    Metadata:
      DockerTag: node-v1
      DockerContext: ./docker-src
      Dockerfile: Dockerfile
```

## Related Topics

- [[AWS-Serverless-and-Event-Driven-Architecture]]
    
      
    
- [[API-Gateway-and-Authentication-Strategies]]
    
      
    
- [[Containerization-Docker-and-ECR]]
    
      
    
- [[Distributed-System-Performance-Optimization]]
    
      
    
- [[Infrastructure-as-Code-AWS-SAM-and-CloudFormation]]
    
      
    

## Tags

#fullstack #interview #aws #lambda #serverless #cloud-architecture #sam

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups