# Module 5: Messaging

> "The best way to make a system faster is to do less work. The best way to do less work is to do it later."

Messaging decouples systems, absorbs traffic spikes, and enables asynchronous processing. This module covers queues, pub/sub, event sourcing, and the critical topic of delivery guarantees.

---

## 1. Why Messaging?

### The Problem with Synchronous Communication

```mermaid
sequenceDiagram
    participant User
    participant API
    participant Payment
    participant Inventory
    participant Email
    participant Analytics
    
    User->>API: Place Order
    API->>Payment: Charge Card
    Payment-->>API: Success (200ms)
    API->>Inventory: Reserve Items
    Inventory-->>API: Success (150ms)
    API->>Email: Send Confirmation
    Email-->>API: Success (300ms)
    API->>Analytics: Log Event
    Analytics-->>API: Success (100ms)
    API-->>User: Order Complete (750ms total)
    
    Note over User,Analytics: User waits for ALL services
```

**Problems:**
- User waits 750ms (sum of all services)
- If Email service is slow/down, order fails
- Tight coupling between services
- No retry mechanism
- Can't handle traffic spikes

### The Solution: Asynchronous Messaging

```mermaid
sequenceDiagram
    participant User
    participant API
    participant Queue
    participant Payment
    participant Email
    participant Analytics
    
    User->>API: Place Order
    API->>Payment: Charge Card
    Payment-->>API: Success (200ms)
    API->>Queue: Publish OrderCreated
    API-->>User: Order Placed! (250ms)
    
    Note over Queue,Analytics: Async processing
    Queue-->>Email: Process
    Queue-->>Analytics: Process
    Email-->>Email: Send (can retry)
    Analytics-->>Analytics: Log (can retry)
```

**Benefits:**
- User response: 250ms (only critical path)
- Email/Analytics failures don't affect user
- Services are decoupled
- Built-in retry and failure handling
- Queue absorbs traffic spikes

### When to Use Messaging

| Use Case | Sync or Async? | Why |
|----------|----------------|-----|
| User authentication | Sync | User needs immediate response |
| Payment processing | Sync | Critical, needs confirmation |
| Send email | Async | User doesn't need to wait |
| Generate report | Async | Long-running, do in background |
| Update search index | Async | Eventually consistent is fine |
| Real-time chat | Async (push) | Fire and forget to recipient |
| Log analytics | Async | Non-critical, can batch |

---

## 2. Message Queues (Point-to-Point)

### What is a Message Queue?

A queue holds messages until consumers process them. Each message is delivered to **one** consumer.

```mermaid
flowchart LR
    subgraph Producers
        P1[Producer 1]
        P2[Producer 2]
    end
    
    Q[(Queue<br/>FIFO)]
    
    subgraph Consumers
        C1[Consumer 1]
        C2[Consumer 2]
    end
    
    P1 & P2 -->|Send| Q
    Q -->|Receive| C1 & C2
    
    Note1[Each message goes<br/>to ONE consumer]
```

### Basic Queue Operations

```go
// Producer
func sendOrder(queue *Queue, order Order) error {
    msg := Message{
        ID:   uuid.New().String(),
        Body: serialize(order),
        Metadata: map[string]string{
            "type":      "order.created",
            "timestamp": time.Now().Format(time.RFC3339),
        },
    }
    return queue.Send(ctx, msg)
}

// Consumer
func processOrders(queue *Queue) {
    for {
        msg, err := queue.Receive(ctx) // Blocks until message available
        if err != nil {
            log.Error(err)
            continue
        }
        
        order := deserialize(msg.Body)
        if err := processOrder(order); err != nil {
            queue.Nack(ctx, msg) // Return to queue for retry
            continue
        }
        
        queue.Ack(ctx, msg) // Remove from queue
    }
}
```

### Queue Patterns

#### Work Queue (Competing Consumers)

Multiple consumers share the workload. Each message processed by exactly one consumer.

```mermaid
flowchart LR
    P[Producer<br/>100 msg/sec]
    Q[(Queue)]
    C1[Worker 1<br/>25 msg/sec]
    C2[Worker 2<br/>25 msg/sec]
    C3[Worker 3<br/>25 msg/sec]
    C4[Worker 4<br/>25 msg/sec]
    
    P --> Q
    Q --> C1 & C2 & C3 & C4
```

**Use case:** Distribute CPU-intensive tasks across workers

```go
// Scale workers based on queue depth
func autoScale(queue *Queue, workerPool *Pool) {
    depth := queue.Depth()
    
    if depth > 10000 && workerPool.Size() < maxWorkers {
        workerPool.Add(1)
    } else if depth < 100 && workerPool.Size() > minWorkers {
        workerPool.Remove(1)
    }
}
```

#### Request-Reply

Send request, wait for response on reply queue.

```mermaid
sequenceDiagram
    participant Client
    participant RequestQ as Request Queue
    participant Server
    participant ReplyQ as Reply Queue
    
    Client->>RequestQ: Send(msg, replyTo=reply-123)
    RequestQ->>Server: Receive
    Server->>Server: Process
    Server->>ReplyQ: Send(response, correlationId=msg.id)
    ReplyQ->>Client: Receive (filtered by correlationId)
```

```go
func requestReply(requestQ, replyQ *Queue, request Request) (*Response, error) {
    correlationID := uuid.New().String()
    
    // Send request
    msg := Message{
        ID:            uuid.New().String(),
        CorrelationID: correlationID,
        ReplyTo:       replyQ.Name,
        Body:          serialize(request),
    }
    requestQ.Send(ctx, msg)
    
    // Wait for reply (with timeout)
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()
    
    for {
        reply, err := replyQ.Receive(ctx)
        if err != nil {
            return nil, err
        }
        if reply.CorrelationID == correlationID {
            return deserialize(reply.Body), nil
        }
        replyQ.Nack(ctx, reply) // Not ours, put back
    }
}
```

### Queue Technologies

| Technology | Type | Best For | Throughput |
|------------|------|----------|------------|
| RabbitMQ | Traditional broker | Complex routing, reliability | 10K-50K msg/sec |
| Amazon SQS | Managed service | Simple queuing, AWS integration | 3K msg/sec (standard) |
| Redis Lists | In-memory | Simple, fast, ephemeral | 100K+ msg/sec |
| ZeroMQ | Brokerless | Low latency, embedded | 1M+ msg/sec |

---

## 3. Pub/Sub (Publish-Subscribe)

### What is Pub/Sub?

Publishers send messages to topics. Subscribers receive **all** messages from topics they subscribe to.

```mermaid
flowchart TB
    subgraph Publishers
        P1[Order Service]
        P2[User Service]
    end
    
    subgraph "Topics"
        T1[order.created]
        T2[user.registered]
    end
    
    subgraph Subscribers
        S1[Email Service]
        S2[Analytics]
        S3[Inventory]
    end
    
    P1 -->|Publish| T1
    P2 -->|Publish| T2
    T1 -->|Deliver| S1 & S2 & S3
    T2 -->|Deliver| S1 & S2
```

**Key difference from queues:** Each subscriber gets a **copy** of every message.

### Pub/Sub Patterns

#### Fan-Out

One message delivered to many subscribers.

```mermaid
flowchart LR
    P[Order Created] --> T[Topic]
    T --> S1[Email: Send confirmation]
    T --> S2[Inventory: Reserve stock]
    T --> S3[Analytics: Track conversion]
    T --> S4[Fraud: Check order]
    T --> S5[Shipping: Prepare label]
```

```go
// Publisher (Order Service)
func createOrder(order Order) error {
    // Save to database
    if err := db.SaveOrder(order); err != nil {
        return err
    }
    
    // Publish event - all subscribers notified
    event := Event{
        Type:      "order.created",
        Timestamp: time.Now(),
        Data:      order,
    }
    return pubsub.Publish(ctx, "orders", event)
}

// Subscriber (Email Service)
func subscribeToOrders(pubsub *PubSub) {
    sub := pubsub.Subscribe(ctx, "orders")
    
    for msg := range sub.Channel() {
        var event Event
        json.Unmarshal(msg.Payload, &event)
        
        if event.Type == "order.created" {
            sendOrderConfirmation(event.Data)
        }
    }
}
```

#### Fan-In

Many publishers, aggregated by subscriber.

```mermaid
flowchart LR
    subgraph "IoT Sensors"
        S1[Sensor 1]
        S2[Sensor 2]
        S3[Sensor N...]
    end
    
    T[sensor.readings]
    AGG[Aggregator Service]
    
    S1 & S2 & S3 --> T
    T --> AGG
```

### Consumer Groups

Multiple instances share the load while ensuring each message is processed once per group.

```mermaid
flowchart TB
    T[Topic: orders]
    
    subgraph "Consumer Group: email-service"
        E1[Instance 1]
        E2[Instance 2]
    end
    
    subgraph "Consumer Group: analytics"
        A1[Instance 1]
        A2[Instance 2]
        A3[Instance 3]
    end
    
    T -->|Partition 0| E1
    T -->|Partition 1| E2
    T -->|Partition 0| A1
    T -->|Partition 1| A2
    T -->|Partition 2| A3
```

**Each consumer group** gets all messages, but within a group, messages are distributed.

### Pub/Sub Technologies

| Technology | Type | Best For | Throughput |
|------------|------|----------|------------|
| Apache Kafka | Distributed log | High throughput, replay | 1M+ msg/sec |
| Amazon SNS | Managed | AWS integration, simple fan-out | 30M+ msg/sec |
| Google Pub/Sub | Managed | GCP integration, global | 1M+ msg/sec |
| Redis Pub/Sub | In-memory | Real-time, ephemeral | 1M+ msg/sec |
| NATS | Lightweight | Microservices, low latency | 10M+ msg/sec |

---

## 4. Delivery Guarantees

### The Three Guarantees

```mermaid
flowchart TB
    subgraph "At-Most-Once"
        AMO[Send and forget<br/>May lose messages]
    end
    
    subgraph "At-Least-Once"
        ALO[Retry until ACK<br/>May duplicate messages]
    end
    
    subgraph "Exactly-Once"
        EO[At-least-once +<br/>Idempotent processing]
    end
    
    AMO -->|"+ Retries"| ALO
    ALO -->|"+ Idempotency"| EO
```

### At-Most-Once

**Definition:** Message delivered zero or one time. May be lost.

```go
// At-most-once: Fire and forget
func sendMetric(topic string, metric Metric) {
    msg := serialize(metric)
    pubsub.Publish(ctx, topic, msg) // Don't check error, don't retry
}
```

**Use when:**
- Metrics/telemetry (missing one data point is OK)
- Real-time gaming (stale data is worse than missing data)
- Logging (best effort is fine)

**Trade-off:** Fast, simple, but data loss possible

### At-Least-Once

**Definition:** Message delivered one or more times. Never lost, but may duplicate.

```go
// At-least-once: Retry until success
func sendOrder(queue *Queue, order Order) error {
    msg := Message{ID: uuid.New().String(), Body: serialize(order)}
    
    for retries := 0; retries < maxRetries; retries++ {
        err := queue.Send(ctx, msg)
        if err == nil {
            return nil
        }
        time.Sleep(backoff(retries))
    }
    return errors.New("failed to send after retries")
}

// Consumer must handle duplicates!
func processOrder(msg Message) error {
    order := deserialize(msg.Body)
    
    // Check if already processed
    if processed, _ := cache.Get(ctx, "processed:"+msg.ID); processed != "" {
        log.Info("Duplicate message, skipping", "id", msg.ID)
        return nil // Already processed
    }
    
    // Process order...
    if err := doProcess(order); err != nil {
        return err // Will be retried
    }
    
    // Mark as processed
    cache.Set(ctx, "processed:"+msg.ID, "true", 24*time.Hour)
    return nil
}
```

**Use when:**
- Order processing
- Payment handling
- Any business-critical operation

**Trade-off:** Reliable, but requires idempotent consumers

### Exactly-Once

**Definition:** Message delivered and processed exactly one time.

**The truth:** True exactly-once is impossible in distributed systems. What we actually implement is "effectively exactly-once" = at-least-once delivery + idempotent processing.

```go
// Exactly-once semantics via idempotency
func processPayment(msg Message) error {
    payment := deserialize(msg.Body)
    
    // Idempotency key prevents duplicate processing
    idempotencyKey := payment.IdempotencyKey
    
    // Use database transaction for atomicity
    tx, _ := db.Begin()
    defer tx.Rollback()
    
    // Check if already processed (within transaction)
    var exists bool
    tx.QueryRow("SELECT EXISTS(SELECT 1 FROM processed_payments WHERE key = $1)", 
        idempotencyKey).Scan(&exists)
    
    if exists {
        return nil // Already processed
    }
    
    // Process payment
    if err := chargeCard(payment); err != nil {
        return err
    }
    
    // Mark as processed (atomically with the operation)
    tx.Exec("INSERT INTO processed_payments (key, processed_at) VALUES ($1, $2)",
        idempotencyKey, time.Now())
    
    return tx.Commit()
}
```

**Techniques for exactly-once:**

| Technique | Description | Trade-off |
|-----------|-------------|-----------|
| Idempotency keys | Client provides unique key | Client must generate keys |
| Deduplication window | Broker tracks recent message IDs | Limited time window |
| Transactional outbox | Atomically write to DB + outbox | More complex |
| Event sourcing | Derive state from event log | Different architecture |

### Delivery Guarantee Comparison

| Guarantee | Message Loss | Duplicates | Complexity | Use Case |
|-----------|--------------|------------|------------|----------|
| At-most-once | Possible | No | Low | Metrics, logs |
| At-least-once | No | Possible | Medium | Most business logic |
| Exactly-once | No | No | High | Financial transactions |

---

## 5. Message Ordering

### The Ordering Problem

```mermaid
sequenceDiagram
    participant P as Producer
    participant Q as Queue (2 partitions)
    participant C1 as Consumer 1
    participant C2 as Consumer 2
    
    P->>Q: Order Created (ID: 1)
    P->>Q: Order Updated (ID: 1)
    P->>Q: Order Cancelled (ID: 1)
    
    Note over Q: Messages distributed across partitions
    
    Q->>C1: Order Cancelled
    Q->>C2: Order Created
    Q->>C1: Order Updated
    
    Note over C1,C2: Wrong order! Cancelled processed first
```

### Ordering Guarantees by System

| System | Ordering Guarantee |
|--------|-------------------|
| SQS Standard | No ordering |
| SQS FIFO | Per message group |
| Kafka | Per partition |
| RabbitMQ | Per queue |
| Kinesis | Per shard |

### Partition-Based Ordering

Messages with the same partition key go to the same partition → same consumer → ordered.

```mermaid
flowchart TB
    subgraph "Producer"
        P[Send with partition key]
    end
    
    subgraph "Topic with 3 partitions"
        P0[Partition 0<br/>Orders: A, D, G]
        P1[Partition 1<br/>Orders: B, E, H]
        P2[Partition 2<br/>Orders: C, F, I]
    end
    
    subgraph "Consumers"
        C0[Consumer 0]
        C1[Consumer 1]
        C2[Consumer 2]
    end
    
    P -->|"hash(order_id)"| P0 & P1 & P2
    P0 --> C0
    P1 --> C1
    P2 --> C2
```

```go
// Use order ID as partition key
func publishOrderEvent(producer *Producer, event OrderEvent) error {
    return producer.Send(ctx, Message{
        Topic:        "orders",
        PartitionKey: event.OrderID, // Same order → same partition → ordered
        Body:         serialize(event),
    })
}
```

**Result:** All events for Order #123 go to same partition, processed in order.

### Handling Out-of-Order Messages

When ordering can't be guaranteed, design for it:

```go
type OrderEvent struct {
    OrderID   string
    Version   int       // Monotonically increasing
    Timestamp time.Time // For tie-breaking
    Type      string
    Data      interface{}
}

func processOrderEvent(event OrderEvent) error {
    // Get current version from database
    currentVersion, _ := db.GetOrderVersion(event.OrderID)
    
    if event.Version <= currentVersion {
        // Old event, skip (idempotent)
        log.Info("Skipping old event", "version", event.Version, "current", currentVersion)
        return nil
    }
    
    if event.Version > currentVersion+1 {
        // Gap detected, may need to wait or fetch missing events
        log.Warn("Gap in event versions", "expected", currentVersion+1, "got", event.Version)
        return ErrEventGap
    }
    
    // Process in order
    return applyEvent(event)
}
```

---

## 6. Event Sourcing

### What is Event Sourcing?

Instead of storing current state, store the sequence of events that led to current state.

```mermaid
flowchart LR
    subgraph "Traditional: Store State"
        STATE[(Order<br/>status: cancelled<br/>total: $100)]
    end
    
    subgraph "Event Sourcing: Store Events"
        E1[OrderCreated<br/>total: $100]
        E2[ItemAdded<br/>item: Widget]
        E3[OrderPaid<br/>amount: $100]
        E4[OrderCancelled<br/>reason: customer]
        E1 --> E2 --> E3 --> E4
    end
```

### Event Store Pattern

```go
// Events are immutable facts
type Event struct {
    ID            string
    AggregateID   string    // e.g., order_123
    AggregateType string    // e.g., "Order"
    Type          string    // e.g., "OrderCreated"
    Version       int       // Sequence number
    Timestamp     time.Time
    Data          interface{}
}

// Event store interface
type EventStore interface {
    Append(aggregateID string, events []Event, expectedVersion int) error
    Load(aggregateID string) ([]Event, error)
    LoadFrom(aggregateID string, fromVersion int) ([]Event, error)
}

// Rebuild state from events
func loadOrder(store EventStore, orderID string) (*Order, error) {
    events, err := store.Load(orderID)
    if err != nil {
        return nil, err
    }
    
    order := &Order{}
    for _, event := range events {
        order.Apply(event) // Replay each event
    }
    return order, nil
}

// Apply events to rebuild state
func (o *Order) Apply(event Event) {
    switch event.Type {
    case "OrderCreated":
        data := event.Data.(OrderCreatedData)
        o.ID = data.OrderID
        o.CustomerID = data.CustomerID
        o.Status = "pending"
        
    case "ItemAdded":
        data := event.Data.(ItemAddedData)
        o.Items = append(o.Items, data.Item)
        o.Total += data.Item.Price
        
    case "OrderPaid":
        o.Status = "paid"
        
    case "OrderCancelled":
        o.Status = "cancelled"
    }
    o.Version = event.Version
}
```

### CQRS (Command Query Responsibility Segregation)

Separate write model (events) from read model (projections).

```mermaid
flowchart TB
    subgraph "Write Side"
        CMD[Command] --> AGG[Aggregate]
        AGG --> ES[(Event Store)]
    end
    
    subgraph "Event Bus"
        ES --> BUS[Message Broker]
    end
    
    subgraph "Read Side"
        BUS --> PROJ[Projector]
        PROJ --> READ[(Read Database<br/>Optimized for queries)]
    end
    
    subgraph "Queries"
        Q[Query] --> READ
    end
```

```go
// Write side: Handle command, emit events
func (h *OrderHandler) CreateOrder(cmd CreateOrderCommand) error {
    order := NewOrder()
    events := order.Create(cmd.CustomerID, cmd.Items)
    
    return h.eventStore.Append(order.ID, events, 0)
}

// Read side: Project events to read model
func (p *OrderProjector) Handle(event Event) error {
    switch event.Type {
    case "OrderCreated":
        return p.db.Exec(`
            INSERT INTO orders_view (id, customer_id, status, created_at)
            VALUES ($1, $2, 'pending', $3)
        `, event.AggregateID, event.Data.CustomerID, event.Timestamp)
        
    case "OrderPaid":
        return p.db.Exec(`
            UPDATE orders_view SET status = 'paid', paid_at = $2 
            WHERE id = $1
        `, event.AggregateID, event.Timestamp)
    }
    return nil
}
```

### Event Sourcing Benefits & Drawbacks

| Benefits | Drawbacks |
|----------|-----------|
| Complete audit trail | More complex |
| Can rebuild any past state | Event schema evolution is hard |
| Easy temporal queries | Eventually consistent reads |
| Natural fit for event-driven | Storage grows forever |
| Debug by replaying events | Learning curve |

---

## 7. Dead Letter Queues

### What is a DLQ?

A queue for messages that can't be processed after multiple retries.

```mermaid
flowchart LR
    MAIN[(Main Queue)]
    CONSUMER[Consumer]
    DLQ[(Dead Letter Queue)]
    ALERT[Alert System]
    
    MAIN -->|Message| CONSUMER
    CONSUMER -->|Success| ACK[Acknowledged]
    CONSUMER -->|Fail 3x| DLQ
    DLQ --> ALERT
```

### DLQ Implementation

```go
type Message struct {
    ID          string
    Body        []byte
    RetryCount  int
    MaxRetries  int
    Errors      []string
}

func processWithDLQ(mainQ, dlq *Queue) {
    for {
        msg, _ := mainQ.Receive(ctx)
        
        err := process(msg)
        if err == nil {
            mainQ.Ack(ctx, msg)
            continue
        }
        
        msg.RetryCount++
        msg.Errors = append(msg.Errors, err.Error())
        
        if msg.RetryCount >= msg.MaxRetries {
            // Send to DLQ
            dlq.Send(ctx, msg)
            mainQ.Ack(ctx, msg) // Remove from main queue
            
            alertOps(msg) // Notify operations team
        } else {
            // Retry with backoff
            mainQ.Nack(ctx, msg, backoff(msg.RetryCount))
        }
    }
}
```

### DLQ Best Practices

| Practice | Description |
|----------|-------------|
| Preserve context | Include original message, all error messages, timestamps |
| Set up alerts | Notify when DLQ depth increases |
| Retention policy | How long to keep failed messages |
| Replay mechanism | Tool to retry DLQ messages after fixing bugs |
| Separate DLQs | Different DLQs for different failure types |

---

## 8. Backpressure

### What is Backpressure?

When consumers can't keep up with producers, what happens?

```mermaid
flowchart LR
    P[Producer<br/>1000 msg/sec]
    Q[(Queue)]
    C[Consumer<br/>100 msg/sec]
    
    P -->|1000/sec| Q
    Q -->|100/sec| C
    
    Note[Queue grows forever!<br/>Memory exhaustion<br/>Increased latency]
```

### Backpressure Strategies

#### 1. Drop Messages

```go
func (q *BoundedQueue) Send(msg Message) error {
    select {
    case q.channel <- msg:
        return nil
    default:
        metrics.Increment("messages.dropped")
        return ErrQueueFull // Caller decides what to do
    }
}
```

**Use when:** Data is time-sensitive, old data is useless (real-time metrics)

#### 2. Block Producer

```go
func (q *BlockingQueue) Send(ctx context.Context, msg Message) error {
    select {
    case q.channel <- msg:
        return nil
    case <-ctx.Done():
        return ctx.Err() // Timeout or cancellation
    }
}
```

**Use when:** Every message matters, can tolerate producer slowdown

#### 3. Buffer and Batch

```go
type BatchBuffer struct {
    buffer    []Message
    maxSize   int
    flushTime time.Duration
}

func (b *BatchBuffer) Add(msg Message) {
    b.mu.Lock()
    b.buffer = append(b.buffer, msg)
    
    if len(b.buffer) >= b.maxSize {
        b.flush()
    }
    b.mu.Unlock()
}

func (b *BatchBuffer) flushPeriodically() {
    ticker := time.NewTicker(b.flushTime)
    for range ticker.C {
        b.mu.Lock()
        if len(b.buffer) > 0 {
            b.flush()
        }
        b.mu.Unlock()
    }
}
```

**Use when:** Can batch process, reduces per-message overhead

#### 4. Scale Consumers

```go
func autoScaleConsumers(queue *Queue, pool *WorkerPool) {
    ticker := time.NewTicker(10 * time.Second)
    
    for range ticker.C {
        depth := queue.Depth()
        rate := queue.ProcessingRate()
        
        // Time to drain queue at current rate
        drainTime := time.Duration(depth/rate) * time.Second
        
        if drainTime > 5*time.Minute && pool.Size() < maxWorkers {
            pool.Scale(pool.Size() + 1)
        } else if drainTime < 30*time.Second && pool.Size() > minWorkers {
            pool.Scale(pool.Size() - 1)
        }
    }
}
```

**Use when:** Can add resources, messages must all be processed

---

## 9. Real-World Architectures

### Apache Kafka Architecture

```mermaid
flowchart TB
    subgraph "Producers"
        P1[Producer 1]
        P2[Producer 2]
    end
    
    subgraph "Kafka Cluster"
        subgraph "Topic: orders (3 partitions)"
            PA0[Partition 0<br/>Leader: B1<br/>Replicas: B2, B3]
            PA1[Partition 1<br/>Leader: B2<br/>Replicas: B1, B3]
            PA2[Partition 2<br/>Leader: B3<br/>Replicas: B1, B2]
        end
        
        B1[Broker 1]
        B2[Broker 2]
        B3[Broker 3]
        
        ZK[ZooKeeper/KRaft<br/>Metadata]
    end
    
    subgraph "Consumer Group"
        C1[Consumer 1<br/>Partitions: 0, 1]
        C2[Consumer 2<br/>Partition: 2]
    end
    
    P1 & P2 --> PA0 & PA1 & PA2
    PA0 --> C1
    PA1 --> C1
    PA2 --> C2
```

**Kafka characteristics:**
- Distributed commit log
- High throughput (millions msg/sec)
- Configurable retention (replay old messages)
- Consumer groups for parallel processing
- Strong ordering per partition

### RabbitMQ Architecture

```mermaid
flowchart TB
    subgraph "Publishers"
        P1[Publisher 1]
        P2[Publisher 2]
    end
    
    subgraph "RabbitMQ"
        EX[Exchange<br/>type: topic]
        Q1[(Queue: emails<br/>binding: order.*)]
        Q2[(Queue: analytics<br/>binding: #)]
        Q3[(Queue: inventory<br/>binding: order.created)]
    end
    
    subgraph "Consumers"
        C1[Email Service]
        C2[Analytics]
        C3[Inventory]
    end
    
    P1 -->|order.created| EX
    P2 -->|order.cancelled| EX
    EX -->|order.*| Q1
    EX -->|#| Q2
    EX -->|order.created| Q3
    Q1 --> C1
    Q2 --> C2
    Q3 --> C3
```

**RabbitMQ characteristics:**
- Traditional message broker
- Flexible routing (exchanges, bindings)
- Message acknowledgments
- Lower throughput than Kafka but simpler
- Messages deleted after consumption

### Comparison

| Feature | Kafka | RabbitMQ | SQS |
|---------|-------|----------|-----|
| Model | Log-based | Queue-based | Queue-based |
| Throughput | Very high | Medium | Low-Medium |
| Ordering | Per partition | Per queue | FIFO queues only |
| Replay | Yes (retention) | No | No |
| Routing | Consumer-side | Broker-side | None |
| Operations | Complex | Medium | Managed |
| Best for | Event streaming, high volume | Task queues, RPC | Simple queuing, AWS |

---

## 10. Quick Reference

### Message Design Checklist

```
□ Include idempotency key / message ID
□ Include timestamp
□ Include version for ordering
□ Include correlation ID for tracing
□ Schema versioning strategy
□ Appropriate serialization (JSON, Protobuf, Avro)
□ Size limits considered
```

### Choosing a Messaging Pattern

```
Need to distribute work?           → Work Queue
Need multiple services notified?   → Pub/Sub (Fan-out)
Need request/response?             → Request-Reply or RPC
Need event history / replay?       → Event Sourcing + Kafka
Need simple AWS integration?       → SQS + SNS
```

### Numbers to Remember

| Metric | Value |
|--------|-------|
| Kafka throughput (per broker) | 100K-200K msg/sec |
| RabbitMQ throughput | 20K-50K msg/sec |
| SQS throughput (standard) | 3K msg/sec |
| SQS FIFO throughput | 300 msg/sec |
| Typical message size | 1KB - 1MB |
| Kafka max message | 1MB (configurable) |
| SQS max message | 256KB |

---

## Summary

1. **Queues vs Pub/Sub:** Queues for work distribution (one consumer), Pub/Sub for broadcasting (all subscribers)

2. **Delivery guarantees:** At-most-once (fast, lossy), at-least-once (reliable, duplicates), exactly-once (at-least-once + idempotency)

3. **Ordering:** Use partition keys for ordering related messages. Design for out-of-order when you can't guarantee.

4. **Event sourcing:** Store events, not state. Enables replay, audit, temporal queries.

5. **Backpressure:** Handle it explicitly—drop, block, buffer, or scale.

**The golden rule:** Design consumers to be idempotent. Assume messages will be delivered multiple times.

**Next module:** [Storage](../../storage/concepts/concepts.md) - Block, object, and file storage at scale.