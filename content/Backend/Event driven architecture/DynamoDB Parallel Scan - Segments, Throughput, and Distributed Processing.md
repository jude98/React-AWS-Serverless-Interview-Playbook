# DynamoDB Parallel Scan: Segments, Throughput, and Distributed Processing

## Key Concepts

- **Sequential Scan Bottleneck**: A standard `Scan` operates synchronously through a single evaluation thread, reading one 1 MB page at a time. It cannot saturate network or compute bandwidth for multi-gigabyte tables.

- **Parallel Scan Architecture**: DynamoDB divides physical partitions logically into $N$ segments (`TotalSegments`). Worker threads or compute instances scan their designated `Segment` ($0 \dots N-1$) concurrently.

- **`TotalSegments` vs. `Segment`**:

    - `TotalSegments`: The total number of parallel workers concurrently accessing the table.

    - `Segment`: The zero-indexed identifier ($0 \le \text{Segment} < \text{TotalSegments}$) assigned to a specific worker.

- **Paging per Segment**: Each segment maintains its own independent pagination token (`ExclusiveStartKey` / `LastEvaluatedKey`). One segment finishing does not mean others are finished.

- **RCU Consumption & Throttling**: A parallel scan consumes Read Capacity Units (RCUs) rapidly. Without application-level rate limiting or adaptive concurrency, it will trigger `ProvisionedThroughputExceededException` or exhaust On-Demand burst capacity.

- **Sizing Heuristic**: Choose `TotalSegments` proportional to the table size and partition count (typically 1 segment per 1–2 GB of data or 1 segment per physical partition), capped by available client worker threads or Lambda concurrency.

## Common Interview Questions

- How does a parallel scan divide data under the hood, and does `TotalSegments` map 1:1 to physical storage partitions?

- Why can a parallel scan cause severe read throttling even if total daily RCUs are well within service quotas?

- How do you manage pagination when running parallel scan segments across distributed AWS Lambda invocations?

- What happens if you change `TotalSegments` midway through an in-progress paginated scan?

- How does DynamoDB `FilterExpression` impact RCU consumption during a parallel scan?

- When should you choose Amazon EMR/Glue or DynamoDB Export to S3 over an application-level parallel scan?

## Strong Answers / Talking Points

### 1. Mechanics of Parallel Scan

- **Logical vs. Physical Partitioning**: DynamoDB hashes partition keys across internal storage nodes (physical partitions, max 10 GB each). When a parallel scan runs, DynamoDB internally maps the logical segment parameter across these partition ranges. `TotalSegments` does not need to equal the exact physical partition count, but setting it close to or a multiple of physical partitions prevents segment skew.

- **Independent Segment Traversal**: Each segment is an independent, non-overlapping slice of the table. A worker running `Segment=2` with `TotalSegments=8` will only read data assigned to slice 2, progressing through standard 1 MB page limits until `LastEvaluatedKey` is `undefined`.

- **Immutable Concurrency Contract**: You **cannot** dynamically alter `TotalSegments` midway through a scan. If `TotalSegments` changes, all pagination tokens (`ExclusiveStartKey`) become invalid because the hashing boundary lines change.

### 2. RCU Impact and Filter Expressions

- **The Filter Trap**: A `FilterExpression` is applied **after** DynamoDB reads up to 1 MB of data from storage.

    > [!warning] Critical RCU Cost Rule
    > 
    > You are charged RCUs for **all data read from disk**, not the records returned after filtering. Running a parallel scan with a restrictive filter still burns massive read capacity.
    > 
    >   

- **Throttling Blast Radius**: An unthrottled parallel scan can consume tens of thousands of RCUs in seconds. If the table shares provisioned or on-demand throughput with critical online transaction processing (OLTP) traffic, the scan can throttle end-user operations.

- **Mitigation**: Run parallel scans during off-peak windows, use `Limit` per request, implement client-side token bucket rate limiters, or isolate analytical scans to an asynchronous stream/replica.

### 3. Architectural Patterns for Large Scans

- **Single-Node Worker Pool**: A Node.js worker pool using `Promise.all` across segments. Best for moderate datasets (< 10 GB).

- **Fan-Out via Step Functions + Lambda**: A coordinator step initiates a Map state running $N$ parallel Lambdas, each processing one `Segment` and writing to S3 or an analytical store.

- **Alternative for Massive Data (> 50 GB)**: Avoid live API scans entirely. Use **DynamoDB Point-in-Time Recovery (PITR) Export to Amazon S3**. It reads directly from DynamoDB continuous backups into Parquet/JSON with zero RCU consumption and zero impact on production traffic.

## Code Snippets / Examples

### Concurrent Parallel Scan Worker (Node.js SDK v3)

```typescript
import { DynamoDBClient, ScanCommand, ScanCommandInput } from "@aws-sdk/client-dynamodb";

const client = new DynamoDBClient({ region: "us-east-1" });
const TABLE_NAME = "HighVolumeOrders";
const TOTAL_SEGMENTS = 4; // Equal to number of concurrent workers

async function scanSegment(segment: number, totalSegments: number): Promise<number> {
  let totalItemsRead = 0;
  let exclusiveStartKey: Record<string, any> | undefined = undefined;

  console.log(`[Worker ${segment}] Started scan.`);

  do {
    const params: ScanCommandInput = {
      TableName: TABLE_NAME,
      Segment: segment,
      TotalSegments: totalSegments,
      Limit: 500, // Small limit to regulate throughput per round-trip
      ExclusiveStartKey: exclusiveStartKey,
      ReturnConsumedCapacity: "TOTAL",
    };

    const response = await client.send(new ScanCommand(params));

    if (response.Items) {
      totalItemsRead += response.Items.length;
      await processBatch(response.Items);
    }

    // Pagination continues strictly within this segment's partition slice
    exclusiveStartKey = response.LastEvaluatedKey;

  } while (exclusiveStartKey !== undefined);

  console.log(`[Worker ${segment}] Finished. Read ${totalItemsRead} items.`);
  return totalItemsRead;
}

async function runParallelScan(): Promise<void> {
  const segmentWorkers: Promise<number>[] = [];

  for (let segment = 0; segment < TOTAL_SEGMENTS; segment++) {
    segmentWorkers.push(scanSegment(segment, TOTAL_SEGMENTS));
  }

  const results = await Promise.all(segmentWorkers);
  const totalScanned = results.reduce((acc, count) => acc + count, 0);
  console.log(`Parallel scan completed. Total items processed: ${totalScanned}`);
}

async function processBatch(items: Record<string, any>[]): Promise<void> {
  // Process items or stream downstream
}
```

### Serverless Fan-Out Pattern (AWS SAM / Step Functions)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Fan-out step function triggering Lambda parallel scan workers.

Resources:
  ParallelScanWorkerFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: scan_worker.handler
      Runtime: nodejs20.x
      Timeout: 900 # Max Lambda execution time for paging through segment
      MemorySize: 1024
      Policies:
        - DynamoDBReadPolicy:
            TableName: HighVolumeOrders

  # Input payload to State Machine: { "totalSegments": 8 }
  # Step Functions Map state invokes worker with: { "segment": 0, "totalSegments": 8 } ... up to 7
```

## Related Topics

- [[Debugging DynamoDB Hot Partitions & Hot Keys|DynamoDB Partitioning Mechanics and Hot Partition Mitigation]]

- [[DynamoDB Capacity Modes - Provisioned with Auto Scaling vs. On-Demand|DynamoDB Capacity Modes: On-Demand vs Provisioned]]

- [[Bulk Updating 1 Million Rows in DynamoDB - Architectural Approaches and Trade-offs|DynamoDB PITR Export to S3 vs Application Scans]]

- [[Frontend API Rate Limiting and Third-Party Resiliency Architecture|Distributed Rate Limiting and Token Bucket Algorithm]]

- [[High-Volume Serverless Webhook Ingestion - WAF, API Gateway Direct SQS Integration, and Throttling|Batch Ingestion Pipelines: SQS to DynamoDB]]

## Tags

#fullstack #interview #aws #dynamodb #nosql #system-design #distributed-systems

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups