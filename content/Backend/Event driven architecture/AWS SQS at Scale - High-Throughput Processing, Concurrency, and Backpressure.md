# AWS SQS at Scale: High-Throughput Processing, Concurrency, and Backpressure

## Key Concepts

- **Standard vs. FIFO Throughput**: Standard queues offer nearly unlimited transactions per second (TPS). FIFO queues support 300 msg/s (3,000 with batching) by default, and up to 70,000+ msg/s with High Throughput FIFO mode enabled.

- **Producer-Side Batching**: Send up to 10 messages (max 256 KB total payload) per `SendMessageBatch` API call to cut network overhead, reduce AWS API costs by 90%, and prevent client-side socket starvation.

- **Consumer-Side Batching & Windows**: Lambda Event Source Mapping (ESM) pools messages using `BatchSize` (up to 10,000 for Standard, 10 for FIFO) and `MaximumBatchingWindowInSeconds` (0–300s) to balance throughput against latency.

- **Visibility Timeout Rule of Thumb**: Must be set to at least **6x the consumer Lambda timeout** to allow the ESM to retry invocations without duplicate deliveries during transient delays.

- **Partial Batch Failures**: Enable `ReportBatchItemFailures` (`FunctionResponseTypes: ['ReportBatchItemFailures']`) to return only failed `itemIdentifier` IDs; avoids re-processing successfully processed messages in the batch.

- **FIFO Partition Concurrency**: Concurrency on a FIFO queue is strictly bounded by the number of distinct `MessageGroupId` values. 10 distinct IDs = maximum 10 concurrent Lambda invocations, regardless of queue depth.

- **Backpressure & Poison Pill Defense**: Manage burst traffic using SQS buffering, set Dead Letter Queues (DLQ) with `maxReceiveCount` (typically 3–5), and apply Lambda `ScalingConfig.MaximumConcurrency` to protect downstream datastores from connection exhaustion.

## Common Interview Questions

- How do you design an ingestion pipeline that handles tens of millions of SQS messages per hour without overwhelming downstream databases?

- What is the exact purpose of `ReportBatchItemFailures`, and how does message reprocessing work without it?

- Why does setting `VisibilityTimeout` equal to Lambda function timeout cause message duplication loops under high load?

- How does `MessageGroupId` determine consumer concurrency in SQS FIFO, and what causes the "head-of-line blocking" antipattern?

- What strategies mitigate queue backpressure when Lambda consumer scaling is capped by a relational database bottleneck?

- When would you choose SNS fan-out to multiple standard queues over a single queue architecture?

## Strong Answers / Talking Points

### 1. High-Throughput Sender Architecture

- **API Batching**: Never call `SendMessage` in a tight loop. Always buffer in memory and flush using `SendMessageBatch` (up to 10 messages / 256 KB limit). Use the AWS SQS Buffered Asynchronous Client or custom in-memory batch buffers (e.g., RxJS or async queues in Node.js/Go).

- **Payload Offloading**: For payloads exceeding 256 KB, use the **Claim Check Pattern**: upload the raw payload to Amazon S3 and pass the S3 object reference (`bucket` and `key`) in the SQS message body.

- **Fan-Out via SNS/EventBridge**: When ingestion bursts exceed single-producer limits, write to an SNS topic or EventBridge bus that fans out to multiple regional SQS queues.

### 2. High-Throughput Receiver & Lambda ESM Mechanics

- **Scaling Behavior**: Lambda ESM long-polls SQS using 5 concurrent internal polling processes. For Standard queues, it scales out by adding up to 60 instances per minute (up to 1,000 concurrent executions max by default).

- **Visibility Timeout Math**:

    > [!tip] Formula
    > 
    > $\text{VisibilityTimeout} \ge 6 \times \text{Function Timeout}$
    > 
    >   
    > 
    > _Example_: If Lambda timeout is 30 seconds, set Queue `VisibilityTimeout` to at least 180 seconds.
    > 
    >   

- **Partial Batch Handling**: Without `ReportBatchItemFailures`, throwing an unhandled exception fails the **entire batch**, returning all 10 messages back to the queue even if 9 succeeded. With `ReportBatchItemFailures`, the handler returns a payload containing `batchItemFailures: [{ itemIdentifier: messageId }]`, deleting succeeded messages from the queue and retrying only failed ones.

### 3. FIFO vs. Standard Queue Trade-offs

- **Ordering vs. Concurrency**:

    - **Standard**: Best-effort ordering, at-least-once delivery, infinite horizontal scaling.

    - **FIFO**: Exactly-once processing, strict ordering per `MessageGroupId`. Concurrency cannot exceed the count of active, distinct `MessageGroupId`s.

- **Head-of-Line Blocking**: If message $N$ in a FIFO group fails and is returned to the queue, messages $N+1, N+2, \dots$ with the same `MessageGroupId` are blocked from being processed until message $N$ succeeds or is moved to the DLQ.

- **When to Use**: Use FIFO only for strict sequential workflows (e.g., financial ledger transactions, e-commerce order state machines). Use Standard queues for independent parallel tasks (e.g., image resizing, notification delivery, webhooks).

### 4. Mitigating Backpressure & Downstream Saturation

- **Backpressure Signs**: High `ApproximateNumberOfMessagesVisible` and `ApproximateAgeOfOldestMessage` metrics in CloudWatch.

- **Downstream Protection**: If your consumer writes to Amazon RDS or an external 3rd-party API with rate limits:

    1. Set `MaximumConcurrency` on the Lambda event source mapping to cap concurrent execution instances (e.g., limit to 50 connections).

    2. Implement an exponential backoff retry pattern inside the worker.

    3. Route unprocessable messages to a DLQ after 3–5 attempts to unblock the pipeline, then alert on `ApproximateNumberOfMessagesVisible` on the DLQ.

## Code Snippets / Examples

### AWS SAM Template: High-Throughput Queue + ESM with Batch Failures

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Production-grade SQS + Lambda ESM pattern for millions of requests.

Resources:
  # Dead Letter Queue for poison pill isolation
  ProcessingDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: high-scale-dlq
      MessageRetentionPeriod: 1209600 # 14 days

  # Main Processing Queue
  ProcessingQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: high-scale-queue
      VisibilityTimeout: 180 # 6x function timeout (30s)
      ReceiveMessageWaitTimeSeconds: 20 # Long-polling enabled
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt ProcessingDLQ.Arn
        maxReceiveCount: 3

  # Consumer Lambda
  QueueProcessorFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: processor.handler
      Runtime: nodejs20.x
      Timeout: 30
      MemorySize: 512
      Policies:
        - SQSPollerPolicy:
            QueueName: !GetAtt ProcessingQueue.QueueName
      Events:
        SQSBatchEvent:
          Type: SQS
          Properties:
            Queue: !GetAtt ProcessingQueue.Arn
            BatchSize: 10 # Up to 10,000 with batch window
            MaximumBatchingWindowInSeconds: 5
            FunctionResponseTypes:
              - ReportBatchItemFailures # Critical for partial failure handling
            ScalingConfig:
              MaximumConcurrency: 50 # Prevents downstream DB saturation
```

### Sender: Producer-Side Batching (Node.js SDK v3)

```typescript
import { SQSClient, SendMessageBatchCommand, SendMessageBatchRequestEntry } from "@aws-sdk/client-sqs";
import { randomUUID } from "crypto";

const sqs = new SQSClient({ region: "us-east-1" });

export async function sendBatchedMessages(queueUrl: string, payloads: Record<string, unknown>[]): Promise<void> {
  const CHUNK_SIZE = 10; // Max SQS batch size

  for (let i = 0; i < payloads.length; i += CHUNK_SIZE) {
    const chunk = payloads.slice(i, i + CHUNK_SIZE);

    const entries: SendMessageBatchRequestEntry[] = chunk.map((data) => ({
      Id: randomUUID(),
      MessageBody: JSON.stringify(data),
      // For FIFO queues only:
      // MessageGroupId: data.tenantId,
      // MessageDeduplicationId: data.transactionId
    }));

    const command = new SendMessageBatchCommand({
      QueueUrl: queueUrl,
      Entries: entries,
    });

    const response = await sqs.send(command);

    if (response.Failed && response.Failed.length > 0) {
      console.error("Batch entries failed:", response.Failed);
      // Implement targeted retry logic for entries in response.Failed
    }
  }
}
```

### Receiver: Lambda Handler with Partial Batch Failure Reporting

```typescript
import { SQSBatchResponse, SQSEvent } from "aws-lambda";

export const handler = async (event: SQSEvent): Promise<SQSBatchResponse> => {
  const batchItemFailures: { itemIdentifier: string }[] = [];

  for (const record of event.Records) {
    try {
      const payload = JSON.parse(record.body);
      await processRecord(payload);
    } catch (err) {
      console.error(`Failed to process message ${record.messageId}:`, err);
      // Append ONLY the failed ID so SQS retries this individual message
      batchItemFailures.push({ itemIdentifier: record.messageId });
    }
  }

  return { batchItemFailures };
};

async function processRecord(data: unknown): Promise<void> {
  // Business logic: idempotent write to DB, external API call, etc.
}
```

## Related Topics

- [[AWS Lambda Event Source Mapping (ESM) & Lambda Internal Queues|AWS Lambda Concurrency and Event Source Mappings]]

- [[System Design Scenarios - Payment Workflows, Webhooks, Idempotency & Large S3 Payloads|Idempotency in Distributed Systems]]

- [[SQS DLQ Processing - Correlation IDs, Error Context, and Redrive Pipelines|Dead Letter Queue Redrive Strategies]]

- [[AWS Serverless & Event-Driven Architecture (EDA)|Event-Driven Architecture: SNS vs SQS vs EventBridge]]

- [[AWS VPC & Networking Scenarios - Subnets, Lambda VPC Integration, Endpoints & Security|Database Connection Pooling with AWS RDS Proxy]]

## Tags

#fullstack #interview #aws #serverless #sqs #system-design #distributed-systems

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups