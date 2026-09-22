
## Key Concepts

- **Serverless Definition:** Execution model where cloud providers provision, scale, and manage servers dynamically; billing is based strictly on resource consumption and execution duration (pay-per-request).
    
      
    
- **Core AWS Serverless Stack:** AWS Lambda (compute), API Gateway (ingress/routing), DynamoDB (managed NoSQL), Amazon S3 (object storage), Amazon EventBridge/SNS/SQS (messaging & integration).
    
      
    
- **Event-Driven Architecture (EDA):** An architectural paradigm where services communicate asynchronously by emitting, detecting, and reacting to state changes called **events**.
    
      
    
- **Why It's Called "Event-Driven":** Flow of control is dictated by state changes occurring in the domain (e.g., `OrderPlaced`, `PaymentFailed`), not by synchronous, top-down orchestration requests.
    
      
    
- **Producers vs. Consumers:** Producers publish events without knowing or caring who processes them (complete loose coupling); consumers subscribe and react independently.
    
      
    

## Common Interview Questions

- What is the difference between monolithic, standard microservices (REST/RPC), and event-driven architecture?
    
      
    
- Why choose AWS Serverless over long-running containers (e.g., ECS/EKS) for an event-driven system?
    
      
    
- How do two microservices communicate in an event-driven setup using AWS messaging services?
    
      
    
- When would you use Amazon SQS vs. Amazon SNS vs. Amazon EventBridge?
    
      
    
- How do you handle failure, idempotency, and out-of-order delivery in an event-driven architecture?
    
      
    
- What are the major trade-offs of event-driven architectures (e.g., debugging, eventual consistency)?
    
      
    

## Strong Answers / Talking Points

### 1. Serverless vs. Traditional Server Architectures

- **Talking Point:** Serverless shifts operational burden (patching, scaling, auto-provisioning) to AWS, enabling teams to focus on domain logic.
    
      
    
- **Trade-offs:**
    
      
    - _Pros:_ Auto-scaling to zero, low operational overhead, built-in high availability.
        
          
        
    - _Cons:_ Cold start latency, execution duration limits (Lambda 15-minute cap), vendor lock-in, complex distributed observability.
        
          
        

### 2. How Two Microservices Communicate via EDA (Example: Order Service → Inventory Service)

- **Step 1: Event Emission:** The Order Service processes an HTTP request, writes to its database, and emits an event (e.g., `OrderCreated`) to a central broker (EventBridge or SNS).
    
      
    
- **Step 2: Broker Ingestion & Routing:** The broker evaluates rules/filters and routes the event to targets (e.g., an SQS queue dedicated to the Inventory Service).
    
      
    
- **Step 3: Asynchronous Consumption:** Inventory Service's Lambda consumes batches from the SQS queue, reserves inventory, and acknowledges/deletes the message upon completion.
    
      
    
- **Key Advantage:** If the Inventory Service is down, messages buffer in SQS without cascading failure back to the Order Service or the end user.
    
      
    

### 3. Messaging Patterns: SQS vs. SNS vs. EventBridge

> [!NOTE]
> 
>   
> 
> - **SQS (Queueing):** 1-to-1 decoupling, message buffering, rate limiting, polling-based.
>     
>       
>     
> - **SNS (Pub/Sub):** 1-to-Many push notifications, high throughput, lightweight filtering.
>     
>       
>     
> - **EventBridge (Event Bus):** 1-to-Many smart routing, schema registry, native integration with AWS services and SaaS partners, advanced content-based filtering.
>     
>       
>     

### 4. Handling Distributed Failures & Idempotency

- **Dead Letter Queues (DLQ):** Unprocessable messages (poison pills) automatically divert to a DLQ after $N$ retry attempts.
    
      
    
- **Idempotency Requirement:** Network retries mean consumers must assume at-least-once delivery; enforce idempotency using unique transaction IDs or idempotency keys stored with conditional writes in DynamoDB/Redis.
    
      
    

## Code Snippets / Examples

### Producer: Emitting an Event to EventBridge (Node.js / AWS SDK v3)



```TypeScript
import { EventBridgeClient, PutEventsCommand } from "@aws-sdk/client-eventbridge";

const eventBridge = new EventBridgeClient({});

export async function publishOrderCreated(orderId: string, amount: number) {
  const command = new PutEventsCommand({
    Entries: [
      {
        Source: "service.orders",
        DetailType: "OrderCreated",
        Detail: JSON.stringify({ orderId, amount, timestamp: new Date().toISOString() }),
        EventBusName: "ecommerce-bus",
      },
    ],
  });

  await eventBridge.send(command);
}
```

### Consumer: Idempotent Lambda Handler processing SQS Events



```TypeScript
import { SQSEvent, SQSHandler } from "aws-lambda";

export const handler: SQSHandler = async (event: SQSEvent) => {
  for (const record of event.Records) {
    const { orderId, amount } = JSON.parse(record.body);
    
    // Idempotency check: Ensure order has not already been processed
    const alreadyProcessed = await checkIdempotencyRecord(record.messageId);
    if (alreadyProcessed) continue;

    await reserveInventory(orderId, amount);
    await markAsProcessed(record.messageId);
  }
};
```

## Related Topics

- [[Microservices-Architecture]]
    
      
    
- [[AWS-Lambda-and-Serverless-Patterns]]
    
      
    
- [[Message-Brokers-Kafka-vs-RabbitMQ-vs-SQS]]
    
      
    
- [[Distributed-Transactions-and-Saga-Pattern]]
    
      
    
- [[Idempotency-in-Distributed-Systems]]
    
      
    

## Tags

#fullstack #interview #aws #serverless #system-design #event-driven-architecture

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups