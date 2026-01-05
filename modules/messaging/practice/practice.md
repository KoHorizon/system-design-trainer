# Module 5: Messaging - Practice

Test your understanding of message queues, pub/sub, delivery guarantees, and event-driven architecture.

---

## Section 1: Pattern Selection

### Exercise 1.1: Queue vs Pub/Sub

For each scenario, choose **Queue (Q)** or **Pub/Sub (P)**:

1. ___ Send email after user registration
2. ___ Distribute video encoding tasks across workers
3. ___ Notify multiple services when order is placed
4. ___ Process payment requests one at a time
5. ___ Broadcast stock price updates to all traders
6. ___ Distribute log entries to a processing pipeline
7. ___ Update search index, analytics, and cache when product changes
8. ___ Send push notifications to a specific user

### Exercise 1.2: Sync vs Async

Should each operation be **Synchronous (S)** or **Asynchronous (A)**?

1. ___ Validate user credentials during login
2. ___ Send welcome email after registration
3. ___ Check inventory before accepting order
4. ___ Generate PDF report from data
5. ___ Charge credit card for payment
6. ___ Update recommendation engine with user activity
7. ___ Resize uploaded image to multiple sizes
8. ___ Return user profile data

---

## Section 2: Delivery Guarantees

### Exercise 2.1: Match the Guarantee

For each scenario, choose the appropriate delivery guarantee:

**Options:**
- A) At-most-once
- B) At-least-once
- C) Exactly-once (effectively)

1. ___ Payment processing
2. ___ Real-time game position updates
3. ___ Order confirmation emails
4. ___ Analytics event tracking
5. ___ Bank account transfers
6. ___ IoT sensor readings (missing one is OK)
7. ___ Inventory reservation
8. ___ Application metrics/telemetry

### Exercise 2.2: Idempotency Implementation

**Scenario:** You receive duplicate "OrderPaid" events due to at-least-once delivery.

```go
type OrderPaidEvent struct {
    OrderID       string
    Amount        float64
    TransactionID string
    Timestamp     time.Time
}
```

**Current (buggy) implementation:**
```go
func handleOrderPaid(event OrderPaidEvent) error {
    // This runs multiple times for duplicates!
    return db.Exec(`
        UPDATE orders 
        SET status = 'paid', paid_amount = paid_amount + $1 
        WHERE id = $2
    `, event.Amount, event.OrderID)
}
```

**Questions:**

1. What's wrong with the current implementation?
   ```
   ___
   ```

2. Write an idempotent version:
   ```go
   func handleOrderPaid(event OrderPaidEvent) error {
       // Your idempotent implementation
       ___
   }
   ```

### Exercise 2.3: Exactly-Once Analysis

**Scenario:** Money transfer between accounts.

```go
type TransferEvent struct {
    TransferID string
    FromAccount string
    ToAccount   string
    Amount      float64
}
```

**Questions:**

1. Why is true exactly-once delivery impossible in distributed systems?
   ```
   ___
   ```

2. How would you implement "effectively exactly-once" for this transfer?
   ```go
   func processTransfer(event TransferEvent) error {
       ___
   }
   ```

3. What happens if the system crashes after debiting but before crediting?
   ```
   ___
   ```

---

## Section 3: Ordering

### Exercise 3.1: Partition Key Selection

**Scenario:** E-commerce event streaming

Events:
- OrderCreated
- OrderItemAdded
- OrderItemRemoved
- OrderPaid
- OrderShipped
- OrderDelivered

**Questions:**

1. What partition key ensures all events for one order are ordered?
   ```
   Partition key: ___
   ```

2. If you use `customer_id` as partition key, what could go wrong?
   ```
   ___
   ```

3. You need to process "OrderPaid" before "OrderShipped". How do you guarantee this?
   ```
   ___
   ```

### Exercise 3.2: Out-of-Order Handling

**Scenario:** You receive events in this order:
```
1. OrderShipped (version: 3)
2. OrderCreated (version: 1)
3. OrderPaid (version: 2)
```

**Questions:**

1. What's wrong with processing them in received order?
   ```
   ___
   ```

2. Design a solution using versioning:
   ```go
   func processOrderEvent(event OrderEvent) error {
       ___
   }
   ```

3. What do you do when you receive version 3 but only have version 1?
   ```
   ___
   ```

---

## Section 4: Architecture Design

### Exercise 4.1: Design Order Processing Pipeline

**Requirements:**
- Orders come in at 1000/second peak
- Each order needs: payment processing, inventory reservation, email confirmation, analytics
- Payment must succeed before other steps
- Other steps can happen in parallel
- Must handle failures gracefully

**Design:**

1. Draw/describe the architecture:
   ```
   ___
   ```

2. What messaging pattern for each step?
   ```
   Payment: ___
   After payment: ___
   ```

3. How do you handle payment failure?
   ```
   ___
   ```

4. How do you handle email service being down?
   ```
   ___
   ```

### Exercise 4.2: Design Notification System

**Requirements:**
- Users can subscribe to topics (sports, news, weather)
- Publishers post to topics
- Users receive notifications on multiple devices (mobile, web, email)
- 10M users, 100K messages/day
- Delivery within 1 minute

**Design:**

1. Pub/Sub or Queue for topic→user delivery?
   ```
   ___
   Why: ___
   ```

2. How do you handle one user with 5 devices?
   ```
   ___
   ```

3. User goes offline for a week. What happens to their notifications?
   ```
   ___
   ```

4. Calculate throughput requirements:
   ```
   Messages per second: 100K / 86400 ≈ ___ msg/sec
   If each message fans out to average 1000 subscribers: ___ notifications/sec
   ```

### Exercise 4.3: Event Sourcing Design

**Scenario:** Shopping cart with event sourcing

**Events:**
- CartCreated
- ItemAdded
- ItemRemoved
- ItemQuantityChanged
- CartCheckedOut
- CartAbandoned

**Questions:**

1. Write the Cart aggregate that rebuilds state from events:
   ```go
   type Cart struct {
       ID        string
       UserID    string
       Items     map[string]CartItem
       Status    string
       Version   int
   }
   
   func (c *Cart) Apply(event Event) {
       ___
   }
   ```

2. How do you get "all carts with total > $100" efficiently?
   ```
   ___
   ```

3. A bug incorrectly calculated prices for 2 days. How do you fix it?
   ```
   ___
   ```

---

## Section 5: Failure Scenarios

### Exercise 5.1: Dead Letter Queue

**Scenario:** Messages keep failing with "Invalid product ID".

```go
type OrderEvent struct {
    OrderID   string
    ProductID string
    Quantity  int
}
```

Current behavior: Message retries forever.

**Questions:**

1. Implement DLQ handling:
   ```go
   func processOrder(queue, dlq *Queue) {
       ___
   }
   ```

2. What information should you include when sending to DLQ?
   ```
   ___
   ```

3. How would you alert operators?
   ```
   ___
   ```

4. After fixing the bug, how do you replay DLQ messages?
   ```
   ___
   ```

### Exercise 5.2: Consumer Crash Recovery

**Scenario:** Consumer crashes after processing message but before acknowledging.

```
1. Consumer receives message
2. Consumer processes message (writes to DB)
3. Consumer crashes before ACK
4. Message broker redelivers message
5. Consumer processes again (duplicate!)
```

**Questions:**

1. Why does this happen?
   ```
   ___
   ```

2. How do you prevent duplicate processing?
   ```
   ___
   ```

3. What if you can't make the operation idempotent?
   ```
   ___
   ```

### Exercise 5.3: Backpressure Handling

**Scenario:** 
- Producer: 10,000 messages/second
- Consumer: 1,000 messages/second
- Queue depth growing by 9,000/second

**Questions:**

1. How long until you run out of memory (1GB queue limit, 1KB messages)?
   ```
   Queue capacity: 1GB / 1KB = ___ messages
   Time to fill: ___ / 9000 = ___ seconds
   ```

2. What are your options?
   ```
   Option 1: ___
   Option 2: ___
   Option 3: ___
   ```

3. Implement producer-side backpressure:
   ```go
   func sendWithBackpressure(queue *Queue, msg Message) error {
       ___
   }
   ```

---

## Section 6: Technology Selection

### Exercise 6.1: Choose the Technology

For each scenario, choose the best messaging technology:

**Options:**
- A) Apache Kafka
- B) RabbitMQ
- C) Amazon SQS
- D) Redis Pub/Sub
- E) Amazon SNS + SQS

1. ___ High-throughput event streaming, need to replay events
2. ___ Simple task queue, already on AWS, minimal ops
3. ___ Complex routing rules, request-reply pattern
4. ___ Real-time notifications, ephemeral, speed matters
5. ___ Fan-out to multiple SQS queues
6. ___ Event sourcing with 1M events/second
7. ___ Simple work queue with competing consumers
8. ___ Need message ordering by customer ID

### Exercise 6.2: Kafka Design

**Scenario:** Design Kafka setup for order processing.

- 100K orders/day
- Need 7-day retention for replay
- 3 consumer services: fulfillment, analytics, notifications
- Each service has 2 instances for HA

**Questions:**

1. How many partitions?
   ```
   Orders/second: 100K / 86400 ≈ ___
   Partitions (rule of thumb: 2-3x consumers): ___
   ```

2. How do you ensure ordering per order?
   ```
   ___
   ```

3. Draw consumer group configuration:
   ```
   Topic: orders
   
   Consumer Group 1 (fulfillment):
   ___
   
   Consumer Group 2 (analytics):
   ___
   
   Consumer Group 3 (notifications):
   ___
   ```

4. One fulfillment instance crashes. What happens?
   ```
   ___
   ```

---

## Section 7: Quick Recall Quiz

### Part A: Core Concepts (5 points)

1. Queue delivers message to ___ consumer(s), Pub/Sub delivers to ___ subscriber(s)
2. At-least-once delivery requires ___ consumers to handle duplicates
3. Messages with same ___ ___ go to same Kafka partition
4. DLQ stands for ___ ___ ___
5. CQRS separates ___ model from ___ model

### Part B: Delivery Guarantees (4 points)

1. ___ delivery: may lose messages, no duplicates
2. ___ delivery: no message loss, may duplicate
3. Exactly-once is achieved with at-least-once + ___
4. ___ key ensures duplicate requests have same result

### Part C: Technologies (4 points)

1. ___ is best for high-throughput event streaming with replay
2. ___ is AWS managed queue service
3. ___ uses exchanges and bindings for routing
4. Kafka ordering is guaranteed per ___

### Part D: Patterns (4 points)

1. ___ pattern: multiple consumers share workload from one queue
2. ___ pattern: one message delivered to all subscribers
3. Event ___: store events instead of current state
4. ___ queue: stores messages that failed processing

### Part E: Numbers (3 points)

1. Kafka throughput per broker: ___K msg/sec
2. SQS max message size: ___ KB
3. SQS FIFO throughput: ___ msg/sec

---

## Answer Key

### Section 1

**1.1:**
1. Q (one email per registration)
2. Q (distribute work)
3. P (multiple services need notification)
4. Q (sequential processing)
5. P (broadcast to all)
6. Q (distribute work)
7. P (multiple services)
8. Q (specific recipient)

**1.2:**
1. S (user needs immediate response)
2. A (user doesn't wait for email)
3. S (need answer before proceeding)
4. A (long-running task)
5. S (need confirmation)
6. A (background processing)
7. A (can happen later)
8. S (user waiting for data)

### Section 2

**2.1:**
1. C (financial, no duplicates)
2. A (real-time, stale data worse than missing)
3. B (important but duplicate email is OK)
4. B or A (missing some OK)
5. C (financial)
6. A (missing OK)
7. C (can't double-reserve)
8. A (metrics, best effort)

**2.2:**
1. Amount gets added multiple times for duplicate events
2. Solution:
```go
func handleOrderPaid(event OrderPaidEvent) error {
    // Use transaction ID as idempotency key
    result, err := db.Exec(`
        UPDATE orders 
        SET status = 'paid', 
            paid_amount = $1,
            transaction_id = $2
        WHERE id = $3 
        AND (transaction_id IS NULL OR transaction_id = $2)
    `, event.Amount, event.TransactionID, event.OrderID)
    
    if rows, _ := result.RowsAffected(); rows == 0 {
        // Already processed with different transaction - log but don't fail
        log.Warn("Duplicate payment event", "order", event.OrderID)
    }
    return err
}
```

**2.3:**
1. Network can fail after message processed but before ACK received. Broker doesn't know if processed, so redelivers.
2. Use transfer ID as idempotency key, wrap in transaction
3. Need saga pattern or 2PC: either the transfer record shows incomplete state that can be recovered, or you use compensation

### Section 3

**3.1:**
1. `order_id`
2. Different orders from same customer could go to different partitions, processed out of order relative to each other (usually OK, but depends on requirements)
3. Same partition key (order_id) ensures they go to same partition and are processed in order by same consumer

**3.2:**
1. Can't ship before payment, state machine would be invalid
2. Check current version, buffer out-of-order events, or fetch missing events
3. Buffer it and wait, or fetch events 2 from event store to fill gap

### Section 4

**4.1:**
1. Orders → Sync Payment → On success publish "OrderPaid" → Fan-out to Email, Inventory, Analytics queues
2. Payment: Synchronous (critical path), After payment: Pub/Sub fan-out
3. Return error to customer, don't publish event
4. Email queue holds messages, retries with backoff, DLQ after max retries

**4.2:**
1. Pub/Sub for topic→users, then Queue for user→devices
2. Fan-out: publish to user's device queue, each device pulls from personal queue or push via websocket
3. Depends on requirements: queue with TTL, or store unread notifications in DB
4. ~1.2 msg/sec input, ~1200 notifications/sec after fan-out

**4.3:**
1. Apply method with switch on event type, update Items map accordingly
2. CQRS: project events to read model optimized for queries, or maintain materialized view
3. Fix bug, replay all events to rebuild correct state (event sourcing benefit!)

### Section 5

**5.1:**
1. Check retry count, send to DLQ after max retries, include original message + errors
2. Original message, all error messages, retry count, timestamps, consumer info
3. Alert on DLQ depth increase, include message samples in alert
4. Read from DLQ, attempt to reprocess, send back to DLQ if still failing

**5.2:**
1. Processing and ACK are not atomic
2. Idempotent processing: check if already processed before doing work
3. Use transactional outbox: write to DB and outbox in same transaction

**5.3:**
1. 1M messages capacity, ~111 seconds to fill
2. Drop messages, block producer, scale consumers, increase queue size
3. Check queue depth, implement exponential backoff or circuit breaker

### Section 6

**6.1:**
1. A (Kafka - replay capability)
2. C (SQS - managed, simple)
3. B (RabbitMQ - routing)
4. D (Redis - real-time, ephemeral)
5. E (SNS+SQS - fan-out)
6. A (Kafka - high throughput)
7. C or B (SQS or RabbitMQ)
8. A (Kafka with partition key)

**6.2:**
1. ~1.2 msg/sec, 6 partitions (2x3 consumer instances)
2. Use order_id as partition key
3. Each consumer group has 2 instances, partitions distributed between them
4. Kafka rebalances, surviving instance takes over failed instance's partitions

### Section 7

**Part A:** one, all; idempotent; partition key; Dead Letter Queue; write/command, read/query

**Part B:** At-most-once; At-least-once; idempotency; Idempotency

**Part C:** Kafka; SQS; RabbitMQ; partition

**Part D:** Competing consumers; Fan-out/Pub-Sub; sourcing; Dead letter

**Part E:** 100-200K; 256; 300

---

## Scoring

**Section 7 (20 points)**:
- 18-20: Messaging expert
- 14-17: Strong understanding
- 10-13: Review weak areas
- <10: Re-study concepts

---

## Next Steps

Ready for:
**[Module 6: Storage](../../storage/concepts/concepts.md)**