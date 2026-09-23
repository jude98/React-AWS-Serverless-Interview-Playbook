# Amazon DynamoDB: Architecture, Data Modeling & Scaling

## Key Concepts

- **Core Partitioning Model:** DynamoDB hashes the **Partition Key (PK)** with an MD5-based algorithm to assign data to physical partitions; within a partition, items are ordered by the **Sort Key (SK)** (B-Tree structure).

- **Physical Partitions & Replicas:** Each logical partition is replicated across at least **3 storage nodes in 3 separate Availability Zones (AZs)** forming a Paxos-based replica group with one elected **Leader** and follower peer nodes.

- **Write-Ahead Log (WAL):** Writes hit the Leader's memory and append-only WAL on SSD first, replicate synchronously to peers via Paxos quorum (at least 2 of 3 nodes acknowledge), and return success before asynchronously persisting to B-tree storage files.

- **Read Consistency Models:**

    - **Eventually Consistent (Default):** Can read from any follower replica; lowest latency, half the cost ($0.5\text{ RCU}$ per 4 KB).

    - **Strongly Consistent:** Routed strictly to the partition Leader; guaranteed latest data, double the cost ($1\text{ RCU}$ per 4 KB).

- **Hard Single-Item Limit:** Maximum item size is **400 KB** (including attribute names and binary values).

- **Partition Limits:** A single physical partition caps out at **10 GB of storage**, **1,000 WCU**, and **3,000 RCU**.

## Architecture & Storage Replication

## Common Interview Questions

- How does DynamoDB use the Partition Key and Sort Key to locate items on disk?

- How does Paxos replication work under the hood across the 3 AZ storage nodes for writes vs. strongly consistent reads?

- What are the architectural differences, trade-offs, and scaling behaviors between GSI and LSI?

- How are RCU and WCU calculated for standard, transactional, and consistent read operations?

- How do you detect and solve hot partition issues using write sharding (salting)?

- How do you store records that exceed the 400 KB DynamoDB item size limit?

## Strong Answers / Talking Points

### 1. Leader-Follower Replication & Consistency

- **Write Path:** The Request Router directs writes to the partition's **Leader node**. The Leader appends to its Write-Ahead Log (WAL) and initiates Paxos consensus to replica peers in other AZs. Once a quorum (2 out of 3 nodes) commits to WAL, the write is confirmed to the client.

- **Strongly Consistent Read:** Routed exclusively to the **Leader node**, ensuring the returned value reflects every confirmed write quorum.

- **Eventually Consistent Read:** Routed to any random replica (Leader or follower). A lag of a few milliseconds may occur if the replica has not caught up with the latest Paxos WAL commits.

### 2. Global Secondary Index (GSI) vs. Local Secondary Index (LSI)

|**Feature**|**Global Secondary Index (GSI)**|**Local Secondary Index (LSI)**|
|---|---|---|
|**Partition Key**|Can differ from the base table PK.|Must share the **exact same PK** as the base table.|
|**Sort Key**|Any attribute (different from base SK).|Different attribute from base table SK.|
|**Creation Timing**|Anytime (create, modify, or delete on active tables).|**Only during table creation**; immutable afterwards.|
|**Capacity / Provisioning**|Dedicated RCU/WCU (throttling on GSI throttles base table writes!).|Shares base table RCU/WCU.|
|**Consistency**|**Eventually consistent only** (replicated asynchronously via storage log stream).|Supports both **Strong** and **Eventual** consistency.|
|**Storage Cap**|No partition size limit (scales across partitions).|**10 GB max item collection size** per PK (hard limit!).|

### 3. Read & Write Capacity Unit (RCU / WCU) Math

- **WCU (Write Capacity Unit):**

    - $1\text{ WCU} = 1\text{ write/sec}$ for items up to **1 KB**.

    - Example: A 2.5 KB item rounded up to 3 KB requires $3\text{ WCUs}$.

    - Transactional Writes (`TransactWriteItems`) cost **$2\times\text{ WCU}$** ($2\text{ WCUs}$ per 1 KB).

- **RCU (Read Capacity Unit):**

    - $1\text{ RCU} = 1\text{ strongly consistent read/sec}$ for items up to **4 KB**.

    - $1\text{ RCU} = 2\text{ eventually consistent reads/sec}$ ($0.5\text{ RCU}$ per 4 KB read).

    - Transactional Reads (`TransactGetItems`) cost **$2\times\text{ RCU}$** ($2\text{ RCUs}$ per 4 KB).

### 4. Overcoming Hot Partitions: Write Sharding (Salting)

- When high-volume writes target a single partition key (e.g., date `2026-09-22` or candidate ID in a voting system), the partition exceeds the 1,000 WCU ceiling and throttles.

- **Solution (Write Sharding / Key Salting):** Append a deterministic or random suffix to the PK:

    $$\text{PK} = \text{"VOTE#" } + \text{candidateId} + \text{"#" } + \text{random}(0, N-1)$$

- **Trade-off:** Writes are distributed across $N$ physical partitions; reading the aggregate requires $N$ parallel queries across all salted variations.

### 5. Overcoming the 400 KB Item Limit

1. **S3 Claim-Check Pattern:** Store payload in S3; store the S3 object key/URI and metadata in DynamoDB.

2. **Payload Compression:** Compress JSON into binary GZIP/Brotli strings before persisting (often yields 70–80% size reduction).

3. **Vertical Partitioning (Item Splitting):** Split infrequently accessed wide attributes (e.g., descriptions, audit blobs) into separate items using distinct Sort Keys (`ITEM#123` with `SK: METADATA` vs. `SK: BLOB_PART1`).

### 6. Caching & Cost Optimization

- **DynamoDB Accelerator (DAX):** In-memory, write-through caching cluster providing microsecond response times without application code rewrite. Best for read-heavy, spiky workloads.

- **Sparse Indexes:** Only items containing the GSI key are projected, cutting index storage and write replication costs.

- **Projection Expressions:** Avoid `ALL` attribute projection on GSIs; project only keys and necessary lookup attributes to minimize WCU consumption during updates.

## Code Snippets / Examples

### 1. Write Sharding / Salting Implementation (TypeScript)

```typescript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, PutCommand, QueryCommand } from "@aws-sdk/lib-dynamodb";

const client = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const NUM_SHARDS = 10;

// Write with random shard (0 to 9) to distribute throughput across partitions
export async function recordVote(candidateId: string, voterId: string) {
  const shardId = Math.floor(Math.random() * NUM_SHARDS);
  const saltedPK = `CANDIDATE#${candidateId}#SHARD#${shardId}`;

  await client.send(new PutCommand({
    TableName: "VotesTable",
    Item: {
      PK: saltedPK,
      SK: `VOTER#${voterId}`,
      timestamp: Date.now(),
    },
  }));
}

// Read requires querying all N shards and aggregating the result
export async function getVoteCount(candidateId: string): Promise<number> {
  const queries = Array.from({ length: NUM_SHARDS }, (_, i) => {
    return client.send(new QueryCommand({
      TableName: "VotesTable",
      KeyConditionExpression: "PK = :pk",
      ExpressionAttributeValues: { ":pk": `CANDIDATE#${candidateId}#SHARD#${i}` },
      Select: "COUNT",
    }));
  });

  const results = await Promise.all(queries);
  return results.reduce((sum, res) => sum + (res.Count || 0), 0);
}
```

### 2. S3 Claim-Check Pattern for Items > 400 KB

```typescript
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb";

const s3 = new S3Client({});
const docClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));

export async function saveRecord(id: string, largePayload: object) {
  const serialized = JSON.stringify(largePayload);
  const byteSize = Buffer.byteLength(serialized, "utf8");

  if (byteSize > 350 * 1024) { // Approaching 400 KB limit -> offload to S3
    const s3Key = `payloads/${id}.json`;
    await s3.send(new PutObjectCommand({
      Bucket: "large-record-store",
      Key: s3Key,
      Body: serialized,
      ContentType: "application/json",
    }));

    await docClient.send(new PutCommand({
      TableName: "EntityTable",
      Item: {
        PK: `ENTITY#${id}`,
        SK: "METADATA",
        isPayloadOffloaded: true,
        s3Pointer: s3Key,
      },
    }));
  } else {
    await docClient.send(new PutCommand({
      TableName: "EntityTable",
      Item: {
        PK: `ENTITY#${id}`,
        SK: "METADATA",
        isPayloadOffloaded: false,
        data: largePayload,
      },
    }));
  }
}
```

## Related Topics

- [[DynamoDB Capacity Modes - Provisioned with Auto Scaling vs. On-Demand]]

- [[DynamoDB Single-Table Design - Inventory Management Scenario]]

- [[Debugging DynamoDB Hot Partitions & Hot Keys]]

- [[Adding a Global Secondary Index (GSI) to a Large DynamoDB Table]]

- [[Scaling DynamoDB Streams - High-Volume Event Processing]]

## Tags

#fullstack #interview #aws #dynamodb #nosql #database-internals #system-design

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups