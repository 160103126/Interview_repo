# Redis & Caching — Theory & Patterns

> Distributed caching strategies, Redis internals, eviction policies, and cache invalidation techniques. Essential for backend scaling and system design.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is Caching? Why do we use it?

**Answer:**

Caching is the process of storing copies of frequently accessed data in a temporary, high-speed storage layer (usually RAM) to serve future requests faster.

**Why use it?**
1. **Reduce Latency:** Reading from RAM is orders of magnitude faster than reading from a disk or making a network call.
2. **Reduce Load:** Protects the primary database from being overwhelmed by repetitive queries.
3. **Save Cost:** Computing a complex result once and caching it is cheaper than recomputing it thousands of times.

---

### Q2: What is Redis? How is it different from Memcached?

**Answer:**

Redis (Remote Dictionary Server) is an open-source, in-memory key-value data store.

| Feature | Memcached | Redis |
|---------|-----------|-------|
| **Data Structures** | Simple Strings only | Strings, Hashes, Lists, Sets, Sorted Sets, Bitmaps, HyperLogLog |
| **Persistence** | No (RAM only) | Yes (RDB snapshots, AOF logs) |
| **Replication** | No native replication | Native Master-Slave replication |
| **Transactions** | No | Yes (MULTI/EXEC) |
| **Pub/Sub** | No | Yes |
| **Use case** | Simple object caching | Caching, Message Queuing, Leaderboards, Session Store |

---

### Q3: Explain the common Cache Eviction Policies.

**Answer:**

When a cache fills up, it must evict old data to make room for new data. 

- **LRU (Least Recently Used):** Evicts the item that hasn't been accessed for the longest time. (Most common and effective).
- **LFU (Least Frequently Used):** Evicts the item that has been accessed the fewest number of times overall.
- **FIFO (First In, First Out):** Evicts the oldest item, regardless of how often it's used.
- **Random:** Evicts a random item.
- **TTL (Time To Live):** Not an eviction policy per se, but data expires automatically after a set duration.

*Redis default:* `noeviction` (returns an error on write if full). Usually changed to `allkeys-lru` or `volatile-lru`.

---

## 🟡 Medium (Intermediate)

### Q4: Explain the Cache-Aside, Read-Through, and Write-Through caching patterns.

**Answer:**

**1. Cache-Aside (Lazy Loading)**
- The application is responsible for reading/writing to the cache and the database.
- *Read Flow:* Check cache -> If miss, read DB -> Write to cache -> Return.
- *Write Flow:* Write to DB -> Delete (invalidate) cache.
- *Pros:* Resilient (if cache fails, DB still works). Cache only holds requested data.

**2. Read-Through**
- The application asks the cache for data. The cache itself fetches from the DB on a miss.
- *Pros:* Application code is simpler.

**3. Write-Through**
- The application writes to the cache. The cache synchronously writes to the DB before returning success.
- *Pros:* Cache and DB are always strictly consistent.
- *Cons:* Slower writes (two operations required).

**4. Write-Behind (Write-Back)**
- Application writes to cache and returns immediately. Cache asynchronously writes to the DB later.
- *Pros:* Extremely fast writes.
- *Cons:* High risk of data loss if the cache crashes before writing to the DB.

---

### Q5: What is the "Cache Stampede" (Thundering Herd) problem and how do you solve it?

**Answer:**

**Problem:** A highly popular cache key (e.g., the homepage data) expires or is evicted. Suddenly, 10,000 concurrent requests miss the cache. All 10,000 requests query the database simultaneously to compute the data, crashing the database.

**Solutions:**
1. **Mutex Locks:** When a cache miss occurs, the application attempts to acquire a distributed lock for that key. Only the thread that gets the lock queries the DB and updates the cache. The others wait/poll.
2. **Probabilistic Early Expiration (PERF):** A background thread (or the application itself based on a probability formula) updates the cache *before* it actually expires.
3. **Stale-While-Revalidate:** The cache returns stale data to the user immediately, while triggering a background async request to update the cache.

---

### Q6: What is "Cache Penetration" and how do you solve it?

**Answer:**

**Problem:** An attacker repeatedly requests a key that does **not exist** in the cache AND does **not exist** in the database (e.g., `/user/99999999`). The cache misses, the DB is queried, returns nothing, and nothing is cached. The DB gets hammered.

**Solutions:**
1. **Cache Null/Empty Values:** If the DB returns nothing, cache the `null` value with a short TTL. Subsequent requests hit the cache and see `null` without hitting the DB.
2. **Bloom Filter:** Place a Bloom Filter in front of the cache. It quickly checks if a key *might* exist in the DB. If it definitely doesn't, reject the request immediately.

---

### Q7: Explain Redis Data Persistence (RDB vs AOF).

**Answer:**

Since Redis is in-memory, if the server restarts, data is lost. Redis provides two persistence mechanisms:

| Feature | RDB (Redis Database) | AOF (Append Only File) |
|---------|----------------------|------------------------|
| **Mechanism** | Takes point-in-time snapshots (e.g., every 5 mins). | Logs every single write operation as it happens. |
| **Data Loss Risk**| Moderate (lose data since last snapshot). | Very low (can configure to sync every second or every write). |
| **Restart Speed** | Fast (loading a binary dump). | Slower (replaying a large log of commands). |
| **File Size** | Compact binary file. | Large text file (requires background rewriting/compaction). |

*Best Practice:* Enable both. Use AOF for data safety and RDB for faster restarts and backups.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q8: How does Redis achieve high performance despite being single-threaded?

**Answer:**

Redis operates primarily on a **single thread** for command execution. 

**Why it's fast:**
1. **RAM-based:** No disk I/O during operations.
2. **No Lock Contention:** Because it's single-threaded, there are no race conditions, context switching, or locking overhead associated with multithreading.
3. **I/O Multiplexing (epoll/kqueue):** Redis uses an event loop to handle tens of thousands of concurrent connections efficiently. The OS notifies Redis when a socket is ready to read or write.

*(Note: In modern Redis versions, network I/O and disk persistence happen on background threads, but command execution remains single-threaded to ensure atomicity).*

---

### Q9: Explain Redis Sorted Sets (ZSET). How are they implemented under the hood?

**Answer:**

Sorted Sets store unique strings associated with a floating-point score. Elements are ordered by their score. (Great for leaderboards, rate limiters, priority queues).

**Implementation:**
Under the hood, a ZSET is implemented using a dual data structure:
1. **Hash Table:** Maps the element string to its score (allowing O(1) lookups of a member's score).
2. **Skip List:** A multi-layered linked list that keeps the elements sorted by score (allowing O(log N) inserts, deletions, and range queries).

*Why not a balanced tree?* Skip lists provide similar O(log N) performance but are much easier to implement, require less memory overhead, and are easier to modify without complex rebalancing rotations.

---

### Q10: How would you implement a Distributed Rate Limiter using Redis?

**Answer:**

There are multiple approaches:

**1. Fixed Window (Simplest but flawed):**
Use `INCR key` and `EXPIRE`. 
*Flaw:* If the limit is 100/min, a user can make 100 requests at 12:00:59 and 100 requests at 12:01:01. They did 200 requests in 2 seconds.

**2. Sliding Window Log (Accurate but memory heavy):**
Use a Redis Sorted Set (`ZSET`).
- Key: `user_id`. Value: `timestamp`. Score: `timestamp`.
- When a request comes: 
  1. `ZREMRANGEBYSCORE` to remove timestamps older than 1 minute.
  2. `ZCARD` to count current requests. If < limit, accept and `ZADD`.
*Flaw:* High memory usage, O(N) operations.

**3. Sliding Window Counter / Token Bucket (Best Practice):**
Use a Lua Script to ensure atomicity.
Maintain tokens. Calculate how many tokens should have regenerated since the last request, add them, check if > 0, deduct 1. All logic executes in one atomic Lua script call.

---

### Q11: What is Redis Cluster? How does it handle data sharding?

**Answer:**

Redis Cluster provides horizontal scaling and high availability.

**Data Sharding:**
- Redis Cluster does NOT use consistent hashing. It uses **Hash Slots**.
- There are exactly **16,384 hash slots**.
- `slot = CRC16(key) mod 16384`
- Every master node in the cluster is responsible for a subset of these slots (e.g., Node A takes 0-5500, Node B takes 5501-11000).
- If you add/remove a node, you migrate slots (and their keys) from one node to another.

**Client Redirection:**
If a client queries Node A for a key that belongs to Node B, Node A returns a `MOVED` error pointing to Node B. The client must then query Node B. (Smart clients cache this mapping to avoid the extra hop).

---

### Q12: How do you handle cache invalidation in a microservices architecture?

**Answer:**

Cache invalidation is notoriously difficult ("There are only two hard things in Computer Science: cache invalidation and naming things").

**Strategies:**
1. **TTL (Time to Live):** Easiest. Accept that data might be stale for X minutes.
2. **Synchronous Invalidation:** Service A updates the DB and immediately deletes the cache key. (Risk: Network fails during delete, cache remains permanently stale).
3. **Event-Driven Invalidation (Best Practice):**
   - Service A updates the DB.
   - Using **CDC (Change Data Capture)** like Debezium, or by publishing an event to Kafka, an update event is generated.
   - A Cache Invalidator service listens to the event stream and deletes/updates the cache.
   - *Pros:* Highly resilient, eventual consistency guaranteed, decouples the application from cache management.

---

*End of Redis & Caching Theory — 12 advanced questions covering patterns, internals, distributed systems, and real-world problems.*
