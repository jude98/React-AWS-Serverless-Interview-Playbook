

## Key Concepts

- **ACID** defines the guarantees provided by database transactions to ensure data validity despite software crashes, hardware failures, and concurrent access.
    
      
    
- **Atomicity ($A$):** All-or-nothing execution. If any operation in a transaction fails, the entire transaction is rolled back via undo logs or write-ahead logging (WAL).
    
      
    
- **Consistency ($C$):** Invariants and schema constraints (foreign keys, uniqueness, check constraints) remain valid before and after the transaction commits.
    
      
    
- **Isolation ($I$):** Concurrent transactions execute without cross-talk or interfering with each other's intermediate state.
    
      
    
- **Durability ($D$):** Once committed, writes survive any subsequent crash or restart (persisted to non-volatile storage / synced via WAL).
    
      
    
- **Isolation is a spectrum:** Full isolation (serializability) eliminates anomalies but tanks throughput; databases provide configurable isolation levels to balance consistency against concurrency.
    
      
    

## Common Interview Questions

- What does each letter in ACID guarantee, and how does the "C" in ACID differ from the "C" in CAP?
    
      
    
- What are read phenomena (dirty read, non-repeatable read, phantom read), and what causes them?
    
      
    
- What are the ANSI SQL isolation levels, and which read phenomena does each level prevent?
    
      
    
- What is Snapshot Isolation (MVCC), and why is it not identical to ANSI Serializable?
    
      
    
- What is Write Skew, and which isolation level is required to prevent it?
    
      
    
- How do real-world databases (e.g., PostgreSQL, MySQL InnoDB) implement isolation under the hood?
    
      
    

## Strong Answers / Talking Points

### 1. The Core ACID Definitions

- **Atomicity:** Managed via the undo log / WAL. Changes are staged in memory; if an unhandled exception or abort occurs, previous states are restored.
    
      
    
- **Consistency:** Unlike CAP consistency (linearizability across nodes), ACID consistency is application/schema correctness. If an account balance cannot go below zero, a transaction violating this is aborted.
    
      
    
- **Isolation:** Defined formally by the ANSI SQL-92 standard via the specific concurrent read anomalies it prevents.
    
      
    
- **Durability:** Ensured via synchronous disk flushes (`fsync`) of the Write-Ahead Log (WAL) before acknowledging commit success to the client.
    
      
    

### 2. Read Phenomena / Anomalies

> [!WARNING] The Distinction Between Non-Repeatable Read and Phantom Read
> 
> A **non-repeatable read** affects _existing rows_ (updates/deletes). A **phantom read** affects _predicate query ranges_ (new rows inserted matching a `WHERE` clause).
> 
>   

- **Dirty Read:**
    
      
    - Transaction A modifies a row without committing.
        
          
        
    - Transaction B reads that uncommitted modification.
        
          
        
    - Transaction A rolls back. Transaction B has processed phantom, invalid state.
        
          
        
- **Non-Repeatable Read (Fuzzy Read):**
    
      
    - Transaction A reads a row.
        
          
        
    - Transaction B updates or deletes that same row and commits.
        
          
        
    - Transaction A re-reads the row and observes altered data values.
        
          
        
- **Phantom Read:**
    
      
    - Transaction A executes a range query (e.g., `SELECT COUNT(*) WHERE status = 'pending'`).
        
          
        
    - Transaction B inserts a new row matching that filter and commits.
        
          
        
    - Transaction A re-runs the exact same query and observes additional "phantom" rows.
        
          
        

### 3. ANSI SQL Isolation Levels vs. Anomalies

|**Isolation Level**|**Dirty Read**|**Non-Repeatable Read**|**Phantom Read**|**Implementation Mechanism**|
|---|---|---|---|---|
|**Read Uncommitted**|Allowed|Allowed|Allowed|Reads without locks; writes take exclusive row locks.|
|**Read Committed**|**Prevented**|Allowed|Allowed|Reads take short-lived shared locks or read from a snapshot created at _statement_ start (PostgreSQL/MySQL default).|
|**Repeatable Read**|**Prevented**|**Prevented**|Allowed*|Reads take shared locks until transaction end, or read from a snapshot created at _transaction_ start (MVCC).|
|**Serializable**|**Prevented**|**Prevented**|**Prevented**|Strict two-phase locking (2PL), predicate locks, or Serializable Snapshot Isolation (SSI).|

_*Note: In MySQL InnoDB, Repeatable Read also prevents phantom reads for regular (`SELECT`) reads via MVCC and for locking reads via Next-Key Locking (gap locks)._

  

### 4. Advanced Anomaly: Write Skew

- Occurs under **Snapshot Isolation / Repeatable Read** where two transactions concurrently read overlapping data, check an invariant, and update distinct, non-overlapping rows that mutually violate the invariant.
    
      
    
- _Example:_ On-call doctor problem. Invariant: at least 1 doctor must be on call. Both doctors check the DB simultaneously (count = 2), and both submit a request to go off-call for themselves in separate transactions. Both commit; 0 doctors remain on call.
    
      
    
- _Fix:_ Must use **Serializable** isolation, or explicit locking (`SELECT ... FOR UPDATE`).
    
      
    

## Code Snippets / Examples



```SQL
-- Checking and setting isolation levels in PostgreSQL
SHOW default_transaction_isolation;

-- 1. Handling Write Skew / Concurrent Updates with Explicit Locking
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Lock the target row exclusively to prevent non-repeatable read or lost update
SELECT balance 
FROM accounts 
WHERE account_id = 42 
FOR UPDATE;

-- Perform business logic in application runtime, then update
UPDATE accounts 
SET balance = balance - 100 
WHERE account_id = 42;

COMMIT;

-- 2. Handling Range Queries under Serializable
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- SSI detects read/write dependencies across overlapping predicate queries
SELECT COUNT(*) 
FROM doctors 
WHERE is_on_call = TRUE;

-- If another concurrent transaction alters the predicate set, 
-- PostgreSQL throws: ERROR: could not serialize access due to read/write dependencies among transactions (SQLSTATE 40001)
UPDATE doctors 
SET is_on_call = FALSE 
WHERE doctor_id = 101;

COMMIT;
```

## Related Topics

- [[CAP Theorem]]
    
      
    
- [[MVCC (Multi-Version Concurrency Control)]]
    
      
    
- [[Database Locking Mechanisms (Pessimistic vs Optimistic)]]
    
      
    
- [[Write-Ahead Logging (WAL)]]
    
      
    
- [[Distributed Transactions and 2PC (Two-Phase Commit)]]
    
      
    

## Tags

#fullstack #interview #database #system-design #acid #sql

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups