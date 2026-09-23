# Distributed Transactions & Event-Driven Architecture: Sagas, 2PC, Resilience & Messaging Selection

## Key Concepts

- Curated question bank covering distributed transaction coordination, failure compensation, event-driven resilience, circuit breaker integration, and AWS messaging topology selection.
    
      
    
- Evaluates trade-offs between ACID strong consistency (Two-Phase Commit) and BASE eventual consistency (Sagas).
    
      
    
- No answers included—structured strictly for mock interview practice, whiteboard drilling, and flashcard revision.
    
      
    

## Common Interview Questions

### Distributed Transactions: Saga vs. Two-Phase Commit (2PC)

- What is the fundamental mechanism of the Two-Phase Commit (2PC) protocol (Prepare Phase vs. Commit Phase), and why does it fail to scale in modern, cloud-native microservices architectures?
    
      
    
- How does the coordinator failure scenario in 2PC cause blocking distributed locks, and why is this unacceptable for high-availability systems?
    
      
    
- What is the Saga Pattern, and how does it replace atomic distributed locks with a sequence of local transactions and compensating transactions?
    
      
    
- In a Saga, how do you categorize steps into **Compensable Transactions**, **Pivot Transactions**, and **Retryable Transactions**?
    
      
    
- What are the architectural differences between **Choreography-based Sagas** (decentralized events via EventBridge/Kafka) and **Orchestration-based Sagas** (centralized workflows via AWS Step Functions)?
    
      
    
- What are the major data anomalies introduced by Sagas (e.g., dirty reads, non-repeatable reads, lost updates) due to the lack of the "Isolation" property in ACID, and how do you mitigate them with semantic locks or versioning?
    
      
    
- What happens if a **compensating transaction fails** midway through rolling back a Saga? How does the system recover without human intervention?
    
      
    

### Event-Driven Retries, Backoff & Error Classification

- How do you distinguish between **transient faults** (network glitches, rate limits, 503s) and **poison pill / non-retryable faults** (malformed JSON, 400 Bad Request, invalid business logic) in an event consumer?
    
      
    
- Why is simple linear retry considered dangerous in event-driven systems, and how does **exponential backoff with full jitter** prevent the "thundering herd" problem?
    
      
    
- What are the mechanics of Dead Letter Queues (DLQ) in event-driven pipelines, and how do you design an automated, self-healing redrive strategy for transient outages?
    
      
    
- How do you implement the **Retry Queue Pattern** (e.g., Main Queue $\to$ Retry Queue with delay $\to$ DLQ) in systems where the message broker does not have native per-message backoff?
    
      
    

### Circuit Breakers in Asynchronous & Event-Driven Architectures

- The Circuit Breaker pattern was originally designed for synchronous HTTP/RPC calls. How does a Circuit Breaker function in an **asynchronous event-driven architecture**?
    
      
    
- When downstream dependencies (e.g., a third-party payment gateway or relational DB) fail, how does an event-driven circuit breaker throttle or halt consumption:
    
      
    - Inside a Kafka consumer group (pausing topic partitions vs. stopping the consumer)?
        
          
        
    - Inside an SQS/Lambda Event Source Mapping (scaling concurrency to 0 vs. setting max receive count)?
        
          
        
- How do you manage and share circuit breaker states (`CLOSED`, `OPEN`, `HALF-OPEN`) across distributed, stateless serverless worker instances using Redis or DynamoDB?
    
      
    
- What is the difference between a client-side circuit breaker (rejecting outbound calls) and an event source circuit breaker (pausing message ingestion)?
    
      
    

### AWS Messaging Selection: Step Functions vs. SQS vs. EventBridge

- When should you use **AWS Step Functions** instead of **Amazon EventBridge** to coordinate a business workflow across microservices?
    
      
    
- When do you use **Amazon SQS** as an intermediate buffer behind an **EventBridge Rule**, rather than targeting compute (Lambda, ECS) directly?
    
      
    
- How do you choose between **Amazon SNS** and **Amazon EventBridge** when designing an event fan-out pipeline?
    
      
    
- What are the latency, throughput, and pricing trade-offs between Step Functions Express Workflows, Standard Workflows, and EventBridge buses?
    
      
    
- In what scenarios would an architecture combine all three (EventBridge for choreography $\to$ SQS for buffering $\to$ Step Functions for local saga orchestration)?
    
      
    

### Microservice Communication Protocols & Topologies

- What are the architectural trade-offs between **Synchronous Request-Response** (REST / gRPC), **Asynchronous Messaging** (SQS / RabbitMQ), and **Event Streaming** (Kafka / Kinesis)?
    
      
    
- What is the difference between **Event Notification**, **Event-Carried State Transfer**, and **Event Sourcing**?
    
      
    
- How does the **Transactional Outbox Pattern** ensure that a database mutation and an outbound event publication succeed or fail atomically without using 2PC?
    
      
    
- How do you prevent event storming or circular dependency loops when multiple microservices publish and subscribe to related domain events on a shared bus?
    
      
    

## Strong Answers / Talking Points

- _Note: Reference individual topic notes for full technical breakdown and implementation architecture._
    
      
    - See [[Distributed Transactions & Event-Driven Architecture - Sagas, 2PC, Resilience & Messaging Selection|Distributed Transactions: 2PC vs Saga Pattern]] for comparison tables, isolation mitigation, and pivot step modeling.
        
          
        
    - See [[AWS Step Functions - Workflow Types, State Machine Patterns & Integration|AWS Step Functions: Orchestration vs Choreography]] for Step Functions state machine implementations of Sagas.
        
          
        
    - See [[AWS Serverless Interview Scenarios - Advanced System Design & Debugging|Serverless Circuit Breakers: DynamoDB and Redis Implementations]] for distributed state synchronization (`CLOSED`, `OPEN`, `HALF-OPEN`).
        
          
        
    - See [[AWS SNS vs. Amazon EventBridge - Architecture, Differences, and Combined Patterns|AWS SNS vs. Amazon EventBridge: Architecture, Differences, and Combined Patterns]] for routing vs. pub/sub distinctions.
        
          
        
    - See [[Distributed Transactions & Event-Driven Architecture - Sagas, 2PC, Resilience & Messaging Selection|Transactional Outbox Pattern with Debezium and DynamoDB Streams]] for atomic event publishing without dual-writes.
        
          
        

## Code Snippets / Examples

### Distributed Circuit Breaker State Transition Handler (TypeScript / Redis)

```typescript
import { Redis } from "ioredis";

export enum CircuitState {
  CLOSED = "CLOSED",
  OPEN = "OPEN",
  HALF_OPEN = "HALF_OPEN"
}

export class DistributedCircuitBreaker {
  private redis: Redis;
  private serviceKey: string;
  private failureThreshold: number;
  private coolDownPeriodMs: number;

  constructor(serviceName: string, redisClient: Redis, threshold = 5, coolDownMs = 30000) {
    this.serviceKey = `circuit:${serviceName}`;
    this.redis = redisClient;
    this.failureThreshold = threshold;
    this.coolDownPeriodMs = coolDownMs;
  }

  async canExecute(): Promise<boolean> {
    const state = await this.redis.get(`${this.serviceKey}:state`) || CircuitState.CLOSED;

    if (state === CircuitState.OPEN) {
      const openSince = Number(await this.redis.get(`${this.serviceKey}:open_since`));
      if (Date.now() - openSince > this.coolDownPeriodMs) {
        // Transition to HALF_OPEN to probe downstream health
        await this.redis.set(`${this.serviceKey}:state`, CircuitState.HALF_OPEN);
        return true;
      }
      return false; // Fast-fail without attempting call
    }

    return true; // CLOSED or HALF_OPEN can proceed
  }

  async recordSuccess(): Promise<void> {
    // Reset counters and close circuit
    await this.redis.del(`${this.serviceKey}:failures`);
    await this.redis.set(`${this.serviceKey}:state`, CircuitState.CLOSED);
  }

  async recordFailure(): Promise<void> {
    const failures = await this.redis.incr(`${this.serviceKey}:failures`);
    
    if (failures >= this.failureThreshold) {
      // Trip the breaker to OPEN
      await this.redis.set(`${this.serviceKey}:state`, CircuitState.OPEN);
      await this.redis.set(`${this.serviceKey}:open_since`, Date.now().toString());
    }
  }
}
```

## Related Topics


- [[AWS Step Functions - Workflow Types, State Machine Patterns & Integration]]

- [[AWS Serverless & Event-Driven Architecture (EDA)]]

- [[Event-Driven Architecture Scenarios - Flash Sales, High-Scale Ordering & Extreme Inventory Contention]]

- [[Scaling DynamoDB Streams - High-Volume Event Processing]]

- [[CAP Theorem]]


## Tags

#fullstack #interview #distributed-systems #saga #microservices #eventbridge #step-functions #circuit-breaker #system-design

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups