

## Key Concepts

- **Core Pattern:** Fully managed **Publish/Subscribe (Pub/Sub)** messaging service providing instantaneous, many-to-many (`1-to-N`) push notifications to distributed subscribers.
    
      
    
- **Topics & Subscriptions:**
    
      
    - **Topic:** A communication channel where publishers broadcast messages without awareness of consumers.
        
          
        
    - **Subscriber:** Endpoints that subscribe to a topic to receive messages automatically (AWS Lambda, SQS, HTTP/HTTPS webhooks, Email, SMS, Mobile Push).
        
          
        
- **Two Topic Types:**
    
      
    - **Standard Topics:** Nearly unlimited message throughput, best-effort message ordering, at-least-once message delivery.
        
          
        
    - **FIFO Topics:** Strictly preserved message ordering, exactly-once delivery/deduplication, throughput capped (up to 300 msg/s or 3,000 msg/s with batching, expandable with high throughput mode).
        
          
        
- **Fanout Pattern:** Publishing an event to a single SNS topic that simultaneously pushes the message to multiple downstream Amazon SQS queues, decoupling independent microservice domains.
    
      
    
- **Message Filtering:** Subscribers can attach JSON **Subscription Filter Policies** to filter messages by attributes or body, preventing unnecessary invocations or queue clutter.
    
      
    

## Common Interview Questions

- What is the difference between Amazon SNS and Amazon SQS, and how do they work together in the Fanout pattern?
    
      
    
- When would you use Amazon SNS over Amazon EventBridge?
    
      
    
- How does SNS handle delivery retries, backoff strategies, and Dead Letter Queues (DLQs) for failed push endpoints?
    
      
    
- What are Subscription Filter Policies in SNS, and how do they reduce downstream compute costs?
    
      
    
- What is the difference between SNS Standard Topics and SNS FIFO Topics?
    
      
    
- How do you secure an SNS topic to prevent unauthorized publishers or subscriber hijacking?
    
      
    

## Strong Answers / Talking Points

### 1. Messaging Comparison: SNS vs. SQS vs. EventBridge

|**Feature**|**Amazon SNS**|**Amazon SQS**|**Amazon EventBridge**|
|---|---|---|---|
|**Model**|**Pub/Sub (Push)**|**Message Queue (Pull)**|**Event Bus (Push/Rule-based)**|
|**Relationship**|$1\text{-to-Many}$ (Fanout)|$1\text{-to-}1$ (Decoupling)|$Many\text{-to-}Many$ (Integration)|
|**Consumer Nature**|Passive; SNS pushes directly to endpoint.|Active; consumers pull/poll batches from queue.|Passive; routes to targets via content rules.|
|**Persistence**|Ephemeral (drops message if subscriber unavailable and no DLQ).|Durable (messages stored up to 14 days).|Ephemeral (unless archived or routed to queue).|
|**Throughput / Latency**|Ultra-high throughput, single-digit millisecond latency.|Ultra-high throughput, polling latency.|Moderate-to-high throughput, ~tens of ms latency.|
|**Best For**|High-throughput fanout, direct alerts (SMS/Email), push endpoints.|Workload buffering, rate limiting, task queues.|Complex routing, SaaS integration, enterprise event buses.|

### 2. The SNS-to-SQS Fanout Architecture

- **Problem:** When an event occurs (e.g., `OrderPlaced`), multiple independent microservices must react (Inventory Service reserves items, Shipping Service generates labels, Analytics Service records data).
    
      
    
- **Anti-Pattern:** Order service sending sequential HTTP requests to each service—leads to tight coupling, high latency, and cascading failure if one service is down.
    
      
    
- **Fanout Solution:**
    
      
    1. The Order Service publishes `OrderPlaced` **once** to an SNS Topic.
        
          
        
    2. Multiple SQS queues (one for Inventory, one for Shipping, one for Analytics) subscribe to the SNS Topic.
        
          
        
    3. SNS duplicates and delivers the message to all subscribed SQS queues simultaneously in parallel.
        
          
        
    4. Each service's worker pool (or Lambda ESM) reads from its dedicated queue at its own pace.
        
          
        

### 3. Message Filtering & Dead Letter Queues (DLQs)

- **Subscription Filter Policy:** Evaluates attributes on incoming messages. If a filter policy matches, the subscriber receives the message; otherwise, it is skipped entirely without incurring subscriber compute or ingestion costs.
    
      
    
- **Subscriber DLQs:** If an endpoint (like an HTTP webhook or Lambda) fails to receive the message after retry attempts (up to 100 retries over hours/days depending on delivery policy), SNS routes the unhandled message to a configured **SQS Dead Letter Queue (DLQ)** attached to the subscription.
    
      
    

## Code Snippets / Examples

### 1. Publishing an Event with Message Attributes (TypeScript / Node.js)

TypeScript

```
import { SNSClient, PublishCommand } from "@aws-sdk/client-sns";

const sns = new SNSClient({});

export async function publishOrderEvent(orderId: string, orderTotal: number) {
  const command = new PublishCommand({
    TopicArn: process.env.ORDER_EVENTS_TOPIC_ARN,
    Message: JSON.stringify({ orderId, orderTotal, timestamp: new Date().toISOString() }),
    // MessageAttributes allow subscribers to filter without reading the body
    MessageAttributes: {
      EventType: {
        DataType: "String",
        StringValue: "OrderPlaced",
      },
      Tier: {
        DataType: "String",
        StringValue: orderTotal > 1000 ? "VIP" : "Standard",
      },
    },
  });

  await sns.send(command);
}
```

### 2. AWS SAM Template: SNS Topic to SQS Fanout with Subscription Filtering

YAML

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: SNS to SQS Fanout pattern with filter policy

Resources:
  # Central Event Topic
  OrderEventsTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: order-events-topic

  # Queue 1: General Inventory Queue (Subscribes to all OrderPlaced events)
  InventoryQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: inventory-order-queue

  # Queue 2: VIP Fraud Check Queue (Subscribes only to VIP tier)
  VipFraudQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: vip-fraud-queue

  # SQS Queue Policy granting SNS permission to push messages
  InventoryQueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref InventoryQueue
        - !Ref VipFraudQueue
      PolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Service: sns.amazonaws.com
            Action: sqs:SendMessage
            Resource: "*"
            Condition:
              ArnEquals:
                aws:SourceArn: !Ref OrderEventsTopic

  # Subscription for Inventory Queue
  InventorySubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref OrderEventsTopic
      Protocol: sqs
      Endpoint: !GetAtt InventoryQueue.Arn
      RawMessageDelivery: true

  # Subscription with Filter Policy for VIP Queue
  VipFraudSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref OrderEventsTopic
      Protocol: sqs
      Endpoint: !GetAtt VipFraudQueue.Arn
      RawMessageDelivery: true
      FilterPolicy:
        Tier:
          - "VIP"
```

## Related Topics

- [[Amazon-SQS-Queue-Types-and-Internal-Mechanics]]
    
      
    
- [[AWS-Serverless-and-Event-Driven-Architecture]]
    
      
    
- [[Message-Brokers-Kafka-vs-RabbitMQ-vs-SQS]]
    
      
    
- [[Amazon-EventBridge-and-Event-Driven-Routing]]
    
      
    
- [[Idempotency-in-Distributed-Systems]]
    
      
    

## Tags

#fullstack #interview #aws #sns #pubsub #system-design #event-driven

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups