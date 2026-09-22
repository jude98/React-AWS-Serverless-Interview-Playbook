

## Key Concepts

- Curated architectural interview scenarios covering mission-critical payment workflows, distributed financial consistency, and streaming large-file handling.
    
      
    
- Focuses on failure recovery, dual-write prevention, distributed transactions, asynchronous state reconciliation, and memory bounds in serverless architectures.
    
      
    
- No answers included—structured strictly for mock interview practice and flashcard drilling.
    
      
    

## Common Interview Questions

### Payment Infrastructure & Partial Failure Recovery

- In a microservices payment flow (`Order Service` $\to$ `Payment Gateway` $\to$ `Ledger Service` $\to$ `Inventory Service`), the external gateway charges the credit card successfully, but the network drops before your application receives the confirmation. How do you prevent double-charging while reconciling internal system state?
    
      
    
- How do you design a payment workflow using the **Saga Pattern** (Choreography vs. Orchestration with AWS Step Functions), and how are compensating transactions executed when a step fails mid-pipeline (e.g., payment succeeds, but inventory reservation fails)?
    
      
    
- How do you model two-phase financial money movements (e.g., authorization vs. capture, holds, and escrow release) using an event-driven immutable double-entry ledger?
    
      
    
- What is the Outbox Pattern, and how does it prevent the distributed dual-write problem between writing order state to a transactional database and publishing a `PaymentInitiated` event to EventBridge or Kafka?
    
      
    
- How do you implement asynchronous bank reconciliation batches (e.g., end-of-day ACH/SEPA/NACHA settlements) that match ingested third-party CSV files against internal database transactions?
    
      
    

### Payment Event-Driven Architecture (EDA) & Webhooks

- External payment providers (e.g., Stripe, Adyen, PayPal) deliver webhooks out of order (e.g., `payment_intent.succeeded` arrives before `payment_intent.created`). How do you architect the ingestion pipeline to guarantee deterministic state transitions?
    
      
    
- How do you secure public webhook endpoints against replay attacks and forgery without degrading throughput (evaluating HMAC signatures, timestamp drift validation, and replay prevention via SQS/DynamoDB TTL)?
    
      
    
- When third-party webhook delivery encounters downstream database unavailability, what HTTP response codes ($200, 202, 429, 503$) should API Gateway emit, and how do you leverage provider-side exponential backoff policies?
    
      
    
- How do you implement an outgoing webhook dispatch engine (delivering real-time events to your own B2B customers) with per-tenant rate limits, circuit breakers for unresponsive customer endpoints, and dead-letter archival?
    
      
    

### Idempotency in Serverless Handlers

- SQS delivers messages with "at-least-once" guarantees, leading to duplicate invocations. How do you implement a robust idempotency layer in AWS Lambda using DynamoDB conditional writes (`attribute_not_exists`)?
    
      
    
- How do you handle race conditions when two identical payment requests hit concurrent Lambda instances simultaneously within milliseconds of each other (the `IN_PROGRESS` locking state vs. `COMPLETED` cache return)?
    
      
    
- What happens if a Lambda function acquires an idempotency lock, marks the transaction `IN_PROGRESS`, and crashes due to an Out-Of-Memory (OOM) error or timeout? How do you handle lock expiration without causing duplicate billing?
    
      
    
- What payload hashing strategies (e.g., deterministic JSON canonicalization, parameter sorting, volatile header exclusion) ensure an `Idempotency-Key` correctly detects duplicate intents versus payload mutations?
    
      
    

### Processing & Streaming Large Files with S3 & Lambda

- AWS Lambda has a strict 6 MB request/response payload limit for synchronous invocations and an ephemeral `/tmp` disk limit (512 MB to 10 GB). How do you upload and process a 50 GB CSV or video file using S3 and Lambda without running out of disk or memory?
    
      
    
- How does the **S3 Multipart Upload** lifecycle work when paired with **Pre-signed URLs**, and how does the frontend upload chunks in parallel directly to S3 without streaming bytes through backend servers?
    
      
    
- How do you use Node.js streams (`stream.pipeline`, `Transform`, `csv-parser`) or Python generators with the AWS SDK to parse a multi-gigabyte S3 object on the fly using S3 chunked byte-range fetches (`Range: bytes=0-10485760`)?
    
      
    
- What are the architectural differences, latency benchmarks, and cost profiles between querying large datasets using **S3 Select**, **Amazon Athena**, and streaming through Lambda?
    
      
    
- How do you handle incomplete, orphaned multipart uploads in S3 to prevent escalating storage bills (S3 Lifecycle policies for `AbortIncompleteMultipartUpload`)?
    
      
    

## Strong Answers / Talking Points

- _Note: Reference individual topic notes for full technical breakdown and implementation architecture._
    
      
    - See [[Idempotency in Distributed Systems]] and AWS Lambda Powertools Idempotency persistence layers for state-machine locking (`IN_PROGRESS`, `COMPLETE`).
        
          
        
    - See [[High-Volume Serverless Webhook Ingestion: WAF, API Gateway Direct SQS Integration, and Throttling]] for decoupled webhook ingestion.
        
          
        
    - See [[AWS Step Functions: Orchestration vs Choreography]] for Saga pattern and compensating transactions.
        
          
        
    - See [[Transactional Outbox Pattern with Debezium and DynamoDB Streams]] for dual-write mitigation.
        
          
        
    - See [[S3 Multipart Upload Architecture with Presigned URLs]] for client-to-storage direct uploads.
        
          
        

## Code Snippets / Examples

### DynamoDB-Backed Idempotent Execution Lock Pattern

TypeScript

```
import { DynamoDBClient, PutItemCommand, UpdateItemCommand } from "@aws-sdk/client-dynamodb";

const ddb = new DynamoDBClient({ region: "us-east-1" });
const TABLE_NAME = "IdempotencyStore";

export async function processIdempotentTransaction<T>(
  idempotencyKey: string,
  ttlSeconds: number,
  work: () => Promise<T>
): Promise<T> {
  const now = Math.floor(Date.now() / 1000);
  const expiresAt = now + ttlSeconds;

  // 1. Atomically acquire lock
  try {
    await ddb.send(new PutItemCommand({
      TableName: TABLE_NAME,
      Item: {
        id: { S: idempotencyKey },
        status: { S: "IN_PROGRESS" },
        createdAt: { N: now.toString() },
        ttl: { N: expiresAt.toString() },
      },
      ConditionExpression: "attribute_not_exists(id) OR (#status = :in_progress AND #ttl < :now)",
      ExpressionAttributeNames: {
        "#status": "status",
        "#ttl": "ttl"
      },
      ExpressionAttributeValues: {
        ":in_progress": { S: "IN_PROGRESS" },
        ":now": { N: now.toString() }
      }
    }));
  } catch (err: any) {
    if (err.name === "ConditionalCheckFailedException") {
      throw new Error(`Conflict: Transaction ${idempotencyKey} is already in progress or completed.`);
    }
    throw err;
  }

  // 2. Execute business critical payload
  let result: T;
  try {
    result = await work();
  } catch (executionError) {
    // Release or mark failed on unexpected runtime error
    await ddb.send(new UpdateItemCommand({
      TableName: TABLE_NAME,
      Key: { id: { S: idempotencyKey } },
      UpdateExpression: "SET #status = :failed",
      ExpressionAttributeNames: { "#status": "status" },
      ExpressionAttributeValues: { ":failed": { S: "FAILED" } }
    }));
    throw executionError;
  }

  // 3. Mark complete and store cached response
  await ddb.send(new UpdateItemCommand({
    TableName: TABLE_NAME,
    Key: { id: { S: idempotencyKey } },
    UpdateExpression: "SET #status = :completed, #response = :res",
    ExpressionAttributeNames: {
      "#status": "status",
      "#response": "responseData"
    },
    ExpressionAttributeValues: {
      ":completed": { S: "COMPLETED" },
      ":res": { S: JSON.stringify(result) }
    }
  }));

  return result;
}
```

## Related Topics

- [[AWS SQS at Scale: High-Throughput Processing, Concurrency, and Backpressure]]
    
      
    
- [[High-Volume Serverless Webhook Ingestion: WAF, API Gateway Direct SQS Integration, and Throttling]]
    
      
    
- [[DynamoDB Conditional Writes and Optimistic Locking]]
    
      
    
- [[Distributed Transactions: 2PC vs Saga Pattern]]
    
      
    
- [[AWS Lambda Memory Sizing, Disk Limits, and Streaming Responses]]
    
      
    

## Tags

#fullstack #interview #aws #payments #idempotency #webhooks #s3 #system-design #distributed-systems

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups