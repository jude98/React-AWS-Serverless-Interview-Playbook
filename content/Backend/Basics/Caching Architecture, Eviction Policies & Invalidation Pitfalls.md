# Caching Architecture, Eviction Policies & Invalidation Pitfalls

## Key Concepts

- **Caching Objective:** Trade memory overhead for sub-millisecond retrieval latency, protecting primary data stores (e.g., PostgreSQL, DynamoDB) from query amplification and CPU bottlenecks.
    
      
    
- **Cache Topologies:**
    
      
    - **In-Memory / Local:** Stored directly inside application process memory (e.g., Node.js Heap, LRU-cache). Ultra-fast (nanoseconds), but isolated to specific instances and vulnerable to memory thrashing/cold starts across autoscaling fleets.
        
          
        
    - **Distributed / Remote:** Dedicated external storage cluster (e.g., Redis, Memcached) shared across all application instances. Guarantees global consistency across the fleet at the cost of a network hop (1–5ms).
        
          
        
- **Core Caching Patterns:** Cache-Aside (Lazy Loading), Read-Through, Write-Through, and Write-Behind (Write-Back).
    
      
    
- **Eviction vs. Expiration:**
    
      
    - _Expiration (TTL):_ Deletes data when its lifespan expires.
        
          
        
    - _Eviction:_ Proactively purges valid keys to free up space when memory limits (`maxmemory`) are reached.
        
          
        
- **Classic Caching Anomalies:** Cache Stampede (Dog-piling / Thundering Herd), Cache Penetration, Cache Breakdown, Hot Key Saturation, and Dual-Write Drift (Consistency).
    
      
    

## Common Interview Questions

- What are the operational trade-offs between Cache-Aside and Write-Through caching?
    
      
    
- In Write-Behind (Write-Back) caching, how do you handle potential data loss during node crashes?
    
      
    
- How do LRU, LFU, and FIFO eviction algorithms differ in implementation complexity and use cases?
    
      
    
- What is a Cache Stampede (Thundering Herd), and what strategies mitigate it under high concurrency?
    
      
    
- How do you resolve the "Dual-Write Problem" when updating both a cache and a database?
    
      
    
- How do you detect, scale, and protect against "Hot Keys" in distributed clusters like Redis?
    
      
    

## Strong Answers / Talking Points

### 1. Caching Strategies & Write/Read Patterns

|**Pattern**|**Read Path**|**Write Path**|**Pros**|**Cons**|
|---|---|---|---|---|
|**Cache-Aside (Lazy Loading)**|App checks cache $\rightarrow$ On miss, app reads DB and writes result to cache.|App updates DB directly, then **invalidates (deletes)** key from cache.|Cache stores only requested data; node failure doesn't crash the write pipeline.|Stale reads possible between DB write and cache purge; read latency penalty on cold cache miss.|
|**Read-Through**|App requests data from cache provider. On miss, **cache infrastructure itself** queries DB, populates itself, and returns data.|Same as Cache-Aside or Write-Through.|Simplifies application code; decouples DB query orchestration from app services.|Requires custom cache plugins/middleware (e.g., Redis plugins, DAX).|
|**Write-Through**|Normal cache read.|App writes to cache; **cache synchronously writes to DB** before returning success to app.|Cache is never stale; high data consistency across reads.|High write latency (waits for both cache and DB writes to complete).|
|**Write-Behind (Write-Back)**|Normal cache read.|App writes strictly to cache/queue; cache **asynchronously batches writes to DB** in the background.|Extreme write performance; absorbs massive traffic spikes.|**Data loss risk:** If the cache crashes before dirty writes flush to disk/DB, data is lost permanently.|

### 2. Cache Eviction Policies

When the cache hits memory boundaries (e.g., Redis `maxmemory-policy`), an eviction policy decides what to purge:

  

- **LRU (Least Recently Used):**
    
      
    - Purges keys that have not been requested for the longest time.
        
          
        
    - _Mechanism:_ Doubly linked list + Hash map ($O(1)$ operations).
        
          
        
    - _Best For:_ General web workloads where recent access predicts future access.
        
          
        
- **LFU (Least Frequently Used):**
    
      
    - Purges keys with the lowest hit counter over time.
        
          
        
    - _Best For:_ Long-tail catalogs where popular assets must stay pinned regardless of short-lived bursts.
        
          
        
- **FIFO (First-In, First-Out):**
    
      
    - Purges the oldest keys by creation order, ignoring access frequency/recency.
        
          
        
    - _Best For:_ Sequential time-series processing where older data loses relevance quickly.
        
          
        
- **Random / TTL-based (`volatile-ttl`):**
    
      
    - Purges random keys or prioritizes keys closest to their expiration timestamp.
        
          
        

### 3. Critical Caching Pathologies & Production Fixes

#### A. Cache Stampede (Thundering Herd / Breakdown)

- **Problem:** A high-traffic key (e.g., homepage banner or live sports score) expires or gets invalidated. Hundreds of concurrent requests experience a cache miss simultaneously and hammer the primary database at the same instant, leading to thread exhaustion and cascade failure.
    
      
    
- **Mitigations:**
    
      
    1. **Distributed Mutex (Locking):** Only the first worker that misses the cache acquires a lock (via `SET NX EX`) to query the database and rebuild the cache; all other requests wait or return slightly stale data.
        
          
        
    2. **Probabilistic Early Recomputation (XFetch Algorithm):** Recompute the key in the background _before_ it formally expires based on the cost of computing it and remaining TTL.
        
          
        
    3. **Stale-While-Revalidate:** Return stale data immediately while triggering an asynchronous background job to fetch fresh state.
        
          
        

#### B. The Dual-Write Consistency Problem

- **Problem:** Updating the DB and updating the cache concurrently can lead to out-of-order execution, leaving stale data permanently in the cache.
    
      
    - _Anti-Pattern:_ Updating DB, then calling `cache.set(key, newValue)`. Two concurrent updates ($T_1$ and $T_2$) can interleave: $T_1$ writes DB $\rightarrow$ $T_2$ writes DB $\rightarrow$ $T_2$ writes Cache $\rightarrow$ $T_1$ writes Cache (Cache now permanently holds stale $T_1$ data).
        
          
        
- **Solution:**
    
      
    - **Cache Eviction on Write:** Update the DB first, then **delete** the cache key (`cache.del(key)`). Next read repopulates from the DB.
        
          
        
    - **Change Data Capture (CDC):** Use tools like Debezium + Kafka tailing the PostgreSQL write-ahead log (WAL) to invalidate cache asynchronously.
        
          
        

#### C. Hot Key Saturation

- **Problem:** In a Redis cluster sharded by hash slots, an ultra-popular key lives on a single node. If that key receives 100k req/sec, that single shard reaches 100% CPU capacity while other cluster nodes sit idle.
    
      
    
- **Mitigations:**
    
      
    1. **Multi-Key Salted Sharding:** Append random suffixes to the key (`product:42:shard_1`, `product:42:shard_2`) and randomly balance client reads across shards.
        
          
        
    2. **Local In-Process Layer (L1/L2 Cache):** Place a tiny in-memory LRU cache inside the application instances (L1, TTL 2–5 seconds) to absorb 95% of hits before reaching Redis (L2).
        
          
        

## Code Snippets / Examples

```TypeScript
// ============================================================================
// 1. Production Cache-Aside Pattern with Distributed Locking (Anti-Stampede)
// ============================================================================

interface CacheClient {
  get(key: string): Promise<string | null>;
  set(key: string, value: string, ttlSeconds: number): Promise<void>;
  del(key: string): Promise<void>;
  // Redis SET key value NX EX (Atomic acquire lock)
  acquireLock(lockKey: string, ttlSeconds: number): Promise<boolean>;
  releaseLock(lockKey: string): Promise<void>;
}

export class ProductService {
  constructor(
    private readonly cache: CacheClient,
    private readonly db: { findProduct: (id: string) => Promise<any> }
  ) {}

  async getProduct(productId: string) {
    const cacheKey = `product:${productId}`;
    const lockKey = `lock:${cacheKey}`;

    // 1. Try reading from cache
    const cachedData = await this.cache.get(cacheKey);
    if (cachedData) {
      return JSON.parse(cachedData);
    }

    // 2. Cache Miss: Mitigate stampede via distributed lock
    const acquired = await this.cache.acquireLock(lockKey, 5); // 5 sec lock TTL

    if (!acquired) {
      // Another thread is already regenerating the cache. 
      // Back off briefly and re-read cache instead of hitting the DB.
      await new Promise((resolve) => setTimeout(resolve, 100));
      const retryCached = await this.cache.get(cacheKey);
      if (retryCached) return JSON.parse(retryCached);
    }

    try {
      // 3. Winner queries the primary database
      const product = await this.db.findProduct(productId);
      if (!product) return null;

      // 4. Populate cache with TTL + Jitter (avoids simultaneous expiration)
      const jitter = Math.floor(Math.random() * 60);
      await this.cache.set(cacheKey, JSON.stringify(product), 3600 + jitter);

      return product;
    } finally {
      // 5. Release lock
      if (acquired) {
        await this.cache.releaseLock(lockKey);
      }
    }
  }

  // Dual-Write Fix: Invalidate instead of update
  async updateProductPrice(productId: string, newPrice: number) {
    // Write directly to DB first
    await this.updatePriceInDatabase(productId, newPrice);

    // Invalidate cache immediately to force fresh re-read
    await this.cache.del(`product:${productId}`);
  }

  private async updatePriceInDatabase(id: string, price: number) {
    // Database write logic
  }
}
```

## Related Topics

- [[CAP Theorem]]
    
      
    
- [[HTTP Methods, CORS, Status Codes & Caching]]
    
      
    
- [[Database Partitioning vs. Sharding|Database Replication Strategies]]
    
      
    
- [[CAP Theorem|Eventual Consistency]]
    
      
    
- [[Distributed Transactions & Event-Driven Architecture - Sagas, 2PC, Resilience & Messaging Selection|Distributed Locking (Redlock, Optimistic Locking)]]
    
      
    

## Tags

#fullstack #interview #caching #redis #system-design #distributed-systems #performance

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups