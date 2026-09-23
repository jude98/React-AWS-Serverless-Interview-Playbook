# AWS Lambda Event Source Mapping (ESM) & Lambda Internal Queues

## Key Concepts

- **Event Source Mapping (ESM)**: An AWS-managed poller component that sits between stream/queue-based services (SQS, Kinesis, DynamoDB Streams, Kafka) and your Lambda function.

- **Poll-Based Invocation Model**: SQS does not "push" events to Lambda; the ESM continuously executes long-polling (`ReceiveMessage`) on the queue and synchronously invokes your Lambda function with batches of messages.

- **Lambda Internal Queue (Asynchronous Queue)**: When Lambda is invoked asynchronously (e.g., via S3 events, SNS, EventBridge, or `InvocationType: Event`), events enter an AWS-managed internal queue before execution.

- **ESM vs. Internal Queue Distinction**:

    - **ESM**: Pull-based model for stream/queue sources. SQS is external and caller-managed; ESM handles polling, batching, retrying, and deletion on success.

    - **Internal Queue**: Push-based asynchronous buffer managed entirely by the Lambda service inside its execution control plane.

- **Lifecycle Responsibility**: For SQS via ESM, Lambda automatically deletes messages from SQS only upon successful function return (or un-failed batch items when using partial batch responses).

## Common Interview Questions

- What is an Event Source Mapping (ESM) in AWS Lambda, and which AWS services use it?

- Does SQS push events directly to Lambda, or does Lambda poll SQS? Explain the underlying mechanism.

- What is the difference between Lambda's internal asynchronous queue and an SQS queue integrated via ESM?

- How does the ESM handle retries and message visibility when a Lambda execution crashes or times out?

- What happens if your Lambda function experiences throttling when triggered by an SQS ESM vs. an asynchronous invocation (e.g., S3/SNS)?

## Strong Answers / Talking Points

### 1. What is Event Source Mapping (ESM)?

- **Architecture**: ESM is a serverless, managed poller provisioned as part of the Lambda service infrastructure (at no extra compute cost to the user).

- **Supported Sources**: Pull-based services, divided into two categories:

    - **Queue-based**: Amazon SQS (Standard and FIFO), Amazon MQ (RabbitMQ, ActiveMQ).

    - **Stream-based**: Amazon Kinesis Data Streams, Amazon DynamoDB Streams, Apache Kafka (Amazon MSK and self-managed).

- **Core Workflow for SQS**:

    1. The ESM polls the queue using long-polling (`ReceiveMessageWaitTimeSeconds`).

    2. It aggregates messages up to `BatchSize` or until `MaximumBatchingWindowInSeconds` expires.

    3. It synchronously invokes the Lambda execution environment with the batch.

    4. If the execution succeeds ($200\text{ OK}$), ESM calls `DeleteMessage` or `DeleteMessageBatch` on SQS.

    5. If the execution fails completely (or reports failed message IDs), unacknowledged messages re-appear in the queue after the `VisibilityTimeout`.

### 2. What is the Lambda Internal Asynchronous Queue?

- **How It Works**: When a client or service invokes Lambda asynchronously (`X-Amz-Invocation-Type: Event`), the invocation does not wait for function execution. Lambda places the event into an internal SQS-like queue managed entirely inside AWS Lambda's control plane.

- **Retry Semantics**:

    - The internal queue automatically retries failed executions up to **2 times** (3 executions total) with exponential backoff.

    - Events stay in the internal queue for a configurable retention period (up to 6 hours, default 6 hours).

- **DLQ & On-Failure Destinations**: If all retries fail, Lambda routes the event to an On-Failure Destination (SQS, SNS, EventBridge, or another Lambda) or a configured function DLQ.

### 3. ESM vs. Lambda Internal Queue (Direct Comparison)

|**Feature**|**Event Source Mapping (ESM)**|**Lambda Internal Async Queue**|
|---|---|---|
|**Invocation Type**|Synchronous (`RequestResponse`) under the hood|Asynchronous (`Event`)|
|**Trigger Sources**|SQS, Kinesis, DynamoDB Streams, Kafka|S3, SNS, EventBridge, AWS CLI / SDK async calls|
|**Queue Ownership**|Customer owns and configures the queue (SQS)|Fully managed internally by the AWS Lambda control plane|
|**Backpressure / Throttling**|Slows down polling automatically; messages stay in SQS|Events queue internally up to 6 hours; drops to DLQ if expired|
|**Batching**|Configurable via `BatchSize` and batch windows|Always 1 event per execution (no native batching)|
|**Deletion Control**|ESM deletes from source queue after successful handler execution|Lambda dequeues internally before invoking the worker|

### 4. Throttling and Failure Dynamics

- **ESM + SQS Throttling**: If the function hits its reserved concurrency limit, ESM decreases polling frequency (scales down poller threads) until capacity is available. Messages remain safely buffered in your SQS queue.

- **Internal Async Queue Throttling**: If concurrency is exceeded, events accumulate in the hidden internal queue for up to 6 hours. If reserved concurrency stays at 0 or the retention period expires, events are sent to the function's DLQ/Destination or discarded.

## Code Snippets / Examples

### AWS SAM: Defining an ESM vs. Asynchronous Destination

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  # 1. ESM Pattern: Pull-based from SQS
  SqsConsumerFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: esm_consumer.handler
      Runtime: nodejs20.x
      Timeout: 15
      Events:
        MySQSEventSource:
          Type: SQS
          Properties:
            Queue: !GetAtt MyInputQueue.Arn
            BatchSize: 10
            MaximumBatchingWindowInSeconds: 5
            FunctionResponseTypes:
              - ReportBatchItemFailures

  # 2. Async Queue Pattern: Managed internally by Lambda
  AsyncWorkerFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: async_worker.handler
      Runtime: nodejs20.x
      Timeout: 30
      EventInvokeConfig:
        MaximumRetryAttempts: 2
        MaximumEventAgeInSeconds: 3600 # 1 hour internal buffer
        DestinationConfig:
          OnFailure:
            Type: SQS
            Destination: !GetAtt AsyncDeadLetterQueue.Arn

  MyInputQueue:
    Type: AWS::SQS::Queue

  AsyncDeadLetterQueue:
    Type: AWS::SQS::Queue
```

### Programmatic Async Invocation (Hits Internal Queue)

```typescript
import { LambdaClient, InvokeCommand } from "@aws-sdk/client-lambda";

const lambda = new LambdaClient({ region: "us-east-1" });

export async function triggerAsyncProcess(payload: Record<string, unknown>): Promise<void> {
  const command = new InvokeCommand({
    FunctionName: "AsyncWorkerFunction",
    // InvocationType 'Event' places the message onto Lambda's internal queue
    InvocationType: "Event",
    Payload: Buffer.from(JSON.stringify(payload)),
  });

  const response = await lambda.send(command);
  // Status 202 Accepted confirms it landed in Lambda's internal queue
  console.log(`Enqueued with status code: ${response.StatusCode}`);
}
```

## Related Topics

- [[AWS SQS at Scale - High-Throughput Processing, Concurrency, and Backpressure|AWS SQS at Scale: High-Throughput Processing, Concurrency, and Backpressure]]

- [[AWS Lambda Core Architecture & Execution Model|AWS Lambda Execution Context and Lifecycle]]

- [[Amazon SNS (Simple Notification Service) - Architecture, Fanout & Delivery|Push vs Pull Architectures in Cloud Systems]]

- [[Scaling DynamoDB Streams - High-Volume Event Processing|Kinesis and DynamoDB Streams with Lambda ESM]]

- [[AWS Lambda Event Invocations - Synchronous, Asynchronous & Event Source Mappings|Lambda Concurrency: Reserved vs Provisioned]]

## Tags

#fullstack #interview #aws #lambda #serverless #sqs #system-design

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups