# Module 2: Scaling - Practice

Test your understanding of vertical vs horizontal scaling, stateless design, and load balancing.

---

## Section 1: Concept Check

### Exercise 1.1: Vertical vs Horizontal

For each scenario, choose **Vertical (V)** or **Horizontal (H)** scaling:

1. ___ Your single PostgreSQL database is at 90% CPU
2. ___ Your 3 web servers can't handle traffic spikes
3. ___ Your Redis cache needs more memory
4. ___ Your API response times are slow due to compute-heavy image processing
5. ___ Your real-time game server needs to handle more concurrent players
6. ___ Your legacy monolith can't be easily distributed

### Exercise 1.2: Stateful vs Stateless

Identify if each component is **Stateful (SF)** or **Stateless (SL)**:

1. ___ Web server storing user sessions in memory
2. ___ Web server reading sessions from Redis
3. ___ Database primary server
4. ___ Lambda function processing images
5. ___ WebSocket server maintaining connections
6. ___ REST API server with no local storage

### Exercise 1.3: Load Balancer Selection

Choose the appropriate load balancer type for each scenario:

**Options**: L4 (Layer 4), L7 (Layer 7)

1. ___ Route `/api/*` to API servers and `/web/*` to web servers
2. ___ Balance TCP traffic for a database proxy
3. ___ Terminate SSL and add custom headers
4. ___ Maximum throughput for a gaming server
5. ___ A/B testing based on cookies
6. ___ gRPC load balancing with connection multiplexing

---

## Section 2: Load Balancing Algorithms

### Exercise 2.1: Algorithm Matching

Match each scenario to the best load balancing algorithm:

**Algorithms**: 
- A) Round Robin
- B) Weighted Round Robin  
- C) Least Connections
- D) IP Hash
- E) Least Response Time

**Scenarios**:

1. ___ Three identical servers, uniform request cost
2. ___ Mix of m5.large and m5.4xlarge instances
3. ___ Video processing requests that vary from 100ms to 10s
4. ___ Need to maintain session affinity without cookies
5. ___ Latency-critical financial trading API

### Exercise 2.2: Calculate the Distribution

Given: 3 servers with weights:
- Server A: weight 4
- Server B: weight 2  
- Server C: weight 1

Using **Weighted Round Robin**, how many of the next 14 requests go to each server?

```
Server A: ___ requests
Server B: ___ requests
Server C: ___ requests
```

### Exercise 2.3: Least Connections Simulation

Current state:
- Server 1: 50 active connections, capacity 100
- Server 2: 30 active connections, capacity 100
- Server 3: 45 active connections, capacity 100

Five new requests arrive. Using **Least Connections**, trace where each goes:

```
Request 1 → Server ___ (connections after: S1=___, S2=___, S3=___)
Request 2 → Server ___ (connections after: S1=___, S2=___, S3=___)
Request 3 → Server ___ (connections after: S1=___, S2=___, S3=___)
Request 4 → Server ___ (connections after: S1=___, S2=___, S3=___)
Request 5 → Server ___ (connections after: S1=___, S2=___, S3=___)
```

---

## Section 3: Architecture Design Exercises

### Exercise 3.1: Make It Stateless

The following web server stores state locally. Refactor to be stateless.

**Current (stateful)**:
```python
# In-memory storage on each server
user_sessions = {}
shopping_carts = {}
rate_limits = {}

def login(user_id, password):
    if authenticate(user_id, password):
        user_sessions[user_id] = {
            "logged_in_at": time.now(),
            "ip": request.ip
        }
        return session_token(user_id)

def add_to_cart(user_id, item):
    if user_id not in shopping_carts:
        shopping_carts[user_id] = []
    shopping_carts[user_id].append(item)

def check_rate_limit(ip):
    if ip not in rate_limits:
        rate_limits[ip] = {"count": 0, "window_start": time.now()}
    rate_limits[ip]["count"] += 1
    return rate_limits[ip]["count"] < 100
```

**Your refactored version**:
```python
# What external stores would you use?
# How would each function change?

def login(user_id, password):
    # Your code here
    pass

def add_to_cart(user_id, item):
    # Your code here
    pass

def check_rate_limit(ip):
    # Your code here
    pass
```

### Exercise 3.2: Design a Scaling Strategy

**Scenario**: E-commerce site preparing for Black Friday

Current state:
- 10 web servers (each handles 2,000 QPS)
- 1 PostgreSQL primary (handles 5,000 write QPS, currently at 2,000)
- 2 PostgreSQL read replicas (each handles 10,000 read QPS)
- 1 Redis cluster (handles 100,000 QPS, currently at 30,000)

Expected Black Friday traffic:
- 5x normal traffic
- 10x traffic during 2-hour flash sales

**Questions**:

1. Current capacity:
   - Web tier total QPS: ___
   - Normal traffic (assumed): ___
   - Can web tier handle 5x? ___

2. During flash sale (10x traffic), what's the bottleneck?
   ```
   Web tier needed: ___ servers
   DB write capacity: ___ (sufficient/insufficient)
   DB read capacity: ___ (sufficient/insufficient)
   Redis capacity: ___ (sufficient/insufficient)
   ```

3. What would you scale and how?
   ```
   Web tier: ___
   Database: ___
   Cache: ___
   ```

### Exercise 3.3: Multi-Region Architecture

Design a load balancing strategy for a service deployed in 3 regions:
- US-East
- US-West  
- Europe

**Requirements**:
- Users should be routed to nearest region
- If a region fails, traffic should failover to others
- Some requests must go to US-East (contains primary database)

**Draw or describe your solution**:
```
Components needed:
- Global load balancing: ___
- Regional load balancing: ___
- Failover mechanism: ___
- Handling database writes: ___
```

---

## Section 4: What's Wrong With This Design?

### Exercise 4.1: The Overloaded Database

```
                    ┌──────────────┐
                    │   Clients    │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ Load Balancer│
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           │               │               │
    ┌──────▼─────┐  ┌──────▼─────┐  ┌──────▼─────┐
    │ Web Server │  │ Web Server │  │ Web Server │
    │   (50)     │  │   (50)     │  │   (50)     │
    └──────┬─────┘  └──────┬─────┘  └──────┬─────┘
           │               │               │
           └───────────────┼───────────────┘
                           │
                    ┌──────▼───────┐
                    │  PostgreSQL  │
                    │  (1 server)  │
                    └──────────────┘

Traffic increased 10x. Team added 50 web servers (now 150 total).
Application is still slow.
```

**Questions**:
1. What's wrong with this approach?
2. Why didn't adding web servers help?
3. What should they have done instead?

### Exercise 4.2: The Sticky Session Problem

```python
# Load balancer config
load_balancer:
  algorithm: ip_hash  # Sticky sessions
  
# Application code
class WebServer:
    def __init__(self):
        self.user_data = {}  # Local cache
    
    def get_user_profile(self, user_id):
        if user_id in self.user_data:
            return self.user_data[user_id]  # Fast!
        
        profile = database.get_user(user_id)
        self.user_data[user_id] = profile  # Cache locally
        return profile
    
    def update_user_profile(self, user_id, data):
        database.update_user(user_id, data)
        self.user_data[user_id] = data  # Update local cache
```

**Questions**:
1. What happens when Server 1 (with User A's cached data) goes down?
2. What happens when User A's IP address changes (mobile network)?
3. How would you fix this while keeping the caching benefit?

### Exercise 4.3: The Auto-Scaling Disaster

```yaml
# Auto-scaling configuration
auto_scaling:
  min_instances: 2
  max_instances: 100
  scale_up:
    metric: cpu_utilization
    threshold: 80%
    cooldown: 30s
  scale_down:
    metric: cpu_utilization  
    threshold: 20%
    cooldown: 30s
```

**Scenario**: Traffic spike at 9 AM causes scaling from 2 to 50 instances. At 9:05 AM, traffic drops slightly. System oscillates between 20 and 50 instances for the next hour.

**Questions**:
1. Why is the system oscillating?
2. What configuration changes would fix this?
3. What additional safeguards would you add?

### Exercise 4.4: The Unbalanced Load Balancer

```
Health check results:
- Server 1: Healthy (200 OK in 50ms)
- Server 2: Healthy (200 OK in 50ms)  
- Server 3: Healthy (200 OK in 50ms)

Actual request latencies:
- Server 1: p50=100ms, p99=500ms
- Server 2: p50=100ms, p99=500ms
- Server 3: p50=2000ms, p99=10000ms

Load balancer algorithm: Round Robin
```

**Questions**:
1. Why does Server 3 pass health checks but perform poorly?
2. What's the impact on users?
3. How would you improve the health check?
4. What algorithm might help here?

---

## Section 5: Capacity Planning

### Exercise 5.1: Server Sizing

**Requirements**:
- Target: 100,000 QPS
- Each web server handles 2,500 QPS
- Want 50% headroom for traffic spikes
- Need N+2 redundancy (survive 2 server failures)

**Calculate**:
```
Base servers needed: 100,000 / 2,500 = ___
With 50% headroom: ___ × 1.5 = ___
With N+2 redundancy: ___ + 2 = ___
```

### Exercise 5.2: Load Balancer Capacity

**Scenario**: You need to choose between AWS ALB and NLB.

**Requirements**:
- 500,000 concurrent connections
- 200,000 new connections per second
- Need URL-based routing
- SSL termination required

**Limits**:
| Feature | ALB | NLB |
|---------|-----|-----|
| New connections/sec | 25,000 | 1,000,000 |
| Concurrent connections | 100,000 | Millions |
| URL routing | Yes | No |
| SSL termination | Yes | Yes |

**Questions**:
1. Can a single ALB handle your requirements?
2. Can a single NLB handle your requirements?
3. What architecture would you propose?

### Exercise 5.3: WebSocket Scaling

**Requirements**:
- 1 million concurrent WebSocket connections
- Each server can handle 50,000 connections
- 99.99% availability required

**Calculate**:
```
Minimum servers needed: 1,000,000 / 50,000 = ___
For 99.99% availability with server failure tolerance: ___
Recommended total: ___
```

**Additional question**: Can you use a standard ALB for this? Why or why not?

---

## Section 6: Trade-off Questions

### Exercise 6.1: Scale Up vs Scale Out

**Scenario**: Your database server (32 CPU, 256GB RAM) is at 80% capacity.

**Option A**: Upgrade to 64 CPU, 512GB RAM ($3,000/month more)
**Option B**: Implement read replicas and application-level sharding (2 months engineering time)

**Questions**:
1. What factors would influence your decision?
2. If you're 3 months from a major launch, which would you choose?
3. If you're expecting 10x growth in 12 months, which would you choose?

### Exercise 6.2: Managed vs Self-Hosted Load Balancer

**Option A**: AWS ALB ($20/month + $0.008 per LCU-hour)
**Option B**: Self-hosted HAProxy on EC2 ($200/month for HA pair)

**Questions**:
1. At what traffic level does self-hosted become cheaper?
2. What hidden costs exist for self-hosted?
3. When would you choose self-hosted despite higher cost?

### Exercise 6.3: Session Storage Trade-offs

**Options for session storage**:

| Option | Latency | Cost | Complexity | Durability |
|--------|---------|------|------------|------------|
| JWT (stateless) | 0ms | Free | Low | N/A |
| Redis | 1ms | Medium | Medium | Optional |
| Database | 5ms | Low | Low | High |
| Sticky sessions | 0ms | Free | Low | Low |

**Scenarios** - choose the best option:

1. ___ Banking app with strict audit requirements
2. ___ High-traffic social media API
3. ___ Simple internal admin tool
4. ___ Mobile app with frequent session checks

---

## Section 7: Quick Recall Quiz

Answer without looking at notes.

### Part A: Vertical vs Horizontal (5 points)

1. Maximum AWS instance size (vCPUs): ___
2. Which is typically cheaper at scale: V or H? ___
3. Which requires application changes: V or H? ___
4. Which gives you redundancy by default: V or H? ___
5. Best for databases: V or H (initially)? ___

### Part B: Stateless Design (5 points)

1. Where should user sessions be stored in stateless architecture? ___
2. What's wrong with sticky sessions? (one problem) ___
3. Name two external state stores: ___, ___
4. Can a WebSocket server be stateless? ___

### Part C: Load Balancing (5 points)

1. Layer 4 LB can do SSL termination: True/False? ___
2. Best algorithm for mixed-size servers: ___
3. Typical health check interval: ___
4. Layer 7 LB overhead compared to Layer 4: ___
5. AWS L4 load balancer is called: ___

### Part D: Scaling Numbers (5 points)

1. Typical web server QPS capacity: ___
2. Typical WebSocket connections per server: ___
3. When scaling 10 servers to 20, expected throughput increase: ___
4. Auto-scaling cooldown (typical): ___
5. N+2 redundancy means: ___

---

## Answer Key

### Section 1: Concept Check

**1.1 Vertical vs Horizontal:**
1. V (database scaling is complex, start vertical)
2. H (web servers are stateless, easy to scale out)
3. V (adding RAM to existing cache is simpler)
4. H (CPU-bound work distributes well)
5. H (game servers are typically stateless per connection)
6. V (can't easily distribute a monolith)

**1.2 Stateful vs Stateless:**
1. SF (local session storage)
2. SL (external state store)
3. SF (holds the data)
4. SL (no state between invocations)
5. SF (connection state)
6. SL (no local storage)

**1.3 Load Balancer Selection:**
1. L7 (URL routing requires application layer)
2. L4 (simple TCP)
3. L7 (SSL and header modification)
4. L4 (maximum performance)
5. L7 (cookie inspection)
6. L7 (gRPC is HTTP/2)

### Section 2: Load Balancing Algorithms

**2.1:** 1-A, 2-B, 3-C, 4-D, 5-E

**2.2:**
- Server A: 8 requests (4/7 × 14)
- Server B: 4 requests (2/7 × 14)
- Server C: 2 requests (1/7 × 14)

**2.3:**
```
Request 1 → Server 2 (S1=50, S2=31, S3=45)
Request 2 → Server 2 (S1=50, S2=32, S3=45)
Request 3 → Server 2 (S1=50, S2=33, S3=45)
Request 4 → Server 2 (S1=50, S2=34, S3=45)
Request 5 → Server 2 (S1=50, S2=35, S3=45)
```
(All go to Server 2 because it stays the lowest)

### Section 3: Architecture Design

**3.1 Stateless Refactor:**
```python
# External stores
redis = RedisClient()
database = DatabaseClient()

def login(user_id, password):
    if authenticate(user_id, password):
        session_data = {
            "logged_in_at": time.now(),
            "ip": request.ip
        }
        token = generate_token(user_id)
        redis.setex(f"session:{user_id}", 3600, json.dumps(session_data))
        return token

def add_to_cart(user_id, item):
    cart_key = f"cart:{user_id}"
    redis.rpush(cart_key, json.dumps(item))
    redis.expire(cart_key, 86400)  # 24 hour expiry

def check_rate_limit(ip):
    key = f"rate:{ip}"
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, 60)  # 1 minute window
    return count < 100
```

**3.2 Scaling Strategy:**
1. Web tier: 20,000 QPS total, normal ~4,000 QPS, yes can handle 5x (20K)
2. Flash sale (10x = 40,000 QPS):
   - Web tier: Need 20 servers (insufficient with 10)
   - DB writes: 2,000 × 10 = 20,000 (insufficient, max 5,000)
   - DB reads: Depends on read ratio (likely insufficient)
   - Redis: 300,000 QPS needed (insufficient, max 100,000)
3. Scale: Add 15+ web servers, add Redis nodes, add read replicas, implement write queuing

**3.3 Multi-Region:**
- Global: GeoDNS or Anycast (Route 53, Cloudflare)
- Regional: ALB per region
- Failover: Health-check based DNS failover
- Writes: Route to US-East via internal routing or global accelerator

### Section 4: What's Wrong

**4.1:**
1. Scaled the wrong tier - database is the bottleneck
2. All 150 servers still hitting same database
3. Add caching (Redis), read replicas, or database sharding

**4.2:**
1. User loses cached data, must refetch everything from DB
2. User gets routed to different server, loses cache
3. Use Redis for caching instead of local memory

**4.3:**
1. Scale-down cooldown too short (30s), traffic naturally fluctuates
2. Increase scale-down cooldown to 300s, use step scaling
3. Add minimum healthy hosts, predictive scaling, scale-down percentage limits

**4.4:**
1. Health check only tests if server responds, not actual performance
2. 1/3 of users get slow responses (round robin is even)
3. Add response time to health check, or check actual endpoint performance
4. Use Least Response Time algorithm

### Section 5: Capacity Planning

**5.1:**
- Base: 100,000 / 2,500 = 40
- Headroom: 40 × 1.5 = 60
- Redundancy: 60 + 2 = **62 servers**

**5.2:**
1. No (25K new conn/sec vs 200K needed)
2. Yes for connections, but no URL routing
3. Multiple ALBs behind NLB, or split traffic by path at DNS/CDN level

**5.3:**
- Minimum: 1,000,000 / 50,000 = 20
- For 99.99%: Need at least 25% overhead = 25 servers
- Recommended: **25-30 servers**
- ALB: No, 100K concurrent limit per ALB; need NLB or multiple ALBs

### Section 6: Trade-offs

**6.1:**
1. Time to implement, expected growth, engineering capacity
2. Option A (quick fix for launch)
3. Option B (need long-term scalability)

**6.2:**
1. Roughly 500K+ requests/hour (depends on LCU calculation)
2. Maintenance, monitoring, upgrades, on-call
3. Need features not available in managed (custom algorithms, protocols)

**6.3:**
1. Database (audit trail, durability)
2. Redis (fast, scalable)
3. Sticky sessions or JWT (simple, low traffic)
4. JWT (no server round-trip)

### Section 7: Quick Recall

**Part A:** 448, H, H, H, V
**Part B:** Redis/Memcached/external store; server failure loses sessions; Redis + Database (or Memcached, S3); No (connections are stateful)
**Part C:** False, Weighted Round Robin, 5-30 seconds, Higher/slower (~1-2ms), NLB
**Part D:** 1,000-10,000 QPS, 10,000-100,000, Less than 2x (diminishing returns), 60-300 seconds, survives 2 failures

---

## Scoring Guide

**Section 7 (20 points total)**:
- 18-20: Ready for scaling discussions in interviews
- 14-17: Good foundation, review gaps
- 10-13: Re-read concepts, focus on weak areas
- <10: Study the concepts module again

---

## Next Steps

Once comfortable with scaling concepts, move to:
**[Module 3: Databases](../databases/concepts/concepts.md)**