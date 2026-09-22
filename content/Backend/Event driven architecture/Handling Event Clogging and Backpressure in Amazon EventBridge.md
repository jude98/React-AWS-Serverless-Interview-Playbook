
## Key Concepts

- **What is Event Clogging**: Occurs when ingestion velocity surpasses target processing capacity or when a slow/failing downstream target backs up delivery, exhausting service quotas (`PutEvents` throttling, concurrent rule invocation limits, or target throttling).
    
      
    
- **The "Square Fan-Out" (Matrix / Sharded Fan-Out) Pattern**: A multi-tier routing topology designed to avoid single-bus bottlenecking. An ingestion layer shards events across intermediate processing lanes ($N$ producers $\to M$ intermediate buses/queues $\to K$ workers), reducing fan-out density from $O(N \times K)$ to an $N \times M$ grid.
    
      
    
- **Target Buffering (Queue-as-Shock-Absorber)**: Never target high-volume consumers (Lambda, HTTP endpoints, Step Functions) directly from an EventBridge rule. Always place **Amazon SQS in front of the target** to absorb spikes and decouple push-based dispatch from pull-based consumption.
    
      
    
- **Rule & Target Limits**: An EventBridge rule supports a maximum of 5 targets. Attaching dozens of synchronous targets directly to a bus triggers rule quota limits and cascading retries.
    
      
    
- **Target-Level Dead Letter Queues (DLQ)**: Every rule target must configure an SQS DLQ. EventBridge retries failed deliveries for 24 hours (with exponential backoff); without a DLQ, undeliverable events are silently discarded after expiration.
    
      
    
- **Backpressure Mechanism**: EventBridge does not natively queue; it routes or drops/DLQs. True backpressure must be established by buffering with SQS and tuning Lambda Event Source Mapping (ESM) concurrency downstream.
    
      
    

## Common Interview Questions

- What causes event clogging in Amazon EventBridge, and how does the service behave when targets throttle or fail?
    
      
    
- What is the "Square Fan-Out" (tiered routing) pattern, and how does it prevent single-bus saturation?
    
      
    
- Why is targeting Lambda directly from EventBridge considered an antipattern for bursty or high-volume workloads?
    
      
    
- How does EventBridge handle retry policies, maximum event age, and dead-letter queues at the target level?
    
      
    
- How do you isolate "noisy neighbor" event sources from blocking mission-critical events on a shared custom bus?
    
      
    
- How do you implement backpressure when the final consumer writes to a strictly rate-limited third-party API?
    
      
    

## Strong Answers / Talking Points

### 1. Root Causes of Event Clogging

- **Direct Compute Saturation**: If an EventBridge rule invokes a Lambda function directly, a sudden burst of 10,000 events/sec will immediately exhaust regional Lambda concurrency limits (default pool 1,000), causing cascade throttling across unrelated systems.
    
      
    
- **Downstream Target Latency**: When targeting third-party webhooks via EventBridge API Destinations, slow response times (> 5s) exhaust connection pools and trigger delivery retries, choking the EventBridge retry pipeline.
    
      
    
- **Noisy Neighbor on a Monolithic Bus**: Consolidating every business event onto a single custom bus allows high-frequency telemetry/audit events to consume default `PutEvents` TPS quotas, starving critical domain events.
    
      
    

### 2. The Solution: The "Square Fan-Out" (Tiered Bus / Sharded Queue) Architecture

When an architecture faces exponential fan-out ($N$ event producers emitting events consumed by $K$ different systems), connecting everything to a single bus causes rule bloat and throughput contention.

  

- **How Square Fan-Out Works**:
    
      
    1. **Tier 1 (Root Ingestion Buses / Shards)**: Producers emit to partitioned domain buses (or an initial SNS/Kinesis shard array) based on hash or domain partition (e.g., `Bus_A`, `Bus_B`).
        
          
        
    2. **Tier 2 (Router / Dispatcher Layer)**: Coarse-grained routing rules fan out events to intermediate processing lanes (an $M \times M$ grid or "square" topology).
        
          
        
    3. **Tier 3 (Queue Buffering Layer)**: Each individual subscriber owns its own dedicated Amazon SQS queue target.
        
          
        
    4. **Tier 4 (Throttled Workers)**: Consumer Lambdas or ECS tasks pull from their dedicated queue using bounded concurrency (`MaximumConcurrency` on ESM).
        
          
        
- **Result**: Eliminates $O(N \times K)$ cross-dependencies. Isolates failures so that a slow consumer on lane $B$ never creates backpressure on lane $A$.
    
      
    

### 3. Buffering & Rate Limiting via SQS Shock Absorbers

- **Never Route Bus $\to$ Lambda**:
    
      
    
    $$\text{EventBridge} \xrightarrow{\text{Rule}} \text{Amazon SQS} \xrightarrow{\text{ESM (Concurrency Capped)}} \text{Lambda}$$
    
- **Decoupling Rate**: If 50,000 events hit the bus in 2 seconds, SQS absorbs 100% of the burst instantly.
    
      
    
- **Controlled Drain**: The Lambda ESM drains the queue at a deterministic rate (e.g., `MaximumConcurrency: 20`, `BatchSize: 10`), shielding downstream relational databases or external APIs from being overwhelmed.
    
      
    

### 4. API Destinations: Native EventBridge Rate Limiting

- When forwarding events to external partner webhooks (e.g., Stripe, Shopify, internal legacy REST endpoints), use **EventBridge API Destinations**.
    
      
    
- API Destinations have a built-in **Rate Limiting** configuration (invocations per second, from 1 to 10,000+).
    
      
    
- If traffic exceeds the rate limit, EventBridge holds the events in an internal buffer for up to 24 hours, automatically providing backpressure without deploying custom queue/poller infrastructure.
    
      
    

## Code Snippets / Examples

### AWS SAM: Clogging-Resistant Architecture (Bus $\to$ SQS Buffer $\to$ Throttled ESM + DLQ)

YAML

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Production anti-clogging pattern using SQS buffering, target DLQ, and bounded ESM.

Resources:
  CoreEventBus:
    Type: AWS::Events::EventBus
    Properties:
      Name: core-domain-bus

  # 1. Target Dead Letter Queue (Catches failed deliveries from EventBridge)
  TargetDeliveryDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: eb-target-delivery-dlq
      MessageRetentionPeriod: 1209600 # 14 days

  # 2. Ingestion Shock-Absorber Queue (Buffers bursts to prevent compute clogging)
  ConsumerBufferQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: order-processing-buffer
      VisibilityTimeout: 120
      ReceiveMessageWaitTimeSeconds: 20

  # Allow EventBridge to write to the Buffer Queue
  QueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref ConsumerBufferQueue
      PolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Service: events.amazonaws.com
            Action: sqs:SendMessage
            Resource: !GetAtt ConsumerBufferQueue.Arn

  # 3. Rule targeting the Buffer Queue with Retry and DLQ Policies
  OrderRoutingRule:
    Type: AWS::Events::Rule
    Properties:
      EventBusName: !Ref CoreEventBus
      EventPattern:
        source:
          - "ecommerce.orders"
        detail-type:
          - "OrderCreated"
      Targets:
        - Arn: !GetAtt ConsumerBufferQueue.Arn
          Id: "BufferedSqsTarget"
          DeadLetterConfig:
            Arn: !GetAtt TargetDeliveryDLQ.Arn # DLQ for undeliverable bus events
          RetryPolicy:
            MaximumEventAgeInSeconds: 7200 # Discard to DLQ if unrouted after 2 hours
            MaximumRetryAttempts: 5

  # 4. Bounded Consumer (Strict concurrency control prevents downstream DB clogging)
  BufferedProcessorLambda:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: consumer.handler
      Runtime: nodejs20.x
      Timeout: 20
      Events:
        BufferEventSource:
          Type: SQS
          Properties:
            Queue: !GetAtt ConsumerBufferQueue.Arn
            BatchSize: 10
            FunctionResponseTypes:
              - ReportBatchItemFailures
            ScalingConfig:
              MaximumConcurrency: 25 # Limits concurrent writes to downstream DB
```

### EventBridge API Destination Rate Limiting (CloudFormation / SAM)

YAML

```
  # Connection authorization for external target
  ThirdPartyConnection:
    Type: AWS::Events::Connection
    Properties:
      AuthorizationType: API_KEY
      AuthParameters:
        ApiKeyAuthParameters:
          ApiKeyName: "Authorization"
          ApiKeyValue: "Bearer secret-token"

  # API Destination with built-in rate-limiting backpressure
  RateLimitedApiDestination:
    Type: AWS::Events::ApiDestination
    Properties:
      ConnectionArn: !GetAtt ThirdPartyConnection.Arn
      HttpMethod: POST
      InvocationEndpoint: "https://api.partner.com/v1/orders"
      InvocationRateLimitPerSecond: 50 # Caps egress rate to protect partner from clogging
```

## Related Topics

- [[AWS SQS at Scale: High-Throughput Processing, Concurrency, and Backpressure]]
    
      
    
- [[AWS SNS vs. Amazon EventBridge: Architecture, Differences, and Combined Patterns]]
    
      
    
- [[AWS Lambda Concurrency: Reserved vs Provisioned]]
    
      
    
- [[Leaky Bucket and Token Bucket Rate Limiting]]
    
      
    
- [[Dead Letter Queue Redrive Strategies]]
    
      
    

## Tags

#fullstack #interview #aws #eventbridge #serverless #system-design #distributed-systems #backpressure

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups