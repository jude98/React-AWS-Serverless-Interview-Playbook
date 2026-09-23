# Scaling DynamoDB Streams: High-Volume Event Processing

## Key Concepts

- **Shard-to-Concurrency Coupling**: DynamoDB Streams partitions data into shards mapped directly to table partitions. By default, **1 shard = 1 concurrent Lambda execution** to strictly guarantee in-order delivery per partition key.

- **Parallelization Factor**: Breaks the 1-shard/1-worker limit. Allows configuring concurrent batches per shard (values 1 to 10) while preserving in-order processing per partition key.

- **The IteratorAge Threat**: CloudWatch metric `IteratorAge` measures the latency between record insertion and Lambda processing. If `IteratorAge` approaches **24 hours**, records expire from the stream buffer and are permanently lost.

- **Head-of-Line Blocking**: In stream processing, an unhandled error blocks the entire shard because the poller retries the failed batch until success or expiration.

- **Partial Failure & Bisecting**:

    - `BisectBatchOnFunctionError`: Automatically splits a failing batch in half to isolate the poisonous record.

    - `ReportBatchItemFailures`: Allows returning exact failed sequence numbers so Lambda only retries the failed subset.

- **DynamoDB Streams vs. Kinesis Data Streams (KDS)**:

    - **DynamoDB Streams**: Up to 2 concurrent readers per shard, 24-hour fixed retention, free read requests, shard management fully abstracted by AWS.

    - **Kinesis Data Streams (via DynamoDB Kinesis Adapter)**: Retention up to 365 days, unlimited consumers via Enhanced Fan-Out (EFO), dedicated $2\text{ MB/s}$ read bandwidth per consumer.

## Common Interview Questions

- What determines the initial number of shards in a DynamoDB Stream, and how does Lambda scale its consumers?

- How does `ParallelizationFactor` increase stream consumption throughput while still guaranteeing in-order processing?

- What is `IteratorAge`, why does it spike during high-write bursts, and how do you recover from a runaway `IteratorAge`?

- How does error handling differ between an SQS ESM and a DynamoDB Streams ESM?

- What are the architectural trade-offs between native DynamoDB Streams and streaming DynamoDB changes to Amazon Kinesis Data Streams?

- How do you prevent downstream database exhaustion (e.g., RDS / OpenSearch) when a DynamoDB stream processes millions of mutation records?

## Strong Answers / Talking Points

### 1. The Scaling Bottleneck: Shard-Level Concurrency

- DynamoDB shards split automatically as table storage grows (> 10 GB) or write throughput increases (> 1,000 WCU/shard).

- If write bursts hit a concentrated set of partition keys ("hot shard"), only a single Lambda instance processes that shard by default.

- **Solution — `ParallelizationFactor`**: Configure `ParallelizationFactor` (from 1 to 10) on the Event Source Mapping. Lambda subdivides each shard by partition key hash ranges, invoking up to 10 concurrent Lambda instances per physical shard concurrently without violating per-key ordering.

### 2. Preventing Head-of-Line Blocking & Poison Pills

- In SQS, failed messages return to the queue while other workers continue. In DynamoDB Streams, a failing record halts the shard poller.

- **Resiliency Controls**:

    1. `BisectBatchOnFunctionError: true`: When an error occurs, the batch is split into two halves and retried. Continues recursively until the exact poison record is isolated.

    2. `MaximumRecordAgeInSeconds`: Discards items older than a set threshold (e.g., 3,600s) to keep workers from falling 24 hours behind.

    3. `MaximumRetryAttempts`: Caps retries (e.g., 2–3 attempts) instead of retrying indefinitely until stream expiration.

    4. `DestinationConfig.OnFailure`: Automatically directs skipped/failed records to an SQS Dead Letter Queue (DLQ) for asynchronous analysis.

    5. `FunctionResponseTypes: ['ReportBatchItemFailures']`: Return `itemIdentifier` for stream records to avoid reprocessing successful records within a batch.

### 3. Fan-Out & Rate Decoupling Architecture

- **Problem**: Ingesting 10M stream records/hour directly into external destinations (like Elasticsearch/OpenSearch, third-party APIs, or transactional databases) will overwhelm downstream connection pools.

- **Pattern — Stream Buffer to SQS**:

    - Keep the stream consumer Lambda extremely thin: validate the CDC event, wrap it, and push it in batches (`SendMessageBatch`) into an Amazon SQS queue.

    - SQS buffers the burst and decouples the rigid 24-hour stream lifecycle, allowing downstream worker Lambdas to consume at a controlled rate via `ScalingConfig.MaximumConcurrency`.

### 4. DynamoDB Streams vs. Kinesis Data Streams Integration

|**Feature**|**DynamoDB Streams**|**DynamoDB to Kinesis Data Streams (KDS)**|
|---|---|---|
|**Max Retention**|24 hours (strict)|Up to 365 days|
|**Concurrent Consumers**|Up to 2 processes per shard (beyond 2 causes read throttling)|Unlimited with Enhanced Fan-Out (dedicated HTTP/2 push)|
|**Cost Model**|Included with DynamoDB (free read requests)|Charged per shard-hour, PUT payload units, and EFO data retrieval|
|**Target Use Case**|Single microservice triggers, cache invalidation, audit trails|Multi-consumer event hubs, real-time analytics (Flink/Spark), long retention|

## Code Snippets / Examples

### AWS SAM: Production-Grade DynamoDB Stream Event Source Mapping

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: High-throughput DynamoDB Stream consumer with resilience controls.

Resources:
  OrdersStreamDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: orders-stream-dlq
      MessageRetentionPeriod: 1209600 # 14 days

  OrdersStreamProcessor:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: stream_processor.handler
      Runtime: nodejs20.x
      Timeout: 60
      MemorySize: 1024
      Events:
        TableStreamEvent:
          Type: DynamoDB
          Properties:
            Stream: !GetAtt OrdersTable.StreamArn
            StartingPosition: TRIM_HORIZON
            BatchSize: 100 # Adjust between 10-1000 based on payload size
            MaximumBatchingWindowInSeconds: 2 # Buffer short bursts
            ParallelizationFactor: 5 # 5 concurrent Lambdas per shard (up to 10)
            BisectBatchOnFunctionError: true # Halve batch on unhandled errors
            MaximumRetryAttempts: 3 # Stop endless retry loops
            MaximumRecordAgeInSeconds: 3600 # Drop records older than 1 hour to DLQ
            DestinationConfig:
              OnFailure:
                Type: SQS
                Destination: !GetAtt OrdersStreamDLQ.Arn
            FunctionResponseTypes:
              - ReportBatchItemFailures # Partial batch failure handling
```

### Lambda Handler: Partial Batch Failure with DynamoDB Stream Records

```typescript
import { DynamoDBStreamEvent, DynamoDBBatchResponse } from "aws-lambda";

export const handler = async (event: DynamoDBStreamEvent): Promise<DynamoDBBatchResponse> => {
  const batchItemFailures: { itemIdentifier: string }[] = [];

  for (const record of event.Records) {
    try {
      // EventName: 'INSERT' | 'MODIFY' | 'REMOVE'
      const eventType = record.eventName;
      const newImage = record.dynamodb?.NewImage;
      const oldImage = record.dynamodb?.OldImage;

      await processChange(eventType, oldImage, newImage);

    } catch (err) {
      console.error(`Error processing sequence ${record.dynamodb?.SequenceNumber}:`, err);
      // For DynamoDB streams, the identifier MUST be the SequenceNumber
      if (record.dynamodb?.SequenceNumber) {
        batchItemFailures.push({ itemIdentifier: record.dynamodb.SequenceNumber });
      }
      // Break early: Stream order matters! Avoid attempting subsequent records in this partition
      break;
    }
  }

  return { batchItemFailures };
};

async function processChange(eventType?: string, oldImg?: unknown, newImg?: unknown): Promise<void> {
  // Business logic: Invalidate Redis, fan-out, replicate to analytics
}
```

## Related Topics

- [[Amazon DynamoDB -  Architecture, Data Modeling & Scaling]]

- [[AWS Lambda Event Source Mapping (ESM) & Lambda Internal Queues]]

- [[DynamoDB Single-Table Design - Inventory Management Scenario]]

- [[Debugging DynamoDB Hot Partitions & Hot Keys]]

- [[Distributed Transactions & Event-Driven Architecture - Sagas, 2PC, Resilience & Messaging Selection]]

## Tags

#fullstack #interview #aws #dynamodb #streams #lambda #system-design #distributed-systems

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
