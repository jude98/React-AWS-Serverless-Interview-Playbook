# Debugging DynamoDB Hot Partitions & Hot Keys

## Key Concepts

- **The Partition Throughput Limit**: Each physical DynamoDB partition has strict hard limits: **1,000 WCUs** or **3,000 RCUs** per second. Exceeding this limit on a single partition key causes throttling (`ProvisionedThroughputExceededException`), even if the overall table has ample capacity.
    
      
    
- **Hot Key vs. Hot Partition**: A hot key is an individual partition key receiving disproportionately high read or write traffic (e.g., celebrity user, flash-sale item, or monotonic timestamp). A hot partition occurs when the physical storage node housing that key saturates its capacity ceiling.
    
      
    
- **Adaptive Capacity (The Cushion)**: DynamoDB automatically shifts unused throughput from idle partitions to a hot partition and splits partitions automatically. However, adaptive capacity **cannot exceed the hard ceiling of 1,000 WCU / 3,000 RCU for a single partition key**.
    
      
    
- **Observability Stack**:
    
      
    - **DynamoDB CloudWatch Metrics**: Identifies table-wide throttling symptoms.
        
          
        
    - **DynamoDB Contributor Insights**: Pinpoints the exact partition keys and sort keys causing the hot spot.
        
          
        
    - **AWS CloudTrail Data Events**: Captures API-level request patterns and client identity.
        
          
        
    - **Client-Side SDK Metrics & Traces (AWS X-Ray / OpenTelemetry)**: Correlates application-level caller workflows to hot-key latency spikes.
        
          
        

## Common Interview Questions

- How does DynamoDB partition data physically, and why can a table with 50,000 provisioned RCUs still throttle on a read request?
    
      
    
- What is the difference between table-level throttling and partition-level throttling in CloudWatch metrics?
    
      
    
- What is Amazon DynamoDB Contributor Insights, and how do you use it to identify the exact hot partition key?
    
      
    
- How does DynamoDB Adaptive Capacity mitigate uneven access patterns, and what is its fundamental limit?
    
      
    
- How do you configure AWS CloudTrail Data Events and CloudWatch Logs Insights to audit hot key mutations?
    
      
    
- What architectural patterns resolve a hot key once identified (write sharding, caching, DAX)?
    
      
    

## Strong Answers / Talking Points

### 1. The Detection Phase: Identifying the Symptom

- **CloudWatch Symptoms**:
    
      
    - Monitor `ReadThrottleEvents` and `WriteThrottleEvents` alongside `ConsumedReadCapacityUnits` / `ConsumedWriteCapacityUnits`.
        
          
        
    - If throttles occur while total consumed capacity is far below the table-level provisioned/target limit, you have a **partition-level hot spot**.
        
          
        
    - Review `TransactionConflictExceptions` if using multi-item transactions on concurrent records with the same partition key.
        
          
        

### 2. The Diagnosis Phase: Locating the Culprit Key

- **Primary Tool: Amazon DynamoDB Contributor Insights (DCI)**
    
      
    - Fully managed diagnostic tool that continuously analyzes DynamoDB operational logs in real-time.
        
          
        
    - Generates top-N contributor graphs showing:
        
          
        - **Most accessed partition keys** (`PartitionKey`).
            
              
            
        - **Most throttled partition keys** (`ThrottledKeys`).
            
              
            
    - Identifies the exact partition key value (e.g., `PK = "MERCHANT#9821"`) responsible for disproportionate traffic or throttle events.
        
          
        
    - Works on both base tables and Global Secondary Indexes (GSIs).
        
          
        
- **Secondary Tool: AWS CloudTrail Data Events + CloudWatch Logs Insights**
    
      
    - Standard CloudTrail only logs control plane actions (`CreateTable`, `UpdateTable`).
        
          
        
    - Enable **CloudTrail Data Events** for DynamoDB (`PutItem`, `UpdateItem`, `GetItem`, `Query`).
        
          
        
    - Query logs in CloudWatch Logs Insights to aggregate high-frequency callers by IP, IAM role, or request parameters.
        
          
        
    - _Trade-off_: CloudTrail Data Events are expensive; enable them temporarily in production during targeted debugging sessions.
        
          
        
- **Client-Side Instrumentation (OpenTelemetry / AWS X-Ray)**:
    
      
    - Add attributes to active spans capturing the target entity ID / partition key.
        
          
        
    - Filter traces by `aws.table_name = "OrdersTable"` and HTTP response status `400` to find which user actions cause SDK retry backoff loops.
        
          
        

### 3. Remediation Patterns for Hot Keys

Once Contributor Insights identifies the problematic key:

  

- **Write Sharding (Salted Keys)**:
    
      
    - If a key absorbs $>1,000\text{ WCU/s}$ (e.g., an append-only log or live vote counter), append a randomized suffix or deterministic hash to the partition key:
        
          
        
        $$\text{PK} = \text{"ORDER\_AGGREGATE\#"} + \text{random}(0, N)$$
        
    - Reads must scatter-gather across all $N$ partition segments and aggregate results client-side.
        
          
        
- **Read Caching (Amazon DAX or ElastiCache/Redis)**:
    
      
    - For hot read keys (e.g., global site settings or trending catalog items), insert **DynamoDB Accelerator (DAX)** in-memory microsecond caching in front of the table to intercept reads before hitting physical partitions.
        
          
        
- **GSI Sparse Indexing & Key Distribution**:
    
      
    - Hot keys often occur on GSIs with low cardinality (e.g., `status = "PENDING"` across 10M records). Redesign the GSI partition key to combine status with date or tenant (`status#date`).
        
          
        

### Diagnostic Tool Comparison

|**Diagnostic Tool**|**What It Reveals**|**Data Granularity**|**Performance / Cost Impact**|
|---|---|---|---|
|**CloudWatch Metrics**|Aggregate table throughput and throttle event counts|1-minute system metrics; no key visibility|Free / default; detects the symptom only|
|**Contributor Insights (DCI)**|Exact hot partition and sort key values causing throttles|Top-N contributor rankings in real-time|Low monthly fee per table; zero impact on base table RCUs|
|**CloudTrail Data Events**|Exact API caller identity, request payload, IP, and timestamp|Full granular audit logs per API invocation|High ingest cost; best for temporary investigations|
|**AWS X-Ray / OpenTelemetry**|Client-side call chains, execution latency, and retry loops|Distributed trace context per customer request|Minimal overhead; requires application-level instrumentation|

## Code Snippets / Examples

### Enabling Contributor Insights via AWS CLI

```bash
# Enable Contributor Insights on the base table
aws dynamodb update-contributor-insights \
    --table-name HighVolumeOrders \
    --contributor-insights-action ENABLE

# Enable Contributor Insights on a specific Global Secondary Index (GSI)
aws dynamodb update-contributor-insights \
    --table-name HighVolumeOrders \
    --index-name CustomerEmail-OrderDate-GSI \
    --contributor-insights-action ENABLE
```

### Querying CloudTrail Data Events via CloudWatch Logs Insights

```sql
-- Identify the top 20 partition key targets for throttled PutItem/UpdateItem calls
fields @timestamp, eventName, userIdentity.arn, responseElements
| filter eventSource = 'dynamodb.amazonaws.com'
| filter errorCode = 'ProvisionedThroughputExceededException'
| filter eventName in ['PutItem', 'UpdateItem', 'GetItem', 'Query']
| stats count(*) as throttleCount by requestParameters.tableName, requestParameters.key
| sort throttleCount desc
| limit 20
```

### Mitigating Hot Write Keys via Partition Key Salting (TypeScript)

```typescript
import { DynamoDBClient, UpdateItemCommand } from "@aws-sdk/client-dynamodb";

const ddb = new DynamoDBClient({ region: "us-east-1" });
const TOTAL_SHARDS = 10; // Distributes write traffic across 10 physical partition keys

export async function recordTelemetryEvent(metricName: string, value: number): Promise<void> {
  // Compute random shard suffix: e.g., "CPU_UTILIZATION#3"
  const randomShard = Math.floor(Math.random() * TOTAL_SHARDS);
  const shardedPartitionKey = `${metricName}#${randomShard}`;

  const command = new UpdateItemCommand({
    TableName: "AggregatedMetrics",
    Key: {
      PK: { S: shardedPartitionKey },
      SK: { S: new Date().toISOString() },
    },
    UpdateExpression: "ADD #val :v",
    ExpressionAttributeNames: {
      "#val": "metricValue",
    },
    ExpressionAttributeValues: {
      ":v": { N: value.toString() },
    },
  });

  await ddb.send(command);
}
```

## Related Topics


- [[Amazon DynamoDB -  Architecture, Data Modeling & Scaling]]

- [[DynamoDB Capacity Modes - Provisioned with Auto Scaling vs. On-Demand]]

- [[DynamoDB Parallel Scan - Segments, Throughput, and Distributed Processing]]

- [[DynamoDB Single-Table Design - Inventory Management Scenario]]

- [[Adding a Global Secondary Index (GSI) to a Large DynamoDB Table]]


## Tags

#fullstack #interview #aws #dynamodb #database #observability #system-design #distributed-systems

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups