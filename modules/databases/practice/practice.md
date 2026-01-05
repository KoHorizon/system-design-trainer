# Module 3: Databases - Practice

Test your understanding of SQL vs NoSQL, ACID, indexing, replication, and sharding.

---

## Section 1: Database Selection

### Exercise 1.1: Match the Database

For each scenario, choose the best database type:

**Options**: 
- A) PostgreSQL (SQL)
- B) MongoDB (Document)
- C) Redis (Key-Value)
- D) Cassandra (Wide-Column)
- E) Neo4j (Graph)

1. ___ E-commerce platform with complex inventory and order transactions
2. ___ Real-time gaming leaderboard with millions of score updates per second
3. ___ IoT platform storing billions of sensor readings per day
4. ___ Social network's "friend of friend" recommendations
5. ___ Content management system with varying article structures
6. ___ Session storage for 10 million concurrent users
7. ___ Financial trading platform requiring strict consistency
8. ___ Event logging system with append-only writes

### Exercise 1.2: SQL vs NoSQL Trade-offs

For each requirement, mark whether SQL or NoSQL is better suited:

| Requirement | SQL | NoSQL |
|-------------|-----|-------|
| Complex JOIN queries across 5+ tables | | |
| Horizontal write scaling to 100K writes/sec | | |
| Strict ACID transactions | | |
| Flexible, evolving schema | | |
| Ad-hoc analytical queries | | |
| Global distribution with local writes | | |
| Strong data integrity with foreign keys | | |
| Sub-millisecond key-value lookups | | |

---

## Section 2: ACID Properties

### Exercise 2.1: Identify the Violation

For each scenario, identify which ACID property is violated:

**Options**: Atomicity, Consistency, Isolation, Durability

1. ___ System crashes after debiting Account A but before crediting Account B. $100 is lost.

2. ___ Two users simultaneously book the last concert ticket. Both succeed, but there's only one ticket.

3. ___ Database confirms a write, but data is lost after power failure.

4. ___ User sets account balance to -$500, violating the "balance >= 0" constraint.

### Exercise 2.2: Isolation Level Scenarios

What's the minimum isolation level needed to prevent each problem?

**Options**: Read Uncommitted, Read Committed, Repeatable Read, Serializable

1. ___ Prevent reading data from uncommitted transactions

2. ___ Ensure the same query returns the same rows throughout a transaction

3. ___ Prevent another transaction from inserting rows that match your WHERE clause

4. ___ Ensure two concurrent transfers don't create/destroy money

### Exercise 2.3: Transaction Analysis

```sql
-- Transaction A (Bank Transfer)
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- Returns $1000
-- 500ms pause
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT;

-- Transaction B (runs during A's pause)
BEGIN;
UPDATE accounts SET balance = balance - 300 WHERE id = 1;
COMMIT;
```

**Questions**:

1. With READ COMMITTED isolation, what's Account 1's final balance?
   ```
   Starting: $1000
   After Transaction B: $___
   After Transaction A: $___
   ```

2. Is money conserved? Why or why not?

3. What isolation level prevents this problem?

---

## Section 3: Indexing

### Exercise 3.1: Index Usage Prediction

Given this table and index:
```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    status VARCHAR(20),
    total DECIMAL(10,2),
    created_at TIMESTAMP
);

CREATE INDEX idx_orders_user_status_created 
ON orders(user_id, status, created_at);
```

Will the index be used for each query? Answer **Yes**, **Partial**, or **No**:

1. ___ `SELECT * FROM orders WHERE user_id = 123`

2. ___ `SELECT * FROM orders WHERE status = 'pending'`

3. ___ `SELECT * FROM orders WHERE user_id = 123 AND status = 'pending'`

4. ___ `SELECT * FROM orders WHERE user_id = 123 AND created_at > '2024-01-01'`

5. ___ `SELECT * FROM orders WHERE user_id = 123 ORDER BY created_at`

6. ___ `SELECT * FROM orders WHERE status = 'pending' AND created_at > '2024-01-01'`

7. ___ `SELECT * FROM orders WHERE YEAR(created_at) = 2024`

8. ___ `SELECT user_id, status, created_at FROM orders WHERE user_id = 123`

### Exercise 3.2: Design the Index

For each query pattern, design the optimal composite index:

1. **Query**: `SELECT * FROM products WHERE category_id = ? AND price BETWEEN ? AND ? ORDER BY rating DESC`
   
   ```sql
   CREATE INDEX idx_products_??? ON products(???);
   ```

2. **Query**: `SELECT * FROM users WHERE country = ? AND status = 'active' AND last_login > ?`
   
   ```sql
   CREATE INDEX idx_users_??? ON users(???);
   ```

3. **Query**: `SELECT sender_id, COUNT(*) FROM messages WHERE receiver_id = ? AND read = false GROUP BY sender_id`
   
   ```sql
   CREATE INDEX idx_messages_??? ON messages(???);
   ```

### Exercise 3.3: Index Trade-off Analysis

**Scenario**: Table with 50 million rows, 1000 inserts/second, 5000 reads/second.

Current indexes:
- Primary key (id)
- idx_user_id (user_id)
- idx_status (status)
- idx_created (created_at)
- idx_user_status (user_id, status)
- idx_user_created (user_id, created_at)
- idx_status_created (status, created_at)

**Questions**:

1. How many index updates occur per insert?
   ```
   ___ index updates per insert
   ```

2. Which indexes might be redundant?
   ```
   ___
   ```

3. If write performance is critical, which indexes would you consider dropping?
   ```
   ___
   ```

---

## Section 4: Replication

### Exercise 4.1: Replication Topology Selection

Match each scenario to the best replication topology:

**Options**:
- A) Single-leader (primary-replica)
- B) Multi-leader
- C) Leaderless (quorum)

1. ___ Read-heavy web application with 95% reads, 5% writes

2. ___ Global application where users need low-latency writes from any region

3. ___ System requiring high availability where any node failure should not block writes

4. ___ Application requiring strict consistency for financial transactions

5. ___ Collaborative document editing with offline support

### Exercise 4.2: Replication Lag Problem

**Scenario**: E-commerce site with async replication (100ms average lag)

```
Timeline:
T=0ms:    User adds item to cart (writes to Primary)
T=10ms:   Primary confirms success
T=20ms:   User clicks "View Cart" (reads from Replica)
T=100ms:  Replication completes
```

**Questions**:

1. What does the user see at T=20ms?
   ```
   ___
   ```

2. Name two strategies to fix this:
   ```
   Strategy 1: ___
   Strategy 2: ___
   ```

3. What's the trade-off of reading from primary for this user?
   ```
   ___
   ```

### Exercise 4.3: Quorum Calculation

**Setup**: 5-node cluster with quorum replication

**Questions**:

1. For W=3, R=3, is strong consistency guaranteed?
   ```
   W + R = ___ + ___ = ___
   N = ___
   W + R > N? ___
   Consistent? ___
   ```

2. Node 1 and Node 2 fail. Can the system still:
   - Accept writes (W=3)? ___
   - Accept reads (R=3)? ___

3. For high availability writes with eventual consistency reads, what W and R would you choose?
   ```
   W = ___
   R = ___
   ```

---

## Section 5: Sharding

### Exercise 5.1: Shard Key Selection

**Scenario**: Social media platform with these access patterns:
- Get user profile by user_id
- Get user's posts (most recent first)
- Get user's feed (posts from followed users)
- Search posts by hashtag
- Get trending posts globally

**Proposed shard keys**:
- A) user_id
- B) post_id
- C) hashtag
- D) created_at

**Questions**:

1. Which shard key works best for "Get user profile"?
   ```
   ___
   ```

2. Which shard key works best for "Get user's posts"?
   ```
   ___
   ```

3. Which query will be slowest regardless of shard key chosen?
   ```
   ___
   Why: ___
   ```

4. If you shard by user_id, how would you handle "Get trending posts globally"?
   ```
   ___
   ```

### Exercise 5.2: Hash vs Range Sharding

**Scenario**: Time-series data for IoT devices
- 1 million devices
- Each device sends data every minute
- Common queries:
  - Get latest reading for device X
  - Get readings for device X in time range
  - Get all readings in last hour (across all devices)

**Questions**:

1. If you range-shard by timestamp:
   - Where do new writes go? ___
   - Is this a problem? Why? ___

2. If you hash-shard by device_id:
   - Are writes evenly distributed? ___
   - Can you efficiently query one device's time range? ___
   - Can you efficiently query all devices in last hour? ___

3. Propose a better sharding strategy:
   ```
   ___
   ```

### Exercise 5.3: Cross-Shard Transaction

**Scenario**: 
- Users sharded by user_id
- User A (shard 1) sends $100 to User B (shard 2)

**Questions**:

1. Why can't you use a normal database transaction?
   ```
   ___
   ```

2. Describe the steps for a two-phase commit:
   ```
   Phase 1 (Prepare):
   ___
   
   Phase 2 (Commit):
   ___
   ```

3. What happens if Shard 2 crashes after Phase 1 but before Phase 2?
   ```
   ___
   ```

4. How would you implement this with the Saga pattern instead?
   ```
   Step 1: ___
   Step 2: ___
   Compensation (if Step 2 fails): ___
   ```

---

## Section 6: Design Exercises

### Exercise 6.1: Design a User Database

**Requirements**:
- 100 million users
- Profile data: ~2KB per user
- Read-heavy: 50,000 reads/sec, 500 writes/sec
- Strong consistency for account balance
- Eventual consistency OK for profile updates
- 99.99% availability

**Design**:

1. Database choice and why:
   ```
   ___
   ```

2. Replication strategy:
   ```
   ___
   ```

3. Do you need sharding? Calculate:
   ```
   Storage: 100M × 2KB = ___
   Read capacity needed: ___
   Single PostgreSQL can handle: ~10,000 reads/sec
   Decision: ___
   ```

4. High availability setup:
   ```
   ___
   ```

### Exercise 6.2: Design a Messaging Database

**Requirements**:
- 500 million users
- 50 billion messages/day
- Access patterns:
  - Get conversation between User A and User B
  - Get User A's recent messages (inbox)
  - Search messages by keyword
- Messages stored for 1 year

**Design**:

1. Calculate storage:
   ```
   Messages/day: 50 billion
   Average message size: 500 bytes
   Daily storage: ___
   Yearly storage: ___
   ```

2. Calculate write throughput:
   ```
   Messages/day: 50 billion
   Messages/second: ___
   ```

3. Database choice:
   ```
   Primary storage: ___
   Search: ___
   Why: ___
   ```

4. Sharding strategy:
   ```
   Shard key: ___
   Why: ___
   How to handle "inbox" query: ___
   ```

### Exercise 6.3: Debug the Slow Query

**Scenario**: Query takes 30 seconds

```sql
SELECT u.name, COUNT(o.id) as order_count, SUM(o.total) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.created_at > '2024-01-01'
  AND o.status IN ('completed', 'shipped')
GROUP BY u.id
ORDER BY total_spent DESC
LIMIT 100;

-- Table sizes:
-- users: 10 million rows
-- orders: 500 million rows

-- Existing indexes:
-- users: PRIMARY KEY (id)
-- orders: PRIMARY KEY (id), INDEX (user_id)
```

**Questions**:

1. What's likely causing the slowness? (List potential issues)
   ```
   1. ___
   2. ___
   3. ___
   ```

2. What indexes would you add?
   ```sql
   CREATE INDEX ___ ON orders(___);
   ```

3. Could you rewrite the query to be faster?
   ```sql
   ___
   ```

4. What if you still can't get acceptable performance?
   ```
   ___
   ```

---

## Section 7: Quick Recall Quiz

### Part A: SQL vs NoSQL (5 points)

1. Document databases are best for ___ data patterns
2. Wide-column stores excel at ___ throughput
3. SQL databases provide ___ transactions
4. Graph databases optimize for ___ traversal
5. Key-value stores offer O(___) lookups

### Part B: ACID (4 points)

1. ___ ensures transactions are all-or-nothing
2. ___ prevents concurrent transactions from interfering
3. ___ ensures committed data survives crashes
4. ___ ensures data moves from valid state to valid state

### Part C: Indexing (4 points)

1. B-tree indexes support ___ queries, hash indexes do not
2. In composite index (A, B, C), you cannot skip ___ to use C
3. Every index adds ___% to write overhead
4. Covering indexes avoid accessing the ___

### Part D: Replication (4 points)

1. Synchronous replication trades ___ for consistency
2. Replication lag in same datacenter is typically ___
3. For quorum consistency: W + R must be > ___
4. Multi-leader replication's biggest challenge is ___

### Part E: Sharding (3 points)

1. Hash sharding prevents ___ but breaks ___ queries
2. Consistent hashing minimizes ___ when adding nodes
3. Cross-shard transactions require ___ or saga pattern

---

## Answer Key

### Section 1: Database Selection

**1.1:**
1. A (complex transactions)
2. C (fast writes, simple key-value)
3. D (write-heavy, time-series pattern)
4. E (relationship traversal)
5. B (flexible schema)
6. C (fast, ephemeral data)
7. A (strict ACID)
8. D (append-only, high write throughput)

**1.2:**
| Requirement | SQL | NoSQL |
|-------------|-----|-------|
| Complex JOIN queries | ✓ | |
| Horizontal write scaling | | ✓ |
| Strict ACID transactions | ✓ | |
| Flexible schema | | ✓ |
| Ad-hoc analytical queries | ✓ | |
| Global distribution | | ✓ |
| Strong data integrity | ✓ | |
| Sub-millisecond key-value | | ✓ |

### Section 2: ACID

**2.1:**
1. Atomicity
2. Isolation
3. Durability
4. Consistency

**2.2:**
1. Read Committed
2. Repeatable Read
3. Serializable
4. Serializable

**2.3:**
1. After B: $700, After A: $200 (A used stale $1000 balance)
2. No! $300 disappeared. A deducted $500 from $1000 (ignoring B's $300 deduction)
3. Serializable (or at minimum Repeatable Read with proper locking)

### Section 3: Indexing

**3.1:**
1. Yes (leftmost column)
2. No (can't skip user_id)
3. Yes (leftmost columns)
4. Partial (user_id yes, but created_at skips status)
5. Partial (user_id yes, but ORDER BY needs status first)
6. No (can't skip user_id)
7. No (function on column)
8. Yes + Covering index (all columns in index)

**3.2:**
1. `CREATE INDEX idx_products_cat_price_rating ON products(category_id, price, rating DESC);`
2. `CREATE INDEX idx_users_country_status_login ON users(country, status, last_login);`
3. `CREATE INDEX idx_messages_recv_read_sender ON messages(receiver_id, read, sender_id);`

**3.3:**
1. 7 index updates per insert (1 PK + 6 secondary indexes)
2. idx_user_id is redundant (covered by idx_user_status and idx_user_created)
3. Consider dropping idx_status (low selectivity), idx_created (if not queried alone)

### Section 4: Replication

**4.1:**
1. A (simple, read scaling with replicas)
2. B (local writes in each region)
3. C (no single leader = no write SPOF)
4. A (strong consistency simpler with single leader)
5. B (offline writes sync later)

**4.2:**
1. Empty cart (replica hasn't received the write yet)
2. Read-your-writes (route to primary after write), Monotonic reads (sticky to primary for session)
3. Increased load on primary, potential bottleneck

**4.3:**
1. W + R = 3 + 3 = 6, N = 5, 6 > 5 = Yes, Consistent = Yes
2. Writes: No (only 3 nodes, need 3 for W=3), Reads: Yes (3 available, need 3 for R=3)
3. W = 1 (fast writes), R = 1 (fast reads) — Note: Not consistent!

### Section 5: Sharding

**5.1:**
1. A (user_id) — direct lookup
2. A (user_id) — all user's posts on same shard
3. "Get trending posts globally" — must query all shards regardless
4. Scatter-gather across all shards, or maintain separate aggregated cache/table

**5.2:**
1. All to latest time shard; Yes, creates hot spot
2. Yes; Yes (same device same shard); No (must query all shards)
3. Composite: hash(device_id) for shard, range(timestamp) for partitioning within shard

**5.3:**
1. Transactions don't span database instances/connections
2. Phase 1: Ask both shards to prepare (lock resources, validate). Phase 2: If both prepared, commit both; otherwise rollback both
3. Coordinator waits/retries until Shard 2 recovers and commits (or times out and rolls back)
4. Step 1: Deduct from A (local tx). Step 2: Credit to B (local tx). Compensation: If Step 2 fails, credit back to A

### Section 6: Design Exercises

**6.1:**
1. PostgreSQL — ACID for balance, mature, handles the scale
2. Primary + 2-3 async replicas (sync for one for durability)
3. Storage: 200GB (easily fits). Reads: 50K/sec needs 5+ replicas. No sharding needed yet.
4. Primary + sync replica in same AZ, async replica in different AZ, automated failover

**6.2:**
1. Daily: 25 TB, Yearly: ~9 PB
2. 50B / 86400 ≈ 580,000 messages/second
3. Primary: Cassandra (write throughput), Search: Elasticsearch
4. Shard key: conversation_id or user_id; Inbox: denormalize or maintain separate inbox table

**6.3:**
1. No index on (created_at, status), scanning 500M orders, GROUP BY on 10M users
2. `CREATE INDEX idx_orders_date_status_user ON orders(created_at, status, user_id, total);`
3. Filter orders first with CTE or subquery, then join to users
4. Pre-aggregate with materialized view, use analytics DB (ClickHouse), or nightly batch job

### Section 7: Quick Recall

**Part A:** hierarchical/nested, write, ACID, relationship, 1
**Part B:** Atomicity, Isolation, Durability, Consistency
**Part C:** range, A and B, 10-30, table
**Part D:** latency/availability, 1-10ms, N, conflict resolution
**Part E:** hot spots, range; data redistribution; two-phase commit

---

## Scoring

**Section 7 (20 points)**:
- 18-20: Database expert level
- 14-17: Strong foundation
- 10-13: Review weak areas
- <10: Re-study the concepts

---

## Next Steps

Ready for:
**[Module 4: Caching](../caching/concepts/concepts.md)**