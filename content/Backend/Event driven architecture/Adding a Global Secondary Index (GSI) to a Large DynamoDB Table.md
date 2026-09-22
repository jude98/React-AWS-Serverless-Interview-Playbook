
## Key Concepts

- **Online Index Creation**: DynamoDB allows adding a GSI to an existing table online at any time without downtime or blocking read/write traffic.
    
      
    
- **Two-Phase Backfill**: GSI creation transitions through distinct internal phases:
    
      
    1. `CREATING`: DynamoDB allocates storage partitions and performs an internal backfill scan of all existing items in the base table.
        
          
        
    2. `ACTIVE`: The index is fully populated, synchronized, and open for read queries.
        
          
        
- **Base Table Read Impact**: Backfilling is managed entirely by DynamoDB's storage nodes. It does **not** consume user-allocated Read Capacity Units (RCUs) from the base table.
    
      
    
- **Index Write Capacity Throttling (Backpressure)**: The backfill process writes items to the GSI. If the GSI's provisioned Write Capacity Units (WCUs) are set too low, the backfill throttles.
    
      
    
- **The Hot Table Danger (Write Throttling on Base Table)**: While in `CREATING` status, live writes to the base table replicate asynchronously to the new GSI. If the GSI runs out of WCUs during creation, **writes to the base table will be throttled** to keep the index from falling behind.
    
      
    
- **Sparse Index Advantage**: If only a subset of table items contain the GSI's partition key (`PK`) or sort key (`SK`), DynamoDB only indexes those specific items. This dramatically shortens backfill duration and cuts storage costs.
    
      
    
- **Strict Limit**: You can only add or delete **one GSI at a time** per table via the AWS API.
    
      
    

## Common Interview Questions

- What happens under the hood when you issue a `CreateTable` or `UpdateTable` request to add a GSI on a table with 500 million records?
    
      
    
- Does creating a new GSI consume your base table's provisioned Read Capacity Units (RCUs)?
    
      
    
- Why can adding a GSI cause `ProvisionedThroughputExceededException` errors on your base table's write operations?
    
      
    
- How should you configure capacity modes (On-Demand vs. Provisioned) before adding a GSI to a high-traffic table?
    
      
    
- How do you monitor the progress of a GSI backfill, and what CloudWatch metrics are critical?
    
      
    
- What are the operational trade-offs between selecting `KEYS_ONLY`, `INCLUDE`, or `ALL` attribute projection for a large GSI?
    
      
    

## Strong Answers / Talking Points

### 1. Under-the-Hood Backfill Lifecycle

- **Step 1: Partition Allocation**: DynamoDB assesses the base table size and provisions the initial number of physical storage partitions for the GSI.
    
      
    
- **Step 2: Internal Backfill Scan**: DynamoDB storage nodes read the base table partitions internally and stream items that possess the GSI's partition key directly into the GSI partitions. **This operation does NOT consume base table provisioned RCUs.**
    
      
    
- **Step 3: Catch-Up Phase**: While the backfill runs, any new writes/updates occurring on the base table are buffered and applied to the GSI.
    
      
    
- **Step 4: Transition to `ACTIVE`**: Once the backfill catches up to current real-time writes, the GSI status changes from `CREATING` to `ACTIVE`. Read queries (`Query`, `Scan`) against the GSI are rejected with a `ResourceInUseException` or validation error until the index reaches `ACTIVE`.
    
      
    

### 2. The Primary Risk: Base Table Write Throttling

- **The Core Problem**: DynamoDB requires that an index in the `CREATING` state does not fall infinitely behind real-time table mutations.
    
      
    
- **The Blast Radius**: If the new GSI lacks adequate WCU capacity to ingest both the backfilled data and live incoming writes, DynamoDB intentionally slows down (throttles) writes to the **base table**.
    
      
    
- **Mitigation Strategy**:
    
      
    - **If on Provisioned Mode**: Temporarily over-provision the GSI's WCUs (e.g., matching or exceeding the base table's peak WCU usage) before starting creation. You can scale down the WCUs once the index becomes `ACTIVE`.
        
          
        
    - **If on On-Demand Mode**: On-Demand handles this automatically by scaling write capacity to meet backfill demands, though it is still vulnerable to sudden hot-partition spikes.
        
          
        

### 3. Choosing the Right Projection Strategy

- **Projection Types**:
    
      
    - `KEYS_ONLY`: Only stores the base table PK/SK and the GSI PK/SK. Minimal storage, fastest backfill. Any missing attributes require an application-level fetch against the base table.
        
          
        
    - `INCLUDE`: Projects specific, frequently queried attributes. Ideal balance between query latency and storage cost.
        
          
        
    - `ALL`: Duplicates the entire item into the GSI. Doubles your total storage cost and write capacity consumption on every write.
        
          
        
- **Impact on 100M+ Rows**: Projecting `ALL` on a multi-terabyte table takes significantly longer to backfill and doubles overall storage spend. Always prefer `INCLUDE` or `KEYS_ONLY` for massive tables.
    
      
    

### 4. Step-by-Step Production Runbook

1. **Enable Continuous Backups / PITR**: Ensure Point-in-Time Recovery is active as a safety baseline.
    
      
    
2. **Review Capacity Mode**: If using Provisioned Mode, pre-scale the GSI's WCU settings to handle peak production write throughput plus backfill velocity.
    
      
    
3. **Execute `UpdateTable` during off-peak hours**: Minimizes the write collision rate between live production traffic and the backfill stream.
    
      
    
4. **Monitor CloudWatch Metrics**:
    
      
    - `OnlineIndexPercentageProgress`: Tracks completion percentage ($0\% \to 100\%$).
        
          
        
    - `OnlineIndexConsumedWriteCapacity`: Tracks WCU burn rate of the backfill.
        
          
        
    - `OnlineIndexThrottleEvents`: Must remain at zero. If spikes occur, immediately increase provisioned WCU on the GSI.
        
          
        
    - `WriteThrottleEvents` on the base table: Confirms whether production traffic is being impacted.
        
          
        
5. **Scale Down**: Once status transitions to `ACTIVE`, re-enable Auto Scaling or adjust WCUs to steady-state operational levels.
    
      
    

## Code Snippets / Examples

### AWS CLI: Adding a GSI to an Existing Table

Bash

```
# Adding a GSI requires the UpdateTable API.
# Only ONE index can be created at a time.
aws dynamodb update-table \
    --table-name HighVolumeOrders \
    --attribute-definitions \
        AttributeName=customerEmail,AttributeType=S \
        AttributeName=orderDate,AttributeType=S \
    --global-secondary-index-updates '[
        {
            "Create": {
                "IndexName": "CustomerEmail-OrderDate-GSI",
                "KeySchema": [
                    { "AttributeName": "customerEmail", "KeyType": "HASH" },
                    { "AttributeName": "orderDate", "KeyType": "RANGE" }
                ],
                "Projection": {
                    "ProjectionType": "INCLUDE",
                    "NonKeyAttributes": ["orderStatus", "orderTotal"]
                },
                "ProvisionedThroughput": {
                    "ReadCapacityUnits": 100,
                    "WriteCapacityUnits": 2000
                }
            }
        }
    ]'
```

### Checking Backfill Progress via Node.js SDK v3

TypeScript

```
import { DynamoDBClient, DescribeTableCommand } from "@aws-sdk/client-dynamodb";

const ddb = new DynamoDBClient({ region: "us-east-1" });
const TABLE_NAME = "HighVolumeOrders";
const INDEX_NAME = "CustomerEmail-OrderDate-GSI";

async function pollGsiCreationProgress(): Promise<void> {
  let isBackfilling = true;

  while (isBackfilling) {
    const response = await ddb.send(
      new DescribeTableCommand({ TableName: TABLE_NAME })
    );

    const gsi = response.Table?.GlobalSecondaryIndexes?.find(
      (idx) => idx.IndexName === INDEX_NAME
    );

    if (!gsi) {
      throw new Error(`Index ${INDEX_NAME} not found on table ${TABLE_NAME}.`);
    }

    const status = gsi.IndexStatus; // 'CREATING' | 'ACTIVE' | 'DELETING'
    const backfilling = gsi.Backfilling; // boolean flag provided by AWS

    console.log(`[Status: ${status}] Backfilling: ${backfilling}`);

    if (status === "ACTIVE" && !backfilling) {
      console.log(`GSI ${INDEX_NAME} is fully online and ready for traffic.`);
      isBackfilling = false;
    } else {
      // Sleep for 30 seconds between polling checks
      await new Promise((resolve) => setTimeout(resolve, 30_000));
    }
  }
}
```

### AWS CloudFormation / SAM Representation

YAML

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  OrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: HighVolumeOrders
      BillingMode: PAY_PER_REQUEST # On-Demand handles variable backfill throughput cleanly
      AttributeDefinitions:
        - AttributeName: PK
          AttributeType: S
        - AttributeName: SK
          AttributeType: S
        - AttributeName: customerEmail
          AttributeType: S
        - AttributeName: orderDate
          AttributeType: S
      KeySchema:
        - AttributeName: PK
          KeyType: HASH
        - AttributeName: SK
          KeyType: RANGE
      GlobalSecondaryIndexes:
        - IndexName: CustomerEmail-OrderDate-GSI
          KeySchema:
            - AttributeName: customerEmail
              KeyType: HASH
            - AttributeName: orderDate
              KeyType: RANGE
          Projection:
            ProjectionType: INCLUDE
            NonKeyAttributes:
              - orderStatus
              - orderTotal
```

## Related Topics

- [[DynamoDB Indexing: GSI vs LSI Architectural Differences]]
    
      
    
- [[DynamoDB Sparse Indexes: Design and Cost Optimization]]
    
      
    
- [[DynamoDB Partitioning Mechanics and Hot Partition Mitigation]]
    
      
    
- [[DynamoDB Capacity Modes: On-Demand vs Provisioned]]
    
      
    
- [[Bulk Updating 1 Million Rows in DynamoDB: Architectural Approaches and Trade-offs]]
    
      
    

## Tags

#fullstack #interview #aws #dynamodb #database #system-design #distributed-systems

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups