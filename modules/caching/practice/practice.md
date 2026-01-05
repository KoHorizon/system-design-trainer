# Module 4: Caching - Practice

Test your understanding of caching strategies, eviction policies, and invalidation patterns.

---

## Section 1: Caching Strategy Selection

### Exercise 1.1: Match the Strategy

For each scenario, choose the best caching strategy:

**Options**:
- A) Cache-Aside
- B) Read-Through
- C) Write-Through
- D) Write-Behind
- E) Write-Around

1. ___ E-commerce product catalog read 1000x more than updated
2. ___ User session that must be immediately consistent after login
3. ___ High-frequency sensor data that can tolerate some data loss
4. ___ Logging system where logs are written but rarely read
5. ___ Social media feed that can be slightly stale
6. ___ Banking transaction that requires read-after-write consistency

### Exercise 1.2: Strategy Trade-offs

Fill in the table:

| Strategy | Write Latency | Read Miss Latency | Consistency | Data Loss Risk |
|----------|---------------|-------------------|-------------|----------------|
| Cache-Aside | ___ | ___ | ___ | ___ |
| Write-Through | ___ | ___ | ___ | ___ |
| Write-Behind | ___ | ___ | ___ | ___ |

**Options**: High, Low, Strong, Eventual, Yes, No

---

## Section 2: Hit Rate Calculations

### Exercise 2.1: Average Latency

**Given:**
- Cache latency: 2ms
- Database latency: 100ms
- Cache hit rate: 85%

**Calculate average request latency:**
```
Average = (hit_rate × cache_latency) + (miss_rate × db_latency)
        = (___ × ___) + (___ × ___)
        = ___ + ___
        = ___ ms
```

### Exercise 2.2: Required Hit Rate

**Goal:** Reduce average latency from 100ms to under 15ms

**Given:**
- Cache latency: 1ms
- Database latency: 100ms

**What hit rate is needed?**
```
Target: 15ms = (hit_rate × 1) + ((1 - hit_rate) × 100)
15 = hit_rate + 100 - 100×hit_rate
15 = 100 - 99×hit_rate
99×hit_rate = 85
hit_rate = ____%
```

### Exercise 2.3: Cache Size vs Hit Rate

You observe these hit rates at different cache sizes:

| Cache Size | Hit Rate |
|------------|----------|
| 1 GB | 70% |
| 2 GB | 82% |
| 4 GB | 90% |
| 8 GB | 95% |
| 16 GB | 97% |

**Questions:**

1. Current cache: 4 GB, average latency: 11.8ms (cache=1ms, DB=100ms). What's the latency if you double cache to 8 GB?
   ```
   Current: (0.90 × 1) + (0.10 × 100) = 10.9ms
   New: (0.___ × 1) + (0.___ × 100) = ___ms
   Improvement: ___ms saved
   ```

2. Is doubling from 8 GB to 16 GB worth it for latency?
   ```
   8 GB: ___ms
   16 GB: ___ms
   Improvement: ___ms
   Worth it? ___
   ```

---

## Section 3: Eviction Policies

### Exercise 3.1: LRU Simulation

**Cache size: 4 items**
**Access sequence: A, B, C, D, E, A, F, B, G**

Trace the cache state and evictions:

| Access | Cache State (oldest → newest) | Evicted |
|--------|-------------------------------|---------|
| A | [A] | - |
| B | [A, B] | - |
| C | [A, B, C] | - |
| D | [A, B, C, D] | - |
| E | ___ | ___ |
| A | ___ | ___ |
| F | ___ | ___ |
| B | ___ | ___ |
| G | ___ | ___ |

**Final hit/miss count:**
- Hits: ___
- Misses: ___

### Exercise 3.2: LFU Simulation

**Cache size: 3 items**
**Access sequence: A, A, B, A, C, B, D, A**

Each item tracks access count. On tie, evict oldest.

| Access | Cache State (item:count) | Evicted |
|--------|--------------------------|---------|
| A | [A:1] | - |
| A | [A:2] | - |
| B | [A:2, B:1] | - |
| A | [A:3, B:1] | - |
| C | [A:3, B:1, C:1] | - |
| B | [A:3, B:2, C:1] | - |
| D | ___ | ___ |
| A | ___ | ___ |

### Exercise 3.3: Policy Selection

Which eviction policy for each scenario?

**Options**: LRU, LFU, FIFO, TTL, Random

1. ___ User sessions that must expire after 30 minutes of inactivity
2. ___ Video streaming cache where popular videos should stay cached
3. ___ Simple embedded system with minimal memory overhead
4. ___ General-purpose web application cache
5. ___ News articles where recency matters more than popularity

---

## Section 4: Cache Invalidation

### Exercise 4.1: Identify the Problem

```go
// Service A
func updateUserName(userID string, newName string) error {
    if err := db.Exec("UPDATE users SET name = ? WHERE id = ?", newName, userID); err != nil {
        return err
    }
    cache.Set(ctx, "user:"+userID, getUserFromDB(userID), time.Hour)
    return nil
}

// Service B (different server)
func getUser(userID string) (*User, error) {
    cached, err := cache.Get(ctx, "user:"+userID)
    if err == nil && cached != "" {
        return deserializeUser(cached), nil
    }
    user, _ := db.GetUser(userID)
    cache.Set(ctx, "user:"+userID, serializeUser(user), time.Hour)
    return user, nil
}
```

**Questions:**

1. What's wrong with the update function?
   ```
   ___
   ```

2. What happens if two users update simultaneously?
   ```
   T1: User A reads from DB (name="Alice")
   T2: User B updates DB to name="Alicia"
   T3: User B updates cache with "Alicia"
   T4: User A updates cache with "Alice" (stale!)
   
   Result: ___
   ```

3. How would you fix this?
   ```go
   func updateUserName(userID string, newName string) error {
       ___
   }
   ```

### Exercise 4.2: Design Invalidation

**Scenario:** User profile cached in multiple places:
- `user:{id}` - Basic profile
- `user:{id}:settings` - User settings
- `user:{id}:feed` - Computed feed (depends on settings)
- `profile_page:{id}` - Rendered profile page
- Search index entry

**When user updates their profile, what needs invalidation?**

```
Direct invalidation:
- ___
- ___

Dependent invalidation (things that derive from profile):
- ___
- ___

External systems:
- ___
```

**How would you implement this reliably?**
```
___
```

### Exercise 4.3: TTL Selection

Choose appropriate TTLs:

| Data Type | Suggested TTL | Reasoning |
|-----------|---------------|-----------|
| User authentication token | ___ | ___ |
| Product price | ___ | ___ |
| User's shopping cart | ___ | ___ |
| Daily leaderboard | ___ | ___ |
| Static site configuration | ___ | ___ |
| Real-time stock price | ___ | ___ |

---

## Section 5: Problem Scenarios

### Exercise 5.1: Cache Stampede

**Scenario:** Your cache key `popular_products` expires every hour. At expiration, you see:
- Database CPU spikes to 100%
- 500 concurrent requests all miss cache
- Response time goes from 5ms to 5000ms for 30 seconds

**Questions:**

1. What's happening?
   ```
   ___
   ```

2. Implement a fix using locking:
   ```go
   func getPopularProducts() ([]Product, error) {
       cached, _ := cache.Get(ctx, "popular_products")
       if cached != "" {
           return deserialize(cached), nil
       }
       
       // Your locking solution:
       ___
   }
   ```

3. Implement a fix using early refresh:
   ```go
   func getPopularProducts() ([]Product, error) {
       // Your early refresh solution:
       ___
   }
   ```

### Exercise 5.2: Hot Key

**Scenario:** Celebrity with 50M followers posts. Their profile key receives 100,000 requests/second. Single Redis node handles 100,000 QPS max.

**Questions:**

1. What's the problem?
   ```
   ___
   ```

2. Solution 1 - Local caching:
   ```go
   // How would you implement this?
   ___
   ```

3. Solution 2 - Key replication:
   ```go
   // How would you implement this?
   ___
   ```

4. Which solution is better and why?
   ```
   ___
   ```

### Exercise 5.3: Cache Penetration

**Scenario:** Attackers send requests for user IDs that don't exist: `user:999999999`, `user:999999998`, etc.

**Observed behavior:**
- 0% cache hit rate for these requests
- Each request hits database
- Database under heavy load

**Questions:**

1. Why doesn't caching help?
   ```
   ___
   ```

2. Solution 1 - Cache null results:
   ```go
   func getUser(userID string) (*User, error) {
       ___
   }
   ```

3. Solution 2 - Bloom filter:
   ```go
   // Assume bloomFilter is pre-populated with existing user IDs
   func getUser(userID string) (*User, error) {
       ___
   }
   ```

4. What are the trade-offs of each solution?
   ```
   Null caching:
   + ___
   - ___
   
   Bloom filter:
   + ___
   - ___
   ```

---

## Section 6: Design Exercises

### Exercise 6.1: E-commerce Caching Strategy

**Requirements:**
- 1M products, 100M users
- Product pages: 50,000 views/second
- Product updates: 100/second
- Prices change frequently (promotions)
- Inventory must be accurate (no overselling)

**Design:**

1. What to cache?
   ```
   Cache: ___
   Don't cache: ___
   ```

2. Caching strategy for product details?
   ```
   Strategy: ___
   TTL: ___
   Invalidation: ___
   ```

3. How to handle inventory?
   ```
   ___
   ```

4. Cache architecture diagram:
   ```
   [Draw or describe the layers]
   ```

### Exercise 6.2: Session Caching

**Requirements:**
- 10M concurrent users
- Session data: ~2KB per user
- Session must survive server restarts
- 99.9% availability required
- Read-after-write consistency for same user

**Design:**

1. Storage calculation:
   ```
   Total size: 10M × 2KB = ___
   With overhead (1.5x): ___
   ```

2. Redis or Memcached? Why?
   ```
   Choice: ___
   Reason: ___
   ```

3. Architecture for high availability:
   ```
   ___
   ```

4. How to handle read-after-write?
   ```
   ___
   ```

### Exercise 6.3: Cache Warming

**Scenario:** You're deploying new cache servers. Cold cache causes:
- 0% hit rate initially
- Database overwhelmed with traffic
- Slow response times for 30+ minutes

**Design a cache warming strategy:**

1. What data to pre-warm?
   ```
   ___
   ```

2. How to identify hot keys?
   ```
   ___
   ```

3. Warming implementation:
   ```go
   func warmCache() {
       ___
   }
   ```

4. How to avoid overwhelming the database during warming?
   ```
   ___
   ```

---

## Section 7: Quick Recall Quiz

### Part A: Strategies (5 points)

1. ___ strategy: App checks cache, on miss loads from DB and populates cache
2. ___ strategy: Writes go to cache first, async to DB
3. ___ strategy: Writes sync to both cache and DB
4. ___ strategy: Writes go directly to DB, bypassing cache
5. Cache-aside is also called ___ loading

### Part B: Eviction (4 points)

1. LRU evicts the ___ accessed item
2. LFU evicts the ___ accessed item
3. TTL-based eviction guarantees data ___ (freshness/size)
4. ___ eviction has lowest memory overhead

### Part C: Problems (4 points)

1. ___ herd: Many requests hit DB when popular key expires
2. ___ key: Single key receives disproportionate traffic
3. Cache ___: Requests for non-existent data bypass cache
4. Cache ___: Many keys expire simultaneously

### Part D: Numbers (4 points)

1. Good cache hit rate target: > ____%
2. Redis single node QPS: ~___
3. Typical cache latency: ___-___ ms
4. Going from 90% to 99% hit rate improves latency by ~___x

### Part E: Redis vs Memcached (3 points)

1. ___ supports data structures (lists, sets, sorted sets)
2. ___ has better memory efficiency
3. ___ supports persistence

---

## Answer Key

### Section 1

**1.1:**
1. A (read-heavy, cache-aside is natural fit)
2. C (write-through for immediate consistency)
3. D (write-behind for speed, acceptable loss)
4. E (write-around for write-once-read-never)
5. A (eventual consistency OK)
6. C (write-through for consistency)

**1.2:**

| Strategy | Write Latency | Read Miss Latency | Consistency | Data Loss Risk |
|----------|---------------|-------------------|-------------|----------------|
| Cache-Aside | Low (direct DB) | High | Eventual | No |
| Write-Through | High | Low | Strong | No |
| Write-Behind | Low | Low | Eventual | Yes |

### Section 2

**2.1:**
```
Average = (0.85 × 2) + (0.15 × 100)
        = 1.7 + 15
        = 16.7ms
```

**2.2:**
```
hit_rate = 85/99 = 85.86% (need ~86% hit rate)
```

**2.3:**
1. New: (0.95 × 1) + (0.05 × 100) = 5.95ms, saved 4.95ms
2. 8GB: 5.95ms, 16GB: 3.97ms, improvement: 1.98ms. Probably not worth 2x memory cost.

### Section 3

**3.1:**

| Access | Cache State | Evicted |
|--------|-------------|---------|
| E | [B, C, D, E] | A |
| A | [C, D, E, A] | B |
| F | [D, E, A, F] | C |
| B | [E, A, F, B] | D |
| G | [A, F, B, G] | E |

Hits: 2 (A, B), Misses: 7

**3.2:**

| Access | Cache State | Evicted |
|--------|-------------|---------|
| D | [A:3, B:2, D:1] | C (lowest count) |
| A | [A:4, B:2, D:1] | - |

**3.3:**
1. TTL
2. LFU
3. FIFO or Random
4. LRU
5. LRU

### Section 4

**4.1:**
1. Race condition - reads stale data, writes to cache
2. Cache contains stale "Alice" instead of "Alicia"
3. Delete instead of set: `cache.delete(f"user:{user_id}")`

**4.2:**
- Direct: `user:{id}`, `user:{id}:settings`
- Dependent: `user:{id}:feed`, `profile_page:{id}`
- External: Search index
- Implementation: Event-based invalidation with message queue

**4.3:**

| Data Type | TTL | Reasoning |
|-----------|-----|-----------|
| Auth token | 15-60 min | Security |
| Product price | 1-5 min | Promotions change |
| Shopping cart | 24-72 hours | User might return |
| Daily leaderboard | Until midnight | Resets daily |
| Static config | 1-24 hours | Rarely changes |
| Stock price | 1-5 seconds | Real-time needed |

### Section 5

**5.1:**
1. Cache stampede - 500 requests all try to rebuild cache simultaneously
2. Use `setnx` for lock, only one request fetches from DB
3. Refresh probabilistically before expiry, or use background refresh

**5.2:**
1. Single Redis node can't handle 100K QPS for one key
2. Local in-memory cache on each app server with short TTL
3. Replicate key: `celebrity:123:r{0-9}`, randomly select on read
4. Local caching better - no network hop, handles higher load

**5.3:**
1. Cache misses don't store anything, so every request hits DB
2. Store "NULL" with short TTL for missing keys
3. Check bloom filter first, return null if definitely not exists
4. Null caching: + Simple, - Uses memory for nulls. Bloom: + Memory efficient, - False positives possible

### Section 6

**6.1:**
1. Cache: product details, images, reviews. Don't cache: inventory, prices (or very short TTL)
2. Cache-aside, TTL 5-15 min, event-based invalidation on update
3. Don't cache inventory OR cache with very short TTL + check DB on purchase
4. CDN → Local cache → Redis → DB

**6.2:**
1. 20GB raw, 30GB with overhead
2. Redis - need persistence for server restarts
3. Redis Sentinel or Redis Cluster with replicas
4. Write to primary, read from primary for same session (sticky routing or read-your-writes)

**6.3:**
1. Popular/hot keys from analytics
2. Query logs, track access frequency
3. Batch load from DB, rate limit to avoid overwhelming DB
4. Rate limiting, staggered loading, parallel but throttled

### Section 7

**Part A:** Cache-aside, Write-behind, Write-through, Write-around, lazy

**Part B:** least recently, least frequently, freshness, Random/FIFO

**Part C:** Thundering, Hot, penetration, avalanche

**Part D:** 90%, 100,000, 0.1-1, 4x

**Part E:** Redis, Memcached, Redis

---

## Scoring

**Section 7 (20 points)**:
- 18-20: Caching expert
- 14-17: Strong understanding
- 10-13: Review weak areas
- <10: Re-study concepts

---

## Next Steps

Ready for:
**[Module 5: Messaging](../../messaging/concepts/concepts.md)**