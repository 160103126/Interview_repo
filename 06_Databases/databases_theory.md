# Database Theory & Architecture

> Core database concepts including ACID, CAP Theorem, Sharding, Replication, and SQL vs NoSQL. Essential for backend and system design interviews.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What are ACID properties?

**Answer:**

ACID is a set of properties that guarantee database transactions are processed reliably.

- **Atomicity ("All or Nothing"):** A transaction is treated as a single unit. If any part of the transaction fails, the entire transaction is rolled back. (e.g., deducting money from Account A and adding to Account B must BOTH succeed or BOTH fail).
- **Consistency:** A transaction must take the database from one valid state to another. All constraints, triggers, and cascades must be satisfied.
- **Isolation:** Concurrent transactions must not interfere with each other. The result must be as if the transactions were executed sequentially.
- **Durability:** Once a transaction is committed, it remains committed even in the event of a system failure (e.g., power loss). Data is written to non-volatile storage (disk/SSD).

---

### Q2: What is the CAP Theorem?

**Answer:**

The CAP Theorem states that a distributed data store can only simultaneously provide **two of the three** following guarantees:

1. **Consistency (C):** Every read receives the most recent write or an error. (All nodes see the same data at the same time).
2. **Availability (A):** Every request receives a non-error response, without the guarantee that it contains the most recent write. (The system is always up).
3. **Partition Tolerance (P):** The system continues to operate despite an arbitrary number of messages being dropped or delayed by the network between nodes.

**In the real world, network partitions (P) are inevitable.** Therefore, systems must choose between Consistency (CP) and Availability (AP):

- **CP Systems (Consistency over Availability):** If a node is down, the system will reject requests rather than return stale data. *Examples: MongoDB, HBase, Redis, ZooKeeper.*
- **AP Systems (Availability over Consistency):** If a node is down, the system will return the most recent data it has (which might be stale). It provides "Eventual Consistency". *Examples: Cassandra, DynamoDB, CouchDB.*
- **CA Systems:** Only possible in single-node systems (like a standalone PostgreSQL or MySQL server). If the network fails, the system fails.

---

### Q3: What is the difference between SQL (Relational) and NoSQL (Non-Relational) databases?

**Answer:**

| Feature | SQL (RDBMS) | NoSQL |
|---------|-------------|-------|
| **Schema** | Rigid, predefined | Flexible, dynamic |
| **Data Model**| Tables with rows & columns | Document, Key-Value, Graph, Wide-Column |
| **Scaling** | Vertical (scale-up: bigger CPU/RAM) | Horizontal (scale-out: more cheap servers) |
| **ACID** | Built-in, strong guarantees | Varies (often Base/Eventual Consistency) |
| **Joins** | Excellent, native support | Poor or unsupported (must denormalize) |
| **Examples**| PostgreSQL, MySQL, Oracle | MongoDB, Cassandra, Redis, Neo4j |

**When to use SQL:** Financial systems, strong consistency requirements, complex relationships/queries, structured data.
**When to use NoSQL:** Rapid development, massive scale (horizontal scaling), unstructured/semi-structured data, high-throughput writes.

---

### Q4: Explain the BASE properties in NoSQL.

**Answer:**

BASE is the NoSQL alternative to ACID, prioritizing availability over strict consistency:

- **B**asically **A**vailable: The system guarantees availability, responding to requests even if parts are failing (might return stale data).
- **S**oft State: The state of the system may change over time, even without input, due to eventual consistency operations.
- **E**ventually Consistent: The system will eventually become consistent once it stops receiving input. All nodes will converge to the same value.

---

## 🟡 Medium (Intermediate)

### Q5: What is Database Normalization? Explain the Normal Forms (1NF, 2NF, 3NF).

**Answer:**

Normalization organizes data to reduce redundancy and improve data integrity.

- **1NF (First Normal Form):** 
  - Every column holds atomic (indivisible) values. (e.g., no comma-separated lists in a single column).
  - Each row must be unique (have a primary key).
- **2NF (Second Normal Form):**
  - Must be in 1NF.
  - No partial dependency: Non-key attributes must depend on the *entire* primary key, not just a part of it (only relevant for composite primary keys).
- **3NF (Third Normal Form):**
  - Must be in 2NF.
  - No transitive dependency: Non-key attributes must depend *only* on the primary key, not on other non-key attributes. (e.g., `City` depends on `ZipCode`, which depends on `CustomerID`. Move `City` and `ZipCode` to a separate table).

**Denormalization:** The intentional process of breaking normal forms (combining tables) to improve read performance (by avoiding expensive joins). Often used in data warehouses (OLAP) or high-read NoSQL systems.

---

### Q6: What is the Write-Ahead Log (WAL)? How does it guarantee durability?

**Answer:**

WAL (or Redo Log) is a standard method for ensuring data integrity and atomicity.

**How it works:**
1. A transaction modifies data in memory (RAM).
2. Before the transaction is considered "committed," the changes are appended to the WAL file on disk.
3. The database returns a success message to the client.
4. *Later* (asynchronously), the database flushes the changed memory pages to the actual database files on disk (checkpointing).

**Why use WAL?**
- **Durability:** If the server crashes before step 4, upon restart, the database replays the WAL to recover the lost memory changes.
- **Performance:** Appending to a sequential log file is much faster than doing random disk writes to update the scattered data pages directly.

---

### Q7: Explain Database Replication strategies.

**Answer:**

Replication copies data across multiple servers to improve availability and read scaling.

**1. Primary-Replica (Master-Slave):**
- One Primary handles all **Writes**.
- Multiple Replicas sync from the Primary and handle **Reads**.
- **Pros:** Great for read-heavy workloads.
- **Cons:** Primary is a single point of failure for writes. Replication lag can cause stale reads.

**2. Multi-Primary (Master-Master):**
- Multiple nodes accept writes and sync with each other.
- **Pros:** Higher write availability.
- **Cons:** Very complex conflict resolution (what if two nodes update the same row simultaneously?).

**3. Leaderless (Peer-to-Peer):**
- Any node can accept reads and writes (e.g., Cassandra).
- Uses Quorum (W + R > N) to ensure consistency.

**Synchronous vs Asynchronous Replication:**
- **Sync:** Primary waits for replicas to confirm the write before returning success. (Slower, highly consistent).
- **Async:** Primary returns success immediately, replicas sync in the background. (Faster, risk of data loss if Primary dies before sync).

---

### Q8: What is Sharding? How is it different from Partitioning?

**Answer:**

Both divide large tables into smaller pieces.

- **Partitioning:** Splitting a large table into smaller tables **on the same physical server**. (e.g., Partitioning an `Orders` table by month). Improves query performance and maintenance (can drop old months easily).
- **Sharding (Horizontal Partitioning):** Splitting a table across **multiple distinct physical servers** (shards). Each shard holds a unique subset of the data.

**Sharding Strategies:**
1. **Range Based:** e.g., Shard A gets User IDs 1-1000, Shard B gets 1001-2000. (Risk of uneven data distribution/hotspots).
2. **Hash Based:** e.g., `hash(user_id) % num_shards`. (Even distribution, but hard to add new shards because the modulo changes).
3. **Directory Based:** A lookup service keeps a mapping of which shard holds which key.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q9: What is MVCC (Multi-Version Concurrency Control)?

**Answer:**

MVCC is a concurrency control method (used by PostgreSQL, MySQL/InnoDB) that provides high performance without locking for reads.

**Core Concept: "Readers don't block writers, and writers don't block readers."**

- Instead of overwriting a row, an `UPDATE` creates a **new version** of the row.
- Each row version has a timestamp/transaction ID (e.g., `xmin` and `xmax` in Postgres) indicating when it was created and deleted.
- When a transaction reads data, it sees a "snapshot" of the database as it existed when the transaction started. It ignores row versions created by transactions that committed *after* its snapshot was taken.
- **VACUUM:** Because old row versions (dead tuples) accumulate, a background process (vacuuming/garbage collection) must periodically clean them up to free space.

---

### Q10: Explain the problem of "Split-Brain" in distributed databases. How is it prevented?

**Answer:**

**Split-Brain** occurs in a distributed system (like a Primary-Replica setup) when a network partition causes nodes to lose communication with each other. Both halves of the network think the other half is dead, so both promote a node to be the "Primary". 
Now you have two Primaries accepting conflicting writes, leading to catastrophic data corruption.

**Prevention:**
1. **Quorum / Consensus Algorithms (Raft, Paxos):** An action (like electing a leader) requires a strict majority vote (N/2 + 1). In a network partition, only the partition with the majority of nodes can elect a leader. The minority partition pauses. (This is why clusters should have an odd number of nodes, e.g., 3, 5, 7).
2. **Fencing (STONITH - Shoot The Other Node In The Head):** A hardware-level solution where the active node explicitly powers off the disconnected node to ensure it cannot accept writes.

---

### Q11: What is a B-Tree vs LSM Tree (Log-Structured Merge Tree)? When are they used?

**Answer:**

These are the two dominant storage engine architectures.

**B-Tree (B+ Tree):**
- **How it works:** Updates are done in-place. Finding data is O(log N).
- **Pros:** Fast random reads. Great for transaction-heavy (OLTP) relational databases.
- **Cons:** Slow random writes (requires disk seeks and node splitting).
- **Used in:** PostgreSQL, MySQL (InnoDB), Oracle.

**LSM Tree:**
- **How it works:** **Never modifies data in-place.** All writes are appended to an in-memory buffer (MemTable). When full, it's flushed to disk as an immutable Sorted String Table (SSTable). Background processes merge and compact SSTables over time.
- **Pros:** Extremely fast writes (sequential disk I/O only).
- **Cons:** Reads can be slower (must check MemTable and multiple SSTables, though Bloom filters help). Compaction uses CPU/IO.
- **Used in:** Cassandra, RocksDB, LevelDB, HBase, DynamoDB.

---

### Q12: Explain the "Two-Phase Commit" (2PC) protocol.

**Answer:**

2PC is an algorithm to achieve atomic transaction commit across multiple distributed databases. (Distributed Transaction).

**Phase 1: Prepare Phase**
1. The **Coordinator** sends a `PREPARE` message to all participant nodes.
2. Participants write to their WAL and lock resources, then reply with `VOTE COMMIT` or `VOTE ABORT`.

**Phase 2: Commit Phase**
1. If the Coordinator receives `VOTE COMMIT` from **ALL** participants, it sends a `COMMIT` message to all.
2. If ANY participant voted `ABORT`, or timed out, the Coordinator sends `ROLLBACK` to all.

**Flaws of 2PC:**
- It is a **blocking protocol**. If the Coordinator crashes during Phase 2, participants are left holding locks indefinitely, freezing the system.
- Modern alternative: **Saga Pattern** (event-driven, asynchronous compensating transactions).

---

*End of Database Theory — 12 advanced questions covering CAP theorem, replication, sharding, MVCC, and distributed consensus.*
