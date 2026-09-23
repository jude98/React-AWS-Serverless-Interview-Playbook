# AWS Lambda Event Invocations: Synchronous, Asynchronous & Event Source Mappings

## Key Concepts

- **Lambda Event Definition:** A JSON-formatted document containing state data indicating _what happened_ (e.g., an HTTP call, an S3 object upload, or a database record change) passed into the function runtime as the `event` parameter.

- **Invocation Models:**

    - **Synchronous (Request-Response):** Caller invokes Lambda and blocks waiting for the execution result.

    - **Asynchronous (Fire-and-Forget):** Caller delivers payload to Lambda’s internal managed event queue and receives an immediate `202 Accepted`; Lambda executes independently in the background.

    - **Polling / Event Source Mapping (ESM):** AWS-managed internal poller reads batches from a streaming or queueing service (SQS, DynamoDB Streams, Kinesis) and synchronously invokes your Lambda.

- **Push vs. Pull:**

    - **Push Model:** The upstream service pushes the event directly to Lambda (either synchronously like API Gateway, or asynchronously like S3/SNS/EventBridge).

    - **Pull (Poll-based) Model:** Lambda provisions an internal fleet of pollers that continuously pull batches from the source (SQS, Kinesis, DynamoDB Streams, Kafka).

- **Dead Letter Queue (DLQ):** An SQS queue or SNS topic where discarded event payloads are automatically routed after retry policies are fully exhausted, preventing data loss.

- **Payload Limits:**

    - **Synchronous payload limit:** **6 MB** (both request and response).

    - **Asynchronous payload limit:** **256 KB**.

## Common Interview Questions

- What is the difference between synchronous push, asynchronous push, and stream-based pull invocations?

- How do error handling, retries, and DLQ configurations differ between Asynchronous Invocations and Event Source Mappings (SQS)?

- What is the difference between a Lambda Dead Letter Queue (DLQ) and Lambda Destinations?

- How does Lambda prevent head-of-line blocking when processing Kinesis or DynamoDB Streams?

- What are the payload size limits for synchronous vs. asynchronous invocations, and how do you process larger payloads?

## Strong Answers / Talking Points

### 1. Synchronous vs. Asynchronous vs. Pull Model Comparison

|**Characteristic**|**Synchronous (Push)**|**Asynchronous (Push)**|**Poll-based / ESM (Pull)**|
|---|---|---|---|
|**Invoking Services**|API Gateway, ALB, Cognito, AWS CLI/SDK (`RequestResponse`)|S3, SNS, EventBridge, CloudWatch Events, SES|SQS, DynamoDB Streams, Kinesis, Amazon MSK|
|**Caller Behavior**|Blocks until response is received or connection times out.|Receives `202 Accepted` immediately (fire-and-forget).|Decoupled; caller writes to queue/stream independently.|
|**Retry Behavior**|Handled entirely by the caller (no automated Lambda retries).|Built-in: retries **2 times** (3 attempts total) with exponential backoff.|Configurable: SQS uses `maxReceiveCount`; Streams retry until record expiry or max retries.|
|**Error Destinations**|Error returned directly in the response payload/HTTP code.|Native Lambda DLQ (SQS/SNS) or **Lambda Destinations** (`OnFailure`).|Configured on the source (SQS Redrive Policy) or ESM Bisect/Destination.|
|**Payload Size Limit**|**6 MB** (request & response).|**256 KB**.|Service-dependent (SQS: 256 KB, Kinesis: 1 MB).|

### 2. Deep Dive: Asynchronous Push (Fire-and-Forget)

- When invoked asynchronously (`InvocationType: Event`), the event enters an **AWS-managed internal queue**.

- **Retry Mechanics:** If the function fails, throttles, or encounters an unhandled exception, Lambda automatically retries twice (interval delay varies up to 5 minutes between attempts). Maximum event age can be configured between **60 seconds to 6 hours**.

- **Lambda Destinations vs. DLQ:**

    - _DLQ (Legacy):_ Only captures the raw invocation payload sent to SQS/SNS on failure; provides no execution metadata or stack trace.

    - _Lambda Destinations (Preferred):_ Supports both `OnFailure` and `OnSuccess` routes (to SQS, SNS, EventBridge, or another Lambda). Passes comprehensive execution context including request payload, response, error message, and stack trace.

### 3. Deep Dive: Pull Model (Event Source Mapping)

- The execution responsibility shifts to Lambda's internal managed pollers:

    - **SQS (Non-streaming):**

        - Poller reads batches (up to 10,000 messages or 10 MB).

        - If the Lambda succeeds, the poller deletes messages from the queue.

        - If it fails, messages become visible again after the visibility timeout expires.

        - Configure SQS redrive policy to direct failed messages to a DLQ after reaching `maxReceiveCount`.

        - Use `ReportBatchItemFailures` so only failed individual records in a batch are retried, avoiding re-processing successful records.

    - **DynamoDB Streams & Kinesis (Streaming):**

        - Processed strictly in-order per shard.

        - A failed record causes the entire shard to halt (head-of-line blocking) unless mitigated.

        - Mitigation controls: **Bisect on error** (splits failing batch in half to isolate the poison pill), **Maximum Record Age**, **Retry Attempts**, and **On-failure destination**.

## Code Snippets / Examples

### 1. Handling Partial Batch Failures with SQS (TypeScript)

```typescript
import { SQSBatchResponse, SQSEvent, SQSHandler } from "aws-lambda";

export const handler: SQSHandler = async (event: SQSEvent): Promise<SQSBatchResponse> => {
  const batchItemFailures: { itemIdentifier: string }[] = [];

  for (const record of event.Records) {
    try {
      await processOrder(JSON.parse(record.body));
    } catch (error) {
      // Return only failed message IDs so SQS re-queues ONLY the failed record
      batchItemFailures.push({ itemIdentifier: record.messageId });
    }
  }

  return { batchItemFailures };
};

async function processOrder(order: any) {
  if (!order.id) throw new Error("Invalid order payload");
}
```

### 2. SAM Template: Async Function with Destinations & SQS Polling with DLQ

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Async Invocations, Lambda Destinations, and SQS Event Source Mappings

Resources:
  # 1. Asynchronous Push Function (S3 -> Lambda -> Destinations)
  AsyncProcessorFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: async-handler.handler
      Runtime: nodejs20.x
      EventInvokeConfig:
        MaximumRetryAttempts: 2
        MaximumEventAgeInSeconds: 3600
        DestinationConfig:
          OnFailure:
            Type: SQS
            Destination: !GetAtt FailureDLQ.Arn
          OnSuccess:
            Type: EventBridge
            Destination: !GetAtt SuccessBus.Arn

  # 2. Pull / Event Source Mapping Function (SQS -> Lambda)
  QueueConsumerFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: queue-handler.handler
      Runtime: nodejs20.x
      Events:
        OrderQueueEvent:
          Type: SQS
          Properties:
            Queue: !GetAtt MainProcessingQueue.Arn
            BatchSize: 10
            FunctionResponseTypes:
              - ReportBatchItemFailures

  # Queues & DLQ Configuration
  MainProcessingQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 300
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt FailureDLQ.Arn
        maxReceiveCount: 5

  FailureDLQ:
    Type: AWS::SQS::Queue

  SuccessBus:
    Type: AWS::Events::EventBus
    Properties:
      Name: processing-success-bus
```

## Related Topics

- [[AWS Lambda Core Architecture & Execution Model|AWS-Lambda-Core-Architecture]]

- [[AWS Serverless & Event-Driven Architecture (EDA)|AWS-Serverless-and-Event-Driven-Architecture]]

- [[Amazon SQS - Queue Types, Internal Mechanics & Limits|Message-Brokers-Kafka-vs-RabbitMQ-vs-SQS]]

- [[System Design Scenarios - Payment Workflows, Webhooks, Idempotency & Large S3 Payloads|Idempotency-in-Distributed-Systems]]

- [[Scaling DynamoDB Streams - High-Volume Event Processing|Streaming-Architectures-Kinesis-vs-DynamoDB-Streams]]

## Tags

#fullstack #interview #aws #lambda #event-driven #sqs #system-design

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
