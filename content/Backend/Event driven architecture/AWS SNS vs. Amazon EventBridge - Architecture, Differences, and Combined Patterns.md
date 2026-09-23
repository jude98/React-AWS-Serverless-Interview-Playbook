# AWS SNS vs. Amazon EventBridge: Architecture, Differences, and Combined Patterns

## Key Concepts

- **Core Mental Model**:
    
      
    - **Amazon SNS** is a **fast pub/sub megaphone**: it broadcasts a payload to thousands/millions of subscribers with sub-30ms latency and high throughput.
        
          
        
    - **Amazon EventBridge** is an **intelligent event router (event bus)**: it evaluates event payloads using declarative pattern-matching rules, transforms data, and directs events to heterogeneous cloud targets.
        
          
        
- **Application-to-Person (A2P) vs. System-to-System (A2A)**:
    
      
    - SNS natively supports end-user communication channels: SMS, Mobile Push Notifications, Email, and HTTP/S webhooks.
        
          
        
    - EventBridge targets serverless compute, microservices, databases, and third-party SaaS APIs; it does not send SMS or push notifications directly.
        
          
        
- **Target Reach & Glue Code Elimination**: EventBridge integrates directly with 35+ AWS services (Step Functions, Kinesis, ECS Tasks, API Destinations) without intermediate Lambda glue code. SNS targets primarily SQS, Lambda, Kinesis Firehose, and HTTP/S endpoints.
    
      
    
- **Filtering & Transformation**:
    
      
    - SNS filtering relies mostly on message attributes (metadata) with basic prefix/exact matching.
        
          
        
    - EventBridge offers deep JSON content-based pattern matching (numeric comparisons, prefix, suffix, wildcards, CIDR, `exists`) and payload transformation via Input Transformers before target invocation.
        
          
        
- **Ordering & Replay**:
    
      
    - SNS supports FIFO topics with strict ordered delivery when paired with SQS FIFO.
        
          
        
    - EventBridge does not guarantee FIFO order, but provides **Event Archive & Replay**, allowing you to re-process historical events after a bug fix.
        
          
        

## Common Interview Questions

- How do you decide between Amazon SNS and Amazon EventBridge for an asynchronous microservice integration?
    
      
    
- What is the difference between message filtering in SNS vs. pattern matching in EventBridge?
    
      
    
- Can EventBridge completely replace SNS in modern serverless architectures? Where does SNS still hold an advantage?
    
      
    
- How do throughput limits, operational latency, and pricing compare between SNS and EventBridge?
    
      
    
- How and why would you chain EventBridge and SNS together in the same architecture?
    
      
    
- What happens when you need to send notifications directly to end users (SMS/Email) from an EventBridge-driven domain?
    
      
    

## Strong Answers / Talking Points

### 1. The Direct Comparison: Feature Breakdown

|**Feature**|**Amazon SNS**|**Amazon EventBridge**|
|---|---|---|
|**Primary Architecture Pattern**|High-throughput Pub/Sub broadcast|Smart Enterprise Event Bus (Choreography)|
|**Latency**|Extremely low (typically $<30\text{ ms}$)|Slightly higher (typically $50\text{–}200\text{ ms}$)|
|**Scale / Throughput**|Virtually unlimited TPS; millions of subscribers per topic|Soft TPS limits per region (adjustable via quotas)|
|**Direct End-User Endpoints**|Native SMS, Email, Mobile Push, Webhooks|None (requires invoking SNS, Pinpoint, or 3rd-party API)|
|**Targets Supported**|Limited: SQS, Lambda, Kinesis Firehose, HTTP/S|35+ AWS services, Step Functions, ECS, API Destinations|
|**Filtering Capabilities**|Attribute-based (up to 10 attributes) & basic payload|Deep content-based JSON pattern matching (numeric, regex, IP, etc.)|
|**Payload Transformation**|Not supported (forwards raw payload)|Built-in Input Transformers (rewrites JSON before target)|
|**FIFO / Ordering**|Supported (SNS FIFO + SQS FIFO)|Not supported|
|**Event Replay & Archive**|FIFO only (archive up to 365 days)|Supported natively on any bus (indefinite retention)|
|**SaaS / Third-Party Integration**|Requires custom ingestion lambdas|Native Partner Event Sources (Auth0, Stripe, DataDog, etc.)|

### 2. When to Use SNS

- **High Fan-Out to End Users**: When you need to send mobile push notifications, transactional emails, or SMS alerts (A2P).
    
      
    
- **Massive Ingestion Scale & Ultra-Low Latency**: When microservices need sub-30ms broadcast latency across hundreds of thousands of identical subscribers.
    
      
    
- **Strict FIFO Ordering Requirements**: When combined with SQS FIFO for sequential processing pipelines (e.g., financial ledger events).
    
      
    
- **Simple, Homogeneous Broadcast**: When 5 downstream services all need to process 100% of the messages without selective, complex routing logic.
    
      
    

### 3. When to Use EventBridge

- **Complex Choreography & Routing**: Routing events based on nested JSON data (e.g., only trigger worker if `$.detail.payment.amount > 1000` and `$.detail.tier == "VIP"`).
    
      
    
- **Direct Service Invocation (Zero Glue Code)**: Triggering AWS Step Functions, starting ECS Tasks, or writing directly to Kinesis without maintaining intermediate Lambda functions.
    
      
    
- **Third-Party SaaS & Cross-Account Routing**: Ingesting webhooks from external providers (Stripe, GitHub, PagerDuty) or centralizing events across multiple AWS accounts in an organization.
    
      
    
- **Event Auditing & Disaster Recovery**: Storing events via EventBridge Archive to replay traffic into staging or replay failed batches after fixing a consumer bug.
    
      
    

### 4. When and How to Use Both Together

Combining EventBridge and SNS leverages **EventBridge as the routing brain** and **SNS as the broadcast/notification muscle**:

  

- **Pattern A: EventBridge $\to$ SNS (Smart Router to Human Notification)**
    
    EventBridge captures fine-grained system domain events (e.g., `OrderFailed` where `reason == "SUSPECTED_FRAUD"`). Instead of writing custom Lambda code to notify security personnel, EventBridge targets an SNS topic that delivers SMS and email alerts directly to on-call teams.
    
      
    
- **Pattern B: SNS $\to$ EventBridge (High-Throughput Ingestion to Smart Bus)**
    
    A high-frequency ingest service publishes raw events to SNS to absorb extreme spikes with minimal latency. SNS fans out to an EventBridge bus for content routing, filtering, and cross-account dispatch.
    
      
    
- **Pattern C: EventBridge Routing to SNS Fan-Out Arrays**
    
    An e-commerce event on EventBridge routes an order event to an SNS topic dedicated to marketing broadcast campaigns, triggering high-scale webhook pushes to external affiliates.
    
      
    

## Code Snippets / Examples

### AWS SAM: EventBridge Routing Directly to an SNS Target

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: EventBridge evaluating payment events and routing VIP alerts to SNS.

Resources:
  # 1. Central Event Bus
  CommerceEventBus:
    Type: AWS::Events::EventBus
    Properties:
      Name: commerce-event-bus

  # 2. SNS Alert Topic for High-Priority Notifications
  VipCustomerAlertTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: vip-order-alerts
      Subscription:
        - Protocol: email
          Endpoint: "ops-team@example.com"

  # 3. SNS Topic Policy allowing EventBridge to Publish
  EventBridgeToSNSPolicy:
    Type: AWS::SNS::TopicPolicy
    Properties:
      Topics:
        - !Ref VipCustomerAlertTopic
      PolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Service: events.amazonaws.com
            Action: sns:Publish
            Resource: !Ref VipCustomerAlertTopic

  # 4. EventBridge Rule: Advanced Content-Based Filter (Numeric + String check)
  HighValueOrderRule:
    Type: AWS::Events::Rule
    Properties:
      EventBusName: !Ref CommerceEventBus
      Description: "Routes orders over $1000 directly to SNS without Lambda"
      EventPattern:
        source:
          - "commerce.orders"
        detail-type:
          - "OrderPlaced.v1"
        detail:
          totalAmount:
            - numeric: [ ">=", 1000 ] # Numeric comparison (impossible in standard SNS)
          tier:
            - "VIP"
      Targets:
        - Arn: !Ref VipCustomerAlertTopic
          Id: "VipSnsTarget"
          # Input Transformer to format human-readable email message
          InputTransformer:
            InputPathsMap:
              orderId: "$.detail.orderId"
              amount: "$.detail.totalAmount"
            InputTemplate: |
              "ALERT: VIP Order <orderId> placed with total value $<amount>."
```

## Related Topics

- [[AWS SQS at Scale - High-Throughput Processing, Concurrency, and Backpressure|AWS SQS at Scale: High-Throughput Processing, Concurrency, and Backpressure]]
    
      
    
- [[Managing Event Schema Evolution and Changes in Amazon EventBridge]]
    
      
    
- [[AWS Step Functions - Workflow Types, State Machine Patterns & Integration|AWS Step Functions: Orchestration vs Choreography]]
    
      
    
- [[AWS Serverless & Event-Driven Architecture (EDA)|Decoupled Microservice Communication Patterns]]
    
      
    
- [[System Design Scenarios - Payment Workflows, Webhooks, Idempotency & Large S3 Payloads|Idempotency in Distributed Systems]]
    
      
    

## Tags

#fullstack #interview #aws #eventbridge #sns #messaging #system-design #serverless

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups