# Amazon EventBridge: Event Buses, Pipes, Patterns & Schemas

## Key Concepts

- **Core Purpose:** A serverless event bus service that ingests, filters, transforms, and routes real-time data across AWS services, custom microservices, and SaaS applications.

- **Event Bus:** The central routing router. Types include:

    - **Default Bus:** Ingests events from all AWS services in the account (e.g., EC2 state changes, S3 events).

    - **Custom Bus:** Created for application microservices to publish domain events (e.g., `ecommerce-bus`).

    - **Partner Bus:** Ingests events from integrated third-party SaaS providers (e.g., Datadog, Auth0, Shopify).

- **Rules & Event Patterns:** Declarative JSON filtering schemas attached to a bus; incoming events matching the pattern are routed to up to 5 targets.

- **EventBridge Pipes:** Point-to-point integration connecting event producers directly to consumers (Source $\rightarrow$ Enrichment $\rightarrow$ Target) with optional inline filtering, eliminating custom glue code.

- **EventBridge Scheduler:** Dedicated serverless cron/scheduler capable of dispatching millions of one-time or recurring tasks with flexible time windows and time-zone support.

- **Schema Registry & Discovery:** Automatically detects event shapes, versions schemas (OpenAPI/JSONSchema), and generates typed client code bindings (TypeScript, Python, Java).

## EventBridge Routing Architecture

## Common Interview Questions

- How does Amazon EventBridge differ from Amazon SNS in routing, throughput, and operational capabilities?

- What are EventBridge Pipes, and how do they replace Lambda glue code between SQS/DynamoDB Streams and targets?

- How do Event Patterns filter events, and what matching syntax is supported (numeric ranges, prefix, exists)?

- How does the EventBridge Archive and Replay feature simplify debugging and disaster recovery?

- How does the Schema Registry and Schema Discovery work in an enterprise multi-team setup?

- What are the throughput limits, payload constraints, and latency profiles of EventBridge?

## Strong Answers / Talking Points

### 1. EventBridge vs. SNS vs. SQS

- **Use SQS when:** You need durable point-to-point message buffering, rate-limiting, and consumer-controlled pulling.

- **Use SNS when:** You need high-throughput, low-latency, multi-subscriber push notification fanout (e.g., pushing to millions of mobile devices or thousands of HTTP endpoints).

- **Use EventBridge when:** You need **smart content-based routing**, cross-account routing, direct SaaS integrations, and audit/replay capabilities without managing fanout plumbing.

### 2. EventBridge Pipes: Eliminating Glue Code

> [!NOTE]
>
> EventBridge Pipes provide a structured, serverless 4-stage pipeline for direct point-to-point integration:


1. **Source:** Ingests from streaming or queuing sources (SQS, Kinesis, DynamoDB Streams, Kafka).

2. **Filter (Optional):** Drops unwanted records before invoking compute, saving cost.

3. **Enrichment (Optional):** Invokes Lambda, Step Functions, or API Gateway to enhance the payload with extra metadata.

4. **Target:** Delivers the transformed payload to Step Functions, EventBridge Bus, SQS, Kinesis, or API Destinations.

### 3. EventBridge Scheduler vs. Rules (Cron)

- Legacy EventBridge Rules support cron expressions but lack timezone awareness, scale limitations, and one-time scheduling capabilities.

- **EventBridge Scheduler:**

    - Scales to millions of scheduled invocations.

    - Supports **one-time schedules** (e.g., "send user an email exactly 3 days after onboarding at 2:00 PM EST").

    - Native **flexible time windows** to spread invocation bursts and prevent downstream throttling.

### 4. Archives, Replays & Audit Logging

- **Event Archive:** Configured on an event bus to capture and persist all or a filtered subset of events for a defined retention period (or indefinitely).

- **Event Replay:** Reprocesses archived events across a specific time range by redelivering them to the bus.

    - _Primary Use Cases:_ Re-running data after fixing a downstream bug in a consumer, disaster recovery, backfilling analytics pipelines, or reproducing integration bugs.

- **Audit Log:** Captures complete operational event histories directly in Amazon CloudWatch Logs or an S3 archive bucket for regulatory compliance.

### 5. Schema Registry & Discovery

- **Schema Discovery:** When enabled on an event bus, it samples incoming JSON events and dynamically maps their structure.

- **Schema Registry:** Central repository storing event definitions in OpenAPI or JSONSchema format.

- **Code Bindings:** Developers download generated client SDK models directly from the AWS CLI/console, providing autocompletion and compile-time type validation for custom events in TypeScript, Python, or Go.

## Code Snippets / Examples

### 1. Complex Event Pattern Filtering (JSON)

```json
{
  "source": ["service.orders"],
  "detail-type": ["OrderCreated", "OrderUpdated"],
  "detail": {
    "status": ["PENDING", "PROCESSING"],
    "amount": [{ "numeric": [">=", 100.0] }],
    "shippingAddress": {
      "country": [{ "prefix": "US" }]
    },
    "paymentDetails": {
      "cardType": [{ "anything-but": "EXPIRED" }]
    }
  }
}
```

### 2. Emitting an Event to a Custom Bus with TypeScript SDK v3

```typescript
import { EventBridgeClient, PutEventsCommand } from "@aws-sdk/client-eventbridge";

const ebClient = new EventBridgeClient({});

export async function emitParcelShipped(trackingId: string, carrier: string) {
  const command = new PutEventsCommand({
    Entries: [
      {
        EventBusName: "logistics-bus",
        Source: "service.logistics",
        DetailType: "ParcelShipped",
        Time: new Date(),
        Detail: JSON.stringify({
          trackingId,
          carrier,
          status: "IN_TRANSIT",
          timestamp: Date.now(),
        }),
      },
    ],
  });

  await ebClient.send(command);
}
```

### 3. AWS SAM Template: Custom Bus, Archive, Rule, and EventBridge Pipe

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: EventBridge Custom Bus, Archive, Rule and Pipes

Resources:
  # 1. Custom Event Bus
  LogisticsEventBus:
    Type: AWS::Events::EventBus
    Properties:
      Name: logistics-bus

  # 2. Event Archive for Replays and Audits
  LogisticsEventArchive:
    Type: AWS::Events::Archive
    Properties:
      ArchiveName: logistics-archive
      SourceArn: !GetAtt LogisticsEventBus.Arn
      RetentionDays: 90
      EventPattern:
        source:
          - "service.logistics"

  # 3. Rule with Target and Dead-Letter Queue
  HighPriorityOrderRule:
    Type: AWS::Events::Rule
    Properties:
      EventBusName: !Ref LogisticsEventBus
      EventPattern:
        source:
          - "service.logistics"
        detail-type:
          - "ParcelShipped"
      Targets:
        - Arn: !GetAtt TargetSQSQueue.Arn
          Id: "SQSTarget"
          DeadLetterConfig:
            Arn: !GetAtt EventRuleDLQ.Arn

  TargetSQSQueue:
    Type: AWS::SQS::Queue

  EventRuleDLQ:
    Type: AWS::SQS::Queue

  # 4. EventBridge Pipe: Ingest DynamoDB Stream -> Filter -> SQS Target
  OrderStreamPipe:
    Type: AWS::Pipes::Pipe
    Properties:
      Name: ddb-stream-to-sqs-pipe
      RoleArn: !GetAtt PipeExecutionRole.Arn
      Source: !GetAtt OrderDynamoTable.StreamArn
      SourceParameters:
        DynamoDBStreamParameters:
          StartingPosition: LATEST
          BatchSize: 5
      SourceFilter:
        Pattern: '{"eventName": ["INSERT"]}'
      Target: !GetAtt TargetSQSQueue.Arn

  PipeExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: pipes.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: PipeAccess
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - dynamodb:GetRecords
                  - dynamodb:GetShardIterator
                  - dynamodb:DescribeStream
                  - dynamodb:ListStreams
                  - sqs:SendMessage
                Resource: "*"
```

## Related Topics

- [[AWS Serverless & Event-Driven Architecture (EDA)|AWS-Serverless-and-Event-Driven-Architecture]]

- [[Amazon SNS (Simple Notification Service) - Architecture, Fanout & Delivery|Amazon-SNS-Architecture-Fanout-and-Delivery]]

- [[Amazon SQS - Queue Types, Internal Mechanics & Limits|Amazon-SQS-Queue-Types-and-Internal-Mechanics]]

- [[AWS Step Functions - Workflow Types, State Machine Patterns & Integration|AWS-Step-Functions-Workflow-Types-and-Patterns]]

- [[AWS Step Functions - Workflow Types, State Machine Patterns & Integration|Microservices-Choreography-vs-Orchestration]]

## Tags

#fullstack #interview #aws #eventbridge #eda #serverless #system-design

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
