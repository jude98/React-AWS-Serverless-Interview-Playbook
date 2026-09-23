# Event-Driven Architecture Scenarios: Flash Sales, High-Scale Ordering & Extreme Inventory Contention

## Key Concepts

- Curated question bank covering ultra-high-concurrency order placement, distributed inventory locking, single-stock contention, and flash-sale traffic absorption.
    
      
    
- Evaluates patterns like asynchronous write queues, memory-tier reservations (Redis Lua), optimistic concurrency control, and transactional outbox.
    
      
    
- No answers included—structured strictly for mock interview practice, whiteboard drilling, and flashcard revision.
    
      
    

## Common Interview Questions

### Flash Sale Architecture & Extreme Ingestion Bursts

- During a flash sale where 500,000 users click "Buy Now" in the exact same second, how do you prevent the order-placement API from crashing the relational transactional database?
    
      
    
- Why is an asynchronous queue buffer (API Gateway $\to$ SQS/Kafka $\to$ Inventory Worker) superior to synchronous HTTP checkout calls during flash sales, and how do you inform the frontend whether the order succeeded or failed asynchronously (WebSockets, SSE, polling)?
    
      
    
- How do you design an edge queue / virtual waiting room (e.g., Cloudflare Waiting Room, AWS CloudFront + Lambda@Edge/KV) to meter traffic into the checkout flow before it hits backend compute?
    
      
    
- How do you prevent bot abuse and script-driven order placement during a flash sale without adding high-latency CAPTCHA steps that degrade legitimate customer checkout?
    
      
    

### High-Volume Order Processing Pipeline

- How do you design an end-to-end event-driven order processing pipeline across multiple microservices (`Checkout` $\to$ `Payment` $\to$ `Inventory` $\to$ `Shipping` $\to$ `Notification`) using the Saga pattern?
    
      
    
- What are the throughput limits and partitioning trade-offs when choosing between **Kafka partition keys**, **Kinesis partition keys**, and **SQS FIFO `MessageGroupId`** for maintaining per-user or per-order sequential integrity?
    
      
    
- How do you handle order cancellations or user timeouts while an order event is still in-flight inside an asynchronous queue buffer?
    
      
    
- How do you implement the **Transactional Outbox Pattern** to ensure that persisting the order to the primary database and publishing the `OrderCreated` event to the message bus succeed atomically without distributed 2PC locks?
    
      
    

### Limited-Offer & Extreme Inventory Contention (1-of-1 Item / Low-Stock Race)

- When only **one single item** is available (e.g., rare NFT, concert ticket front row, auction item, or single warehouse stock) and 100,000 concurrent requests compete for it, how do you guarantee that **exactly one** user gets it without creating race conditions, deadlocks, or database hot-partition crashes?
    
      
    
- Why does a standard relational database transaction (`SELECT ... FOR UPDATE` or row-level locking) collapse under 100k concurrent requests for the same row, and how does atomic memory-tier reservation via **Redis (Lua scripts / Redis transactions)** mitigate this?
    
      
    
- How do you implement a **Temporary Inventory Reservation (Hold)** with a TTL (e.g., hold stock for 10 minutes during payment checkout) and automatically release the stock if the user abandons checkout or payment fails:
    
      
    - Using DynamoDB TTL + DynamoDB Streams?
        
          
        
    - Using Redis Key Expiration (`EXPIRE` / Keyspace notifications)?
        
          
        
    - Using AWS Step Functions with a `Wait` state task token?
        
          
        
- If using DynamoDB for inventory tracking, how do you implement conditional updates (`ConditionExpression: stock > 0 AND attribute_exists(itemId)`) to prevent negative inventory?
    
      
    
- At what write throughput rate does DynamoDB reject conditional updates on a single partition key due to the hard 1,000 WCU limit, and how do you shard a low-count inventory item across multiple sub-counters?
    
      
    

### Payment vs. Order Inconsistencies & Reversals

- What happens if the customer's credit card is charged, but the inventory reservation service fails or exhausts its retry limits? How does the event-driven system trigger an automatic, idempotent refund (compensating transaction)?
    
      
    
- How do you handle the race condition where a payment provider's asynchronous webhook (`payment_intent.succeeded`) arrives **before** the internal order record has finished committing to the database?
    
      
    
- How do you detect and reconcile orphaned orders (e.g., orders stuck in `PENDING_PAYMENT` forever due to lost events or consumer crashes) using periodic reconciliation jobs and dead-letter queues?
    
      
    

### Read Heavy vs. Write Heavy: Real-Time Stock Availability

- During a high-scale sale, millions of users are refreshing product detail pages checking stock status while thousands are concurrently buying. How do you design the read path for real-time stock display without hammering the transactional inventory database?
    
      
    
- How do you balance the trade-off between **eventual consistency** (caching stock numbers in Redis/CDN with slight staleness) versus **overselling prevention** on the final write path?
    
      
    

## Strong Answers / Talking Points

- _Note: Reference individual topic notes for full technical breakdown and implementation architecture._
    
      
    - See [[Distributed Transactions & Event-Driven Architecture - Sagas, 2PC, Resilience & Messaging Selection|Distributed Transactions: 2PC vs Saga Pattern]] for order workflow orchestration and compensating refund steps.
        
          
        
    - See [[AWS SQS at Scale - High-Throughput Processing, Concurrency, and Backpressure|AWS SQS at Scale: High-Throughput Processing, Concurrency, and Backpressure]] for burst absorption during flash sales.
        
          
        
    - See [[Debugging DynamoDB Hot Partitions & Hot Keys]] for hot-spot mitigation and partition key salting.
        
          
        
    - See [[Distributed Transactions & Event-Driven Architecture - Sagas, 2PC, Resilience & Messaging Selection|Transactional Outbox Pattern with Debezium and DynamoDB Streams]] for dual-write avoidance between order state and event publishing.
        
          
        
    - See [[System Design Scenarios - Payment Workflows, Webhooks, Idempotency & Large S3 Payloads|Idempotency in Distributed Systems]] for handling duplicate order submissions.
        
          
        

## Code Snippets / Examples

### Atomic Single-Stock Inventory Claim via Redis Lua Script

Lua

```
-- KEYS[1]: Inventory key (e.g., "inventory:item:1001")
-- ARGV[1]: Quantity to reserve (e.g., 1)
-- ARGV[2]: Reservation hold key (e.g., "reservation:user:8821")
-- ARGV[3]: Hold TTL in seconds (e.g., 600)

local currentStock = tonumber(redis.call('GET', KEYS[1]) or "0")
local requestedQuantity = tonumber(ARGV[1])

if currentStock >= requestedQuantity then
    -- Atomically decrement inventory
    redis.call('DECRBY', KEYS[1], requestedQuantity)
    
    -- Record temporary hold with expiration
    redis.call('SET', ARGV[2], requestedQuantity, 'EX', tonumber(ARGV[3]))
    return 1 -- SUCCESS: Stock reserved
else
    return 0 -- OUT_OF_STOCK: Request rejected immediately without hitting DB
end
```

### DynamoDB Atomic Inventory Decrement with Conditional Check

```typescript
import { DynamoDBClient, UpdateItemCommand } from "@aws-sdk/client-dynamodb";

const ddb = new DynamoDBClient({ region: "us-east-1" });

export async function reserveInventory(itemId: string, quantityToBuy: number): Promise<boolean> {
  try {
    await ddb.send(new UpdateItemCommand({
      TableName: "InventoryTable",
      Key: {
        itemId: { S: itemId }
      },
      // Atomic decrement that enforces stock cannot drop below zero
      UpdateExpression: "SET currentStock = currentStock - :qty",
      ConditionExpression: "attribute_exists(itemId) AND currentStock >= :qty",
      ExpressionAttributeValues: {
        ":qty": { N: quantityToBuy.toString() }
      }
    }));
    return true;
  } catch (err: any) {
    if (err.name === "ConditionalCheckFailedException") {
      // Stock insufficient or item does not exist
      return false;
    }
    throw err;
  }
}
```

## Related Topics

- [[Distributed Transactions & Event-Driven Architecture - Sagas, 2PC, Resilience & Messaging Selection|Distributed Transactions: 2PC vs Saga Pattern]]
    
      
    
- [[AWS SQS at Scale - High-Throughput Processing, Concurrency, and Backpressure|AWS SQS at Scale: High-Throughput Processing, Concurrency, and Backpressure]]
    
      
    
- [[Distributed Transactions & Event-Driven Architecture - Sagas, 2PC, Resilience & Messaging Selection|Transactional Outbox Pattern with Debezium and DynamoDB Streams]]
    
      
    
- [[High-Volume Serverless Webhook Ingestion - WAF, API Gateway Direct SQS Integration, and Throttling|High-Volume Serverless Webhook Ingestion: WAF, API Gateway Direct SQS Integration, and Throttling]]
    
      
    
- [[Debugging DynamoDB Hot Partitions & Hot Keys]]
    
      
    
- [[System Design Scenarios - Payment Workflows, Webhooks, Idempotency & Large S3 Payloads|Idempotency in Distributed Systems]]
    
      
    

## Tags

#fullstack #interview #event-driven-architecture #system-design #flash-sale #inventory #redis #dynamodb #microservices

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups