# CAP Theorem

## Key Concepts

- Formulated by Eric Brewer; states that a distributed data store can simultaneously provide at most two out of three guarantees: **Consistency**, **Availability**, and **Partition Tolerance**.

- In any distributed network, network partitions (dropped messages, network latency spikes, node isolation) are inevitable.

- Therefore, the real architectural choice is not "pick two out of three," but **how the system behaves during a network partition: Consistency vs. Availability (CP vs. AP)**.

- **Consistency ($C$):** Linearizability / Single-system image. Every read receives the most recent write or an error.

- **Availability ($A$):** Every non-failing node returns a non-error response for every request (no guarantee it contains the latest write).

- **Partition Tolerance ($P$):** The system continues to operate despite arbitrary message loss or communication delay across nodes.

- When no partition exists ($P$ is healthy), the trade-off shifts to **Latency vs. Consistency** (formalized by the [[CAP Theorem|PACELC Theorem]]).

## Common Interview Questions

- What is the CAP theorem, and why is "pick two of three" technically misleading?

- How does "Consistency" in CAP differ from the "C" in ACID?

- What happens to a CP system versus an AP system when a network partition occurs?

- Can you classify databases like PostgreSQL, Cassandra, MongoDB, and DynamoDB under CAP?

- What is the PACELC theorem, and how does it extend CAP?

- How do consensus algorithms like Raft or Paxos relate to CAP guarantees?

## Strong Answers / Talking Points

### 1. The Core Meaning of C, A, and P

> [!NOTE] Precision Matters
> 
> Always clarify the precise academic definitions of each letter, as candidates frequently confuse them with colloquial system design terms.
> 
>   

- **Consistency ($C$):**

    - Academic definition: **Linearizability**.

    - Contrast with ACID: ACID consistency means data integrity rules/invariants (e.g., foreign keys, check constraints) are preserved. CAP consistency strictly means read-after-write recency across replicas.

- **Availability ($A$):**

    - Academic definition: Every request to a healthy node must yield a non-error response.

    - Contrast with SLA availability: High availability (99.999% uptime) is an operational metric. CAP availability explicitly disallows returning a `500 Internal Server Error` or a timeout during a partition.

- **Partition Tolerance ($P$):**

    - You cannot choose "CA" over wide-area networks; physical networks will eventually partition.

    - Rejecting $P$ means assuming network communication never fails, which is impossible in real-world distributed infrastructure.

### 2. CP vs. AP: Architectural Scenarios

|**Attribute**|**CP (Consistency + Partition Tolerance)**|**AP (Availability + Partition Tolerance)**|
|---|---|---|
|**Partition Behavior**|Blocks or fails reads/writes to minority partition to prevent stale/split-brain state.|Accepts reads/writes on all partitions; returns potentially stale data.|
|**Data Conflict**|Prevented at ingestion time via synchronous quorum/locking.|Reconciled later via eventual consistency, CRDTs, or vector clocks.|
|**Typical Databases**|MongoDB (majority write concern), HBase, Spanner, Redis (cluster mode).|Cassandra, DynamoDB, CouchDB, Riak.|
|**Ideal Use Case**|Financial ledgers, inventory counts, ticketing systems, auth sessions.|Social media feeds, analytics counters, shopping cart drafts, chat history.|

### 3. When to Choose What (Trade-Off Framing)

- **Choose CP when data divergence causes permanent business loss:**

    - _Example:_ Financial transactions, payment processing, ledger balances. It is better to fail the request with a timeout or retry error than to allow double-spending.

- **Choose AP when customer experience degrades more from downtime than temporary staleness:**

    - _Example:_ Product reviews, social media feeds, session carts. Users prefer viewing data that is a few seconds old over an application crash or blocked request.

- **Beyond CAP (PACELC):**

    - If Partition ($P$): Trade-off between Availability ($A$) and Consistency ($C$).

    - Else ($E$): Trade-off between Latency ($L$) and Consistency ($C$).

    - _Example:_ DynamoDB is PA/EL (favors availability during partitions, favors low latency during normal operation).

## Code Snippets / Examples

```typescript
// Conceptual demonstration: Node behavior under a Network Partition

type NodeState = {
  data: string;
  isPartitionedFromQuorum: boolean;
};

// CP Node: Prioritizes consistency, rejects operations if isolated
function handleReadCP(node: NodeState): { status: number; data?: string; error?: string } {
  if (node.isPartitionedFromQuorum) {
    // Fail fast rather than returning potentially stale data
    return { status: 503, error: "Service Unavailable: Quorum unreachable" };
  }
  return { status: 200, data: node.data };
}

// AP Node: Prioritizes availability, serves stale data regardless of isolation
function handleReadAP(node: NodeState): { status: number; data: string; warning?: string } {
  if (node.isPartitionedFromQuorum) {
    // Serve local copy, accept staleness
    return { status: 200, data: node.data, warning: "Data may be stale" };
  }
  return { status: 200, data: node.data };
}
```

## Related Topics

- [[ACID Properties & Transaction Isolation Levels]]

- [[Database Partitioning vs. Sharding]]

- [[Distributed Transactions & Event-Driven Architecture - Sagas, 2PC, Resilience & Messaging Selection]]

- [[AWS Serverless & Event-Driven Architecture (EDA)]]

- [[Amazon DynamoDB -  Architecture, Data Modeling & Scaling]]

## Tags

#fullstack #interview #system-design #distributed-systems #cap-theorem

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups