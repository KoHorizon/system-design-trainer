# System Design Trainer

A structured learning tool for mastering system design concepts and preparing for technical interviews.

## Philosophy

- **Concrete over abstract**: Real numbers, not hand-wavy "it's fast"
- **Trade-offs over best practices**: There's rarely one right answer
- **Why it breaks > What to do**: Understanding failure modes builds intuition
- **Diagrams are essential**: Architecture is visual

## Project Structure

```
system-design-trainer/
├── README.md
└── modules/
    └── [module-name]/
        ├── concepts/concepts.md    # Theory + diagrams + numbers
        └── practice/practice.md    # Exercises + estimation + recall
```

## Modules

| # | Module | Focus |
|---|--------|-------|
| 1 | fundamentals | Latency numbers, throughput, availability, back-of-envelope math |
| 2 | scaling | Vertical vs horizontal, stateless design, load balancing |
| 3 | databases | SQL vs NoSQL, ACID, indexing, sharding, replication |
| 4 | caching | Strategies, eviction, invalidation patterns |
| 5 | messaging | Queues, pub/sub, event sourcing, delivery guarantees |
| 6 | storage | Block/object/file, blob storage, hot/cold tiers |
| 7 | networking | DNS, CDN, reverse proxies, API gateways |
| 8 | api-design | REST/GraphQL/gRPC, pagination, versioning |
| 9 | consistency | CAP, PACELC, consensus, distributed transactions |
| 10 | rate-limiting | Token bucket, sliding window, distributed limiting |
| 11 | reliability | Redundancy, failover, circuit breakers, retries, idempotency |
| 12 | search | Inverted indexes, full-text search, relevance |
| 13 | real-time | WebSockets, SSE, long polling, fan-out |
| 14 | observability | Logging, metrics, tracing, alerting, SLOs |
| 15 | security | Auth patterns, encryption, secrets management |
| 16 | case-studies | URL shortener, rate limiter, chat, news feed |

## Usage

Open any module folder and read:
1. `concepts/concepts.md` - Learn the theory
2. `practice/practice.md` - Test yourself

## Study Approach

1. **Read concepts.md** - Understand the theory, memorize key numbers
2. **Draw the diagrams** - Can you reproduce them from memory?
3. **Do practice.md** - Test your understanding
4. **Teach it back** - Explain to a rubber duck or friend

## Key Numbers to Memorize

These appear throughout the modules but here's the cheat sheet:

| Operation | Latency |
|-----------|---------|
| L1 cache reference | 1 ns |
| L2 cache reference | 4 ns |
| Main memory reference | 100 ns |
| SSD random read | 100 μs |
| HDD seek | 10 ms |
| Network round trip (same datacenter) | 0.5 ms |
| Network round trip (cross-continent) | 150 ms |

| Throughput | Value |
|------------|-------|
| SSD sequential read | 500 MB/s |
| HDD sequential read | 100 MB/s |
| 1 Gbps network | 125 MB/s |
| 10 Gbps network | 1.25 GB/s |

## License

MIT - Use this to ace your interviews!