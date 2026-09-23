# Bulk Updating 1 Million Rows in DynamoDB: Architectural Approaches and Trade-offs

## Key Concepts

- **No Native `UPDATE WHERE`**: DynamoDB lacks a bulk update SQL API; updating 1 million items requires 1 million individual `UpdateItem` calls (or `BatchWriteItem` with `PutItem`, which fully replaces the item).

- **Write Capacity Unit (WCU) Math**: Updating 1 million 1 KB items consumes at least $1,000,000\text{ WCUs}$. If executed over 10 minutes (600 seconds), your table must absorb $\approx 1,667\text{ WCUs/sec}$ evenly distributed across partitions.

- **Production Isolation**: In-place bulk operations run the risk of throttling operational (OLTP) traffic and blowing through provisioned or on-demand burst quotas.

- **Zero-RCU Identification via S3 Export**: Using DynamoDB Point-in-Time Recovery (PITR) Export to Amazon S3 allows you to read and filter 1M target items with **zero RCU consumption** and zero performance impact on the live table.

- **Orchestration Choices**:

    1. **Direct Scan + Update**: Step Functions Distributed Map orchestrating parallel Lambda workers directly on the table.

    2. **Zero-RCU S3 Export + Distributed Map**: S3 PITR Export $\to$ Step Functions Distributed Map reading S3 JSON lines $\to$ controlled concurrent Lambdas executing `UpdateItem`.

    3. **ETL Offload (EMR Serverless / AWS Glue)**: PySpark / Glue reads S3 export, computes changes, and writes updates using throttled DynamoDB connectors.

    4. **Queue Buffer (SQS Fan-Out)**: S3 Export / Scan $\to$ Amazon SQS $\to$ Lambda ESM with concurrency limits to pace write ingestion.

## Common Interview Questions

- How would you backfill or update a status attribute across 1 million items in DynamoDB without causing downtime or throttling for live customer traffic?

- Why can't you use `BatchWriteItem` for partial status updates?

- What are the trade-offs between performing an in-place parallel scan versus using DynamoDB PITR S3 Export?

- How do you handle race conditions where an item is modified by production traffic between the time it is exported to S3 and when the bulk update runs?

- How does AWS Step Functions Distributed Map compare with AWS Glue or EMR for running 1 million row updates?

## Strong Answers / Talking Points

### Approach 1: Zero-RCU S3 PITR Export + Step Functions Distributed Map (Recommended)

- **Architecture**:

    1. Trigger an asynchronous DynamoDB Export to S3 (uses PITR continuous backup data; costs $0.10/GB, consumes **0 RCUs**).

    2. AWS Step Functions runs in **Distributed Map** mode, configured with `ItemReader` pointing directly to the generated S3 JSON files.

    3. Step Functions batches records (e.g., 50–100 items per batch) and invokes worker Lambdas.

    4. Worker Lambdas execute `UpdateItem` with an exponential backoff loop.

- **Concurrency & Rate Limiting**: Step Functions Distributed Map allows you to strictly define `MaxConcurrency` (e.g., 20 concurrent Lambdas) to maintain a steady, predictable WCU burn rate.

- **Handling Race Conditions**: Use a `ConditionExpression` (e.g., `attribute_exists(PK) AND #status <> :new_status AND #updatedAt <= :export_timestamp`) to ensure you do not overwrite real-time customer writes that occurred after the export snapshot.

### Approach 2: Serverless In-Place Scan + Step Functions Distributed Map

- **Architecture**:

    1. A coordinator Lambda determines `TotalSegments` (e.g., 100 segments).

    2. Step Functions Distributed Map iterates through segments $0 \dots 99$.

    3. Each Lambda worker scans its designated segment (`Segment: i, TotalSegments: 100`) with pagination, filters matching records, and executes `UpdateItem`.

- **Trade-offs**:

    - **Pros**: Fully serverless, no intermediate S3 storage footprint.

    - **Cons**: Consumes massive read capacity ($1,000,000+\text{ RCUs}$) directly from the production table. If the table is not heavily over-provisioned, OLTP traffic will experience `ProvisionedThroughputExceededException`.

### Approach 3: S3 Export + AWS Glue / EMR Serverless (Batch Data Approach)

- **Architecture**:

    1. Export table to S3 via PITR.

    2. Run an AWS Glue Job or Amazon EMR Serverless Spark application.

    3. Spark reads the partitioned gzip JSON lines from S3, applies the schema transformation (`status = 'PROCESSED'`), and writes to DynamoDB using the open-source Spark-DynamoDB connector.

- **Trade-offs**:

    - **Pros**: Best for complex multi-attribute transformations, heavy aggregations, or datasets in the 100M–1B range.

    - **Cons**: High cold-start time (2–5 minutes), expensive DPU/worker costs for only 1M items, harder to throttle granularly compared to Step Functions.

### Approach 4: Parallel Scan $\to$ SQS Buffer $\to$ Lambda ESM (Producer-Consumer Buffer)

- **Architecture**:

    1. Scan worker (ECS task or Lambda) iterates over the table and pushes primary keys of matching items into an Amazon SQS queue.

    2. Lambda Event Source Mapping (ESM) reads from SQS with `BatchSize: 10` and `ReportBatchItemFailures`.

    3. Control write throughput strictly by setting `ScalingConfig.MaximumConcurrency` on the ESM (e.g., 30 concurrent functions).

- **Trade-offs**:

    - **Pros**: SQS acts as a shock absorber; protects downstream DynamoDB partitions from burst traffic. Easy visibility into progress via CloudWatch SQS depth metrics.

    - **Cons**: Additional infrastructure to manage; double network hop (DynamoDB $\to$ SQS $\to$ Lambda $\to$ DynamoDB).

### Why `BatchWriteItem` is NOT an In-Place Update

- In an interview, explicitly clarify that **DynamoDB does not support `BatchUpdateItem`**.

- `BatchWriteItem` only supports `PutItem` (full overwrite) and `DeleteItem`.

- If you use `BatchWriteItem`, you must provide the complete item attribute set. Any attributes omitted in the payload are permanently erased from the item.

- To update only `status`, you **must** use individual `UpdateItem` operations with `UpdateExpression: "SET #s = :new_val"`.

### Comparison Matrix

|**Approach**|**Read Impact (RCU)**|**Write Impact (WCU)**|**Cost**|**Operational Complexity**|**Best Used When**|
|---|---|---|---|---|---|
|**S3 Export + Step Functions Map**|**0 RCU** (PITR snapshot)|Controlled via `MaxConcurrency`|Low (S3 export + Step Fn runs)|Low (Fully serverless)|**Recommended for production 1M–50M item updates**|
|**In-Place Scan + Step Functions**|**High** (reads full table)|Controlled via `MaxConcurrency`|Medium (RCU cost + Step Fn)|Low|Non-production tables or small tables (< 5 GB)|
|**S3 Export + AWS Glue / EMR**|**0 RCU**|High / bursty|High (Glue DPU minimums)|Medium-High|Massive analytical datasets (> 100M rows)|
|**Scan + SQS Buffer + Lambda ESM**|**High** (reads full table)|Strict limit via ESM concurrency|Low-Medium|Medium (Queue lifecycle)|When you already have SQS-based batching pipelines|

## Code Snippets / Examples

### AWS Step Functions: Distributed Map over S3 Export JSON (ASL Definition)

```json
{
  "Comment": "Zero-RCU bulk update reading S3 PITR export via Distributed Map",
  "StartAt": "ProcessS3ExportFiles",
  "States": {
    "ProcessS3ExportFiles": {
      "Type": "Map",
      "ItemProcessor": {
        "ProcessorConfig": {
          "Mode": "DISTRIBUTED",
          "ExecutionType": "STANDARD"
        },
        "StartAt": "UpdateDynamoDBBatch",
        "States": {
          "UpdateDynamoDBBatch": {
            "Type": "Task",
            "Resource": "arn:aws:states:::lambda:invoke",
            "Parameters": {
              "FunctionName": "arn:aws:lambda:us-east-1:123456789012:function:bulk-update-worker",
              "Payload": {
                "Items.$": "$.Items"
              }
            },
            "End": true
          }
        }
      },
      "ItemReader": {
        "Resource": "arn:aws:states:::s3:getObject",
        "ReaderConfig": {
          "InputType": "JSONL"
        },
        "Parameters": {
          "Bucket": "my-dynamodb-exports-bucket",
          "Key": "AWSDynamoDB/016789000-abcd/data/file-1.json.gz"
        }
      },
      "ItemBatcher": {
        "MaxItemsPerBatch": 25
      },
      "MaxConcurrency": 20,
      "End": true
    }
  }
}
```

### Worker Lambda: Idempotent `UpdateItem` with Condition Expression

```typescript
import { DynamoDBClient, UpdateItemCommand } from "@aws-sdk/client-dynamodb";

const ddb = new DynamoDBClient({ region: "us-east-1" });
const TABLE_NAME = "OrdersTable";

interface LambdaEvent {
  Items: Array<{
    Item: {
      PK: { S: string };
      SK: { S: string };
      status: { S: string };
      updatedAt: { N: string };
    };
  }>;
}

export const handler = async (event: LambdaEvent): Promise<{ updated: number; skipped: number }> => {
  let updated = 0;
  let skipped = 0;

  for (const record of event.Items) {
    const pk = record.Item.PK.S;
    const sk = record.Item.SK.S;

    try {
      const command = new UpdateItemCommand({
        TableName: TABLE_NAME,
        Key: {
          PK: { S: pk },
          SK: { S: sk },
        },
        UpdateExpression: "SET #status = :newStatus, #updatedAt = :now",
        // Prevent overwriting if item was updated by live traffic during the job
        ConditionExpression: "attribute_exists(PK) AND #status <> :newStatus",
        ExpressionAttributeNames: {
          "#status": "status",
          "#updatedAt": "updatedAt",
        },
        ExpressionAttributeValues: {
          ":newStatus": { S: "ARCHIVED" },
          ":now": { N: Date.now().toString() },
        },
      });

      await ddb.send(command);
      updated++;
    } catch (err: any) {
      if (err.name === "ConditionalCheckFailedException") {
        // Item already updated or changed by live traffic
        skipped++;
      } else {
        console.error(`Failed to update ${pk}#${sk}:`, err);
        throw err; // Trigger Step Functions retry
      }
    }
  }

  return { updated, skipped };
};
```

## Related Topics

- [[DynamoDB Parallel Scan - Segments, Throughput, and Distributed Processing]]

- [[DynamoDB Capacity Modes - Provisioned with Auto Scaling vs. On-Demand]]

- [[Debugging DynamoDB Hot Partitions & Hot Keys]]

- [[Scaling DynamoDB Streams - High-Volume Event Processing]]

- [[Amazon DynamoDB -  Architecture, Data Modeling & Scaling]]

## Tags

#fullstack #interview #aws #dynamodb #step-functions #serverless #system-design

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups