# AWS Serverless Interview Scenarios: Advanced System Design & Debugging

## Key Concepts

- Curated question bank covering production edge cases across API Gateway, Lambda, DynamoDB, SQS, Step Functions, and EventBridge.
    
      
    
- Focuses on latency breakdown, distributed failure modes, cross-boundary IAM, circuit breakers, and stateful concurrency under high load.
    
      
    
- No answers included—structured strictly for mock interview practice and flashcard drilling.
    
      
    

## Common Interview Questions

### API Gateway & Lambda Latency Profiling

- The client-side latency for an API endpoint is 4 seconds, but the backend Lambda function logs report a `Billed Duration` of only 150 ms. How do you profile and isolate the source of the remaining delay across:
    
      
    - DNS lookup and TLS handshakes
        
          
        
    - AWS WAF WebACL evaluation latency
        
          
        
    - API Gateway integration latency vs. response mapping
        
          
        
    - Lambda cold-start initialization phase (`Init Duration`)
        
          
        
    - VPC Elastic Network Interface (ENI) attachment
        
          
        
- Why might p50/p90 latency hover between 1–2 seconds while p95/p99 spikes to 10–20 seconds, and how do you determine if the cause is event loop starvation, garbage collection pauses, or database connection pool contention?
    
      
    
- How does API Gateway REST API compare with HTTP API regarding latency, feature parity, native payload caching, and request overhead?
    
      
    

### Data Consistency & Silent Failures

- A Lambda function returned a `200 OK` response to the client, but the corresponding update never appeared in DynamoDB. What runtime or design conditions cause this silent failure (e.g., unhandled promise rejections, non-awaited asynchronous calls, Node.js event-loop draining behavior when `context.callbackWaitsForEmptyEventLoop = false`, uncommitted transactions, or fire-and-forget logic)?
    
      
    
- How do you guarantee idempotency in a Lambda handler consuming duplicate events from SQS/API Gateway without introducing a race condition between concurrent executions?
    
      
    

### Cross-Account IAM & Access Revocation

- A Lambda function in Account A previously had access to an S3 bucket in Account B, but access was abruptly revoked. What are the specific layers to audit when debugging cross-account permission loss:
    
      
    - Caller IAM execution role and identity-based policies
        
          
        
    - Destination S3 bucket policy
        
          
        
    - AWS KMS key policy (and customer managed key vs. AWS managed key restrictions)
        
          
        
    - IAM Permission Boundaries and AWS Organizations Service Control Policies (SCPs)
        
          
        
    - Cross-account STS `AssumeRole` session policies vs. resource-based authorization
        
          
        
- How do you configure and execute cross-account DynamoDB access or cross-account EventBridge bus routing securely?
    
      
    

### Distributed Resilience, Circuit Breaking & Throttling

- How do you implement a distributed Circuit Breaker pattern across ephemeral, stateless Lambda instances without relying on in-memory instance state?
    
      
    
- When external dependency downtime triggers a circuit break, how do you manage state transitions (`CLOSED`, `OPEN`, `HALF-OPEN`) using DynamoDB or Redis?
    
      
    
- How does Lambda handle regional concurrency exhaustion, and what are the functional differences between `ReservedConcurrency` (limiting/reserving) and `ProvisionedConcurrency` (pre-warming)?
    
      
    

### Serverless Cost & Compute Optimization

- Under what architectural conditions does increasing Lambda allocated memory (e.g., from 512 MB to 1792 MB or 3008 MB) actually **reduce** total execution cost (the AWS Lambda Power Tuning concept)?
    
      
    
- How does execution duration billing scale relative to proportional vCPU allocation, and at what threshold does a function switch from single-threaded to multi-threaded CPU allocation?
    
      
    

### Deployment, Rollbacks & Infrastructure as Code (IaC)

- A CloudFormation stack deployment triggered on a Friday evening began failing in production. How do you configure automated rollbacks using CloudWatch Alarms and Rollback Triggers without manual operator intervention?
    
      
    
- What happens under the hood when a CloudFormation deployment fails after a Lambda function resource has already been modified? How does CloudFormation manage state restoration, and what triggers an `UPDATE_ROLLBACK_FAILED` state?
    
      
    
- How do you safely deploy a Global Secondary Index (GSI) on a multi-terabyte DynamoDB table via CloudFormation without timing out stack creation or throttling base table writes?
    
      
    
- How do you manage continuous integration and schema changes across multiple microservices without triggering breaking contract failures?
    
      
    

### Distributed Backpressure Across AWS Services

- How do you identify, absorb, and mitigate backpressure across different serverless transport layers:
    
      
    - Inside Amazon SQS (handling message depth, visibility timeout drift, and receiver batch sizing)?
        
          
        
    - Inside AWS Step Functions (Standard vs. Express execution throughput, task throttling, and execution backpressure)?
        
          
        
    - Inside Amazon EventBridge (handling bus throughput limits, API Destination throttling, and target-level dead lettering)?
        
          
        
- What is the difference between backpressure handling on push-based integrations (EventBridge $\to$ Lambda) versus pull-based integrations (SQS $\to$ Lambda ESM)?
    
      
    

## Strong Answers / Talking Points

- Reference individual system design notes for deep-dive implementations:
    
      
    - See [[AWS Lambda Core Architecture & Execution Model|AWS Lambda Execution Context and Lifecycle]] for p95/cold-start latency profiling.
        
          
        
    - See [[AWS Identity and Access Management (IAM) - Identities, Policies, Roles & Best Practices|AWS Cross-Account IAM and S3 Bucket Policies]] for cross-account boundary debugging.
        
          
        
    - See [[AWS Serverless Interview Scenarios - Advanced System Design & Debugging|Serverless Circuit Breakers: DynamoDB and Redis Implementations]] for distributed state management.
        
          
        
    - See [[AWS Lambda Core Architecture & Execution Model|AWS Lambda Power Tuning and Cost Optimization]] for vCPU vs. duration billing dynamics.
        
          
        
    - See [[AWS Serverless Deployments - CloudFormation, Lambda Versions, Aliases, and Safe Deployments|AWS Serverless Deployments: CloudFormation, Lambda Versions, Aliases, and Safe Deployments]] for automated rollback triggers.
        
          
        
    - See [[Handling Event Clogging and Backpressure in Amazon EventBridge]] and [[AWS SQS at Scale - High-Throughput Processing, Concurrency, and Backpressure|AWS SQS at Scale: High-Throughput Processing, Concurrency, and Backpressure]] for backpressure patterns.
        
          
        

## Code Snippets / Examples

### Exponential Backoff with Full Jitter

```typescript
/**
 * Executes an asynchronous operation with exponential backoff and full jitter.
 * Formula: sleep = random_between(0, min(cap, base * 2^attempt))
 */
export async function retryWithBackoff<T>(
  operation: () => Promise<T>,
  maxRetries: number = 5,
  baseDelayMs: number = 100,
  maxDelayMs: number = 3000
): Promise<T> {
  let attempt = 0;

  while (true) {
    try {
      return await operation();
    } catch (error) {
      attempt++;
      if (attempt > maxRetries) {
        throw error;
      }

      // Exponential component: base * 2^(attempt - 1)
      const calculatedDelay = Math.min(maxDelayMs, baseDelayMs * Math.pow(2, attempt - 1));
      
      // Full jitter: decorrelates concurrent retrying clients to prevent thundering herd
      const sleepDuration = Math.random() * calculatedDelay;

      await new Promise((resolve) => setTimeout(resolve, sleepDuration));
    }
  }
}
```

## Related Topics


- [[AWS Observability - CloudWatch, AWS X-Ray & CloudTrail]]

- [[AWS Lambda Core Architecture & Execution Model]]

- [[DynamoDB Single-Table Design - Inventory Management Scenario]]

- [[High-Volume Serverless Webhook Ingestion - WAF, API Gateway Direct SQS Integration, and Throttling]]

- [[AWS Serverless Deployments - CloudFormation, Lambda Versions, Aliases, and Safe Deployments]]


## Tags

#fullstack #interview #aws #lambda #api-gateway #cloudformation #dynamodb #system-design #distributed-systems

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups