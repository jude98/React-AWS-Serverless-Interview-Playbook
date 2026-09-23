# Database Partitioning vs. Sharding

## Key Concepts

- **Core Distinction:** Partitioning is the general term for dividing a database into distinct subsets; **sharding is a specific form of horizontal partitioning across multiple distinct physical machines or instances**.
    
      
    
- **Rule of Thumb:** All sharding is partitioning, but not all partitioning is sharding.
    
      
    
- **Scope & Boundary:**
    
      
    - **Partitioning:** Typically occurs on a **single database engine/server**. The database engine manages division natively across files, tablespaces, or disks.
        
          
        
    - **Sharding:** Distributes data subsets across **multiple independent database nodes/servers**, each with its own CPU, memory, and storage (a "shared-nothing" architecture).
        
          
        
- **Partitioning Types:**
    
      
    - **Horizontal Partitioning:** Splitting rows of a table across subsets using a partition key (e.g., range by date, list by region, or hash).
        
          
        
    - **Vertical Partitioning:** Splitting columns of a table into distinct tables (e.g., moving large `BLOB` / `text` columns or rarely accessed attributes to a separate table to optimize cache locality).
        
          
        
- **Sharding Mechanics:** Always horizontal. Rows are routed to specific server instances using a **Shard Key** via algorithms like Consistent Hashing, Directory-Based Routing, or Range Sharding.
    
      
    
- **Operational Complexity:** Partitioning improves query planning and disk I/O on a single node without changing application topology; sharding requires distributed query routing, distributed transactions (2PC/Saga), rebalancing, and cross-shard join coordination.
    
      
    

## Common Interview Questions

- What is the difference between partitioning and sharding, and when does a system outgrow single-node partitioning?
    
      
    
- How does horizontal partitioning differ from vertical partitioning?
    
      
    
- What are the major routing strategies for sharding (Hash vs. Range vs. Directory-based)?
    
      
    
- What is a "Hot Shard" (or shard skew), and how do you prevent it?
    
      
    
- Why are cross-shard joins and distributed transactions considered anti-patterns in sharded databases?
    
      
    
- How does Consistent Hashing minimize data movement when adding or removing shards in a cluster?
    
      
    

## Strong Answers / Talking Points

### 1. Partitioning vs. Sharding Comparison

|**Attribute**|**Database Partitioning (Single Node)**|**Database Sharding (Distributed)**|
|---|---|---|
|**Physical Topology**|**Single server / instance** (data split across tablespaces, disks, or internal tables).|**Multiple independent servers / clusters** (shared-nothing architecture).|
|**Scaling Target**|**Vertical Scaling ($Scale-Up$):** Optimizes disk I/O, index tree size, and scan speeds on one machine.|**Horizontal Scaling ($Scale-Out$):** Bypasses single-server memory, CPU, and disk storage physical limits.|
|**Management**|Handled natively by the database engine (e.g., PostgreSQL declarative partitioning, MySQL partitions).|Handled by distributed DB engine (CockroachDB, Spanner, Vitess) or custom application routing.|
|**Transactions & ACID**|Full ACID guarantees maintained natively; locks and constraints function normally.|Distributed ACID requires Two-Phase Commit (2PC) or Saga patterns; often trades consistency for availability (CAP/PACELC).|
|**Foreign Keys & Joins**|Supported across partitions with zero network latency.|**Cross-shard joins are expensive and discouraged**; foreign keys across shards are rarely supported natively.|
|**Failure Domain**|Single point of failure: If the host machine goes down, all partitions are unavailable.|Partial availability: If Shard 3 fails, data on Shards 1, 2, and 4 remains accessible (AP-leaning resilience).|

### 2. Forms of Partitioning: Horizontal vs. Vertical

- **Horizontal Partitioning:**
    
      
    - Keeps identical table schema, splits rows into chunks based on a partition key.
        
          
        
    - _Example:_ A table of 500 million logs partitioned by year (`logs_2024`, `logs_2025`, `logs_2026`).
        
          
        
    - _Benefit (Partition Pruning):_ The query planner evaluates `WHERE created_at >= '2026-01-01'` and scans **only** the `logs_2026` partition, bypassing gigabytes of older index pages.
        
          
        
- **Vertical Partitioning:**
    
      
    - Splits columns into distinct tables sharing the same primary key.
        
          
        
    - _Example:_ Separating a `users` table into `users_auth` (`id, email, password_hash`) and `users_profile` (`id, bio_text, avatar_blob, preferences_json`).
        
          
        
    - _Benefit:_ Keeps high-frequency read rows compact in memory (RAM buffer pool) while pushing heavyweight, rarely queried blobs to separate storage.
        
          
        

### 3. Sharding Strategies & Key Selection

- **Hash-Based Sharding:**
    
      
    - Formula: $\text{Shard ID} = \text{hash}(\text{shard\_key}) \pmod N$ (or via Consistent Hashing ring).
        
          
        
    - _Pros:_ Distributes writes and reads evenly across all nodes.
        
          
        
    - _Cons:_ Range queries require broadcasting requests to every single shard (scatter-gather).
        
          
        
- **Range-Based Sharding:**
    
      
    - Routes data based on key ranges (e.g., User IDs 1–1,000,000 to Node 1, 1,000,001–2,000,000 to Node 2).
        
          
        
    - _Pros:_ Range scans (`BETWEEN A AND B`) are co-located on a single shard.
        
          
        
    - _Cons:_ Prone to **Hot Shards** if monotonic keys (like autoincrement IDs or timestamps) direct 100% of current writes to the latest shard.
        
          
        
- **Directory / Lookup-Based Sharding:**
    
      
    - A centralized metadata mapping table tracks which tenant/customer ID maps to which shard node.
        
          
        
    - _Pros:_ Highly flexible for multi-tenant architectures (e.g., Enterprise customer pinned to dedicated Shard 5).
        
          
        
    - _Cons:_ Lookup service becomes a potential bottleneck and single point of failure without caching.
        
          
        

### 4. When to Use Which: Architectural Decision Path

1. **Optimize Existing Queries First:** Add composite indexes, read replicas, and caching (Redis) before partitioning or sharding.
    
      
    
2. **Move to Single-Node Partitioning When:**
    
      
    - Single tables exceed 50–100 million rows.
        
          
        
    - B-Tree indexes no longer fit inside the database buffer pool / RAM.
        
          
        
    - You need quick, zero-cost data archiving (e.g., `ALTER TABLE DROP PARTITION logs_2023` takes milliseconds vs. a massive, lock-heavy `DELETE WHERE ...`).
        
          
        
3. **Move to Distributed Sharding When:**
    
      
    - Write throughput saturates the largest available cloud instance CPU/IOPS limits.
        
          
        
    - Total dataset size exceeds single-node storage limits ($> 10\text{ TB}$ on transactional systems).
        
          
        
    - Regulatory requirements dictate physical data residency by geography (e.g., EU data stored on EU-based shards).
        
          
        

## Code Snippets / Examples

```sql
-- ============================================================================
-- 1. Single-Node Declarative Partitioning (PostgreSQL)
-- All partitions live on the SAME database instance
-- ============================================================================

-- Master table partitioned by RANGE on created_at
CREATE TABLE orders (
    order_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    amount_cents INT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (order_id, created_at)
) PARTITION BY RANGE (created_at);

-- Partitions (Child tables managed internally by Postgres engine)
CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01 00:00:00+00') TO ('2026-01-01 00:00:00+00');

CREATE TABLE orders_2026 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01 00:00:00+00') TO ('2027-01-01 00:00:00+00');

-- Query Planner performs Partition Pruning: skips orders_2025 completely
EXPLAIN SELECT * FROM orders 
WHERE created_at >= '2026-06-01' AND created_at < '2026-07-01';
```

```typescript
// ============================================================================
// 2. Application-Level Sharding Router (Distributed Nodes)
// Directs queries across separate physical connection pools
// ============================================================================
import crypto from "node:crypto";

interface ShardConnection {
  host: string;
  query(sql: string, params: any[]): Promise<any>;
}

export class ShardedDatabaseRouter {
  private shards: ShardConnection[];

  constructor(shardNodes: ShardConnection[]) {
    this.shards = shardNodes;
  }

  // Consistent hash mapping: maps customerId -> target physical shard
  public getShard(customerId: string): ShardConnection {
    const hash = crypto.createHash("md5").update(customerId).digest("hex");
    const intVal = parseInt(hash.slice(0, 8), 16);
    const shardIndex = intVal % this.shards.length;
    return this.shards[shardIndex];
  }

  public async insertOrder(order: { id: string; customerId: string; amount: number }) {
    // Write routes strictly to the specific server hosting that customer's shard
    const targetShard = this.getShard(order.customerId);
    return targetShard.query(
      "INSERT INTO orders (id, customer_id, amount) VALUES ($1, $2, $3)",
      [order.id, order.customerId, order.amount]
    );
  }
}
```

## Related Topics

- [[Database Partitioning vs. Sharding|Database Replication Strategies]]
    
      
    
- [[Database Partitioning vs. Sharding|Consistent Hashing]]
    
      
    
- [[CAP Theorem]]
    
      
    
- [[High-Scale Data Table Architecture Handling Millions of Records|Database Indexing & Cursor vs Offset Pagination]]
    
      
    
- [[Distributed Transactions & Event-Driven Architecture - Sagas, 2PC, Resilience & Messaging Selection|Distributed Transactions and 2PC (Two-Phase Commit)]]
    
      
    

## Tags

#fullstack #interview #database #system-design #distributed-systems #scaling #sql

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups