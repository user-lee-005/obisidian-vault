# Message Queues

> *"The best way to handle something you can't do right now is to write it down and come back to it later."* — That's exactly what a message queue does for your systems.

---

## Phase 1: The Fundamentals

### What is a Message Queue?

A message queue is a **communication mechanism** where one system (producer) sends a message to a **buffer** (the queue), and another system (consumer) picks it up and processes it **later** — without the two systems ever talking to each other directly.

**Analogy:** Think of a **parcel sorting hub** in logistics. The sender drops off a parcel at the hub (producer → queue). The hub holds it until a delivery truck picks it up (consumer). The sender doesn't wait for the delivery — they walk away immediately. The hub guarantees the parcel won't get lost.

```
Synchronous (without queue):
  Client → Service A → Service B → Service C → Response
  (Client waits for entire chain to complete)

Asynchronous (with queue):
  Client → Service A → Response (immediate)
                ↓
             [Queue]
                ↓
           Service B → Service C (happens later)
```

### The Core Problem It Solves

In a synchronous system, if Service B is slow or down, Service A **also becomes slow or fails**. They are **tightly coupled**.

```
❌ Synchronous — tight coupling:
  Booking Service → Notification Service (down!) → ❌ Booking fails!

✅ Asynchronous — decoupled:
  Booking Service → [Message Queue] → Notification Service (can retry when it's back up)
  (Booking succeeds immediately)
```

### Why Do We Need Message Queues?

| Problem | How Queues Help |
|---|---|
| **Tight Coupling** | Producer and consumer are independent — they don't know about each other |
| **Handling Traffic Spikes** | Queue absorbs bursts; consumers process at their own pace |
| **Reliability** | Messages persist in the queue even if the consumer crashes |
| **Scalability** | Add more consumers to process messages faster |
| **Cross-Service Communication** | Different services/languages/teams can communicate via messages |

**Example:** A freight booking system receives 10,000 bookings during morning peak but the rate calculation engine can only process 1,000/min. Without a queue, 9,000 bookings would timeout. With a queue, all 10,000 bookings are accepted immediately and rate calculations happen at the engine's pace.

---

## Phase 2: Core Concepts

### The Basic Components

```
┌──────────┐     ┌─────────────────┐     ┌──────────┐
│ Producer  │────→│  Message Queue  │────→│ Consumer │
│ (Sender)  │     │  (Broker/Bus)   │     │(Receiver)│
└──────────┘     └─────────────────┘     └──────────┘
```

| Component | Role | Example |
|---|---|---|
| **Producer** | Creates and sends messages to the queue | Booking Service sends "new booking" event |
| **Message** | The data payload being transferred | `{ "bookingId": "BK-001", "status": "CREATED" }` |
| **Queue / Topic** | The buffer that holds messages until consumed | `booking-events` topic in Kafka |
| **Broker** | The server that manages queues and routes messages | Kafka broker, RabbitMQ server |
| **Consumer** | Reads and processes messages from the queue | Rate Calculation Service, Notification Service |

### Message vs Event

These terms are often used interchangeably, but there's a subtle difference:

| | Message | Event |
|---|---|---|
| **Intent** | "Please do this" (command) | "This happened" (notification) |
| **Direction** | Targeted to a specific consumer | Broadcast to anyone interested |
| **Example** | "Calculate rate for booking BK-001" | "Booking BK-001 was created" |
| **Coupling** | Producer knows what it wants done | Producer doesn't care who listens |

### Topics and Queues

**Queue (Point-to-Point):**
- A message is consumed by **exactly one consumer**
- Once consumed, the message is removed
- Like a task list — each task is done by one worker

```
Producer → [Queue] → Consumer A picks message 1
                   → Consumer B picks message 2
                   → Consumer A picks message 3
(Messages are distributed, not duplicated)
```

**Topic (Publish-Subscribe):**
- A message is delivered to **all subscribers**
- Each subscriber gets its own copy
- Like a broadcast — everyone listening hears it

```
Producer → [Topic] → Consumer A gets ALL messages
                   → Consumer B gets ALL messages
                   → Consumer C gets ALL messages
(Messages are duplicated to each subscriber)
```

---

## Phase 3: Messaging Patterns

### Pattern 1: Point-to-Point (Work Queue)

One producer, one queue, multiple consumers — but each message goes to **only one consumer**.

```
                    ┌── Consumer 1
Producer → [Queue] ─┼── Consumer 2
                    └── Consumer 3
(Load balanced — each message processed once)
```

**Use case:** Task distribution. 1000 shipment labels need printing → distribute across 5 printer services.

**Example:**
```
Queue: label-print-jobs
Message: { "shipmentId": "SHP-001", "labelType": "SHIPPING" }

Consumer 1 picks SHP-001
Consumer 2 picks SHP-002
Consumer 3 picks SHP-003
(Each label printed exactly once)
```

---

### Pattern 2: Publish-Subscribe (Pub/Sub)

One producer publishes to a topic, **multiple consumers each get every message**.

```
                      ┌── Notification Service (sends email)
Producer → [Topic] ───┼── Analytics Service (updates dashboard)
                      └── Audit Service (logs event)
```

**Use case:** Event broadcasting. When a shipment status changes, multiple services need to react.

**Example:**
```
Topic: shipment-status-updates
Event: { "shipmentId": "SHP-001", "status": "DELIVERED", "timestamp": "..." }

→ Notification Service: sends "Your shipment is delivered" email
→ Analytics Service: increments delivered count on dashboard  
→ Billing Service: triggers final invoice generation
→ Audit Service: logs the delivery event
```

---

### Pattern 3: Fan-Out

One message triggers **multiple independent downstream actions** — each handled by a separate queue/consumer.

```
                   ┌── [Queue A] → Email Service
Producer → [Topic] ┼── [Queue B] → SMS Service
                   └── [Queue C] → Push Notification Service
```

**Use case:** A single "booking confirmed" event needs to trigger email, SMS, and push notification — each independently.

---

### Pattern 4: Fan-In (Aggregation)

**Multiple producers** send messages into a **single queue** for centralized processing.

```
Booking Service ──┐
Tracking Service ──┼──→ [Audit Queue] → Audit Consumer
Billing Service ──┘
```

**Use case:** All services across the platform send audit logs to a centralized audit queue for compliance.

---

### Pattern 5: Request-Reply

Producer sends a message and **waits for a response** on a separate reply queue. This is async but with a response.

```
Producer → [Request Queue] → Consumer
                                ↓
Producer ← [Reply Queue]   ← Consumer
```

**Use case:** Async API call where you need a result but can't do it synchronously (e.g., requesting a freight rate quote that takes 10 seconds to compute).

---

## Phase 4: Popular Message Queue Systems

### Overview Comparison

| Feature | Apache Kafka | RabbitMQ | Amazon SQS | ActiveMQ |
|---|---|---|---|---|
| **Model** | Distributed log | Traditional broker | Managed queue | Traditional broker |
| **Throughput** | Very high (millions/sec) | Moderate (10k-50k/sec) | Moderate | Moderate |
| **Message Retention** | Configurable (days/weeks) | Until consumed | 14 days max | Until consumed |
| **Ordering** | Per partition | Per queue | Best-effort (FIFO available) | Per queue |
| **Replay** | ✅ Yes (offset-based) | ❌ No (consumed = gone) | ❌ No | ❌ No |
| **Protocol** | Custom TCP | AMQP, MQTT, STOMP | HTTP/REST | JMS, AMQP |
| **Best for** | Event streaming, logs, high volume | Task queues, complex routing | Serverless, AWS-native | Legacy Java/JMS apps |
| **Managed options** | Confluent Cloud, AWS MSK | CloudAMQP, Amazon MQ | Fully managed (AWS) | Amazon MQ |

### When to Choose What

```
Need high throughput + event replay?           → Kafka
Need complex routing + message acknowledgment? → RabbitMQ
Need simple managed queue on AWS?              → SQS
Need JMS compatibility for legacy Java?        → ActiveMQ
```

---

## Phase 5: Apache Kafka — Deep Dive

Kafka is the most widely used distributed event streaming platform. Built by LinkedIn, now open-source under Apache.

### Architecture

```
┌────────────────────────────────────────────────────┐
│                  Kafka Cluster                      │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │         │
│  │          │  │          │  │          │         │
│  │ Topic A  │  │ Topic A  │  │ Topic A  │         │
│  │ Part 0   │  │ Part 1   │  │ Part 2   │         │
│  │          │  │          │  │          │         │
│  │ Topic B  │  │ Topic B  │  │ Topic B  │         │
│  │ Part 0   │  │ Part 1   │  │ Part 2   │         │
│  └──────────┘  └──────────┘  └──────────┘         │
│                                                     │
│              ZooKeeper / KRaft                       │
│         (Cluster metadata management)               │
└────────────────────────────────────────────────────┘
```

### Core Concepts

#### Topics

A **topic** is a named category/feed of messages. Think of it as a **table in a database** — but for events.

```
Topic: shipment-events
Topic: booking-events
Topic: payment-events
```

Producers write to topics. Consumers read from topics.

#### Partitions

Each topic is split into **partitions** — ordered, append-only sequences of messages.

```
Topic: shipment-events (3 partitions)

Partition 0: [msg0] [msg1] [msg2] [msg3] [msg4] →
Partition 1: [msg0] [msg1] [msg2] →
Partition 2: [msg0] [msg1] [msg2] [msg3] →
```

**Why partitions?**
- **Parallelism** — multiple consumers can read different partitions simultaneously
- **Scalability** — distribute data across brokers
- **Ordering** — messages within a single partition are strictly ordered

**Partition Key:** Determines which partition a message goes to.

```java
// All events for the same shipment go to the same partition
// This guarantees ordering per shipment
producer.send(new ProducerRecord<>("shipment-events", shipmentId, event));
//                                   topic              key        value
```

Messages with the **same key always go to the same partition** → guaranteeing order for that key.

#### Offsets

Each message in a partition has a unique, sequential **offset** (position number).

```
Partition 0:
  Offset 0: { shipmentId: "SHP-001", status: "CREATED" }
  Offset 1: { shipmentId: "SHP-003", status: "CREATED" }
  Offset 2: { shipmentId: "SHP-001", status: "PICKED_UP" }
  Offset 3: { shipmentId: "SHP-003", status: "PICKED_UP" }
```

Consumers **track their offset** — they know exactly where they left off. This enables:
- **Replay** — reset offset to 0 to reprocess all messages
- **Resume** — consumer crashes, restarts, picks up from last committed offset
- **Parallel processing** — different consumer groups maintain independent offsets

#### Consumer Groups

A **consumer group** is a set of consumers that work together to consume a topic. Each partition is assigned to **exactly one consumer** within a group.

```
Topic: shipment-events (4 partitions)

Consumer Group A (3 consumers):
  Consumer A1 ← Partition 0, Partition 1
  Consumer A2 ← Partition 2
  Consumer A3 ← Partition 3

Consumer Group B (2 consumers):    ← Independent group, gets ALL messages
  Consumer B1 ← Partition 0, Partition 1
  Consumer B2 ← Partition 2, Partition 3
```

**Key rules:**
- Within a group: each partition goes to ONE consumer (no duplicates)
- Across groups: each group gets ALL messages (independent processing)
- More consumers than partitions = some consumers sit idle

**Example:**
```
Consumer Group: "tracking-service"  → Updates tracking dashboard
Consumer Group: "notification-svc"  → Sends status update emails
Consumer Group: "analytics-service" → Updates business intelligence

All three groups independently consume the same shipment-events topic.
```

#### Replication

Each partition has **replicas** across multiple brokers for fault tolerance.

```
Partition 0:
  Broker 1: Leader  (reads/writes happen here)
  Broker 2: Follower (replica, takes over if leader fails)
  Broker 3: Follower (replica)
```

**Replication Factor = 3** means each partition exists on 3 brokers. If one broker dies, no data is lost.

#### Retention

Unlike traditional queues, Kafka **retains messages even after consumption**.

```yaml
# Keep messages for 7 days
log.retention.hours=168

# OR keep up to 1TB per partition
log.retention.bytes=1073741824

# Compacted topic: keeps latest value per key forever
log.cleanup.policy=compact
```

This enables **replay** — a new service can join and process all historical events.

---

### Kafka Producer Configuration

```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka1:9092,kafka2:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

// Durability settings
props.put("acks", "all");              // Wait for all replicas to acknowledge
props.put("retries", 3);              // Retry on failure
props.put("enable.idempotence", true); // Prevent duplicate messages on retry

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.send(new ProducerRecord<>("shipment-events", shipmentId, eventJson));
```

**acks explained:**

| Setting | Behavior | Durability | Latency |
|---|---|---|---|
| `acks=0` | Don't wait for any acknowledgment | Lowest (may lose messages) | Fastest |
| `acks=1` | Wait for leader to write | Moderate (lose if leader crashes before replication) | Moderate |
| `acks=all` | Wait for all in-sync replicas | Highest (no data loss) | Slowest |

---

### Kafka Consumer Configuration

```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka1:9092,kafka2:9092");
props.put("group.id", "tracking-service");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

// Offset management
props.put("enable.auto.commit", false);       // Manual offset commit for reliability
props.put("auto.offset.reset", "earliest");   // Start from beginning if no offset exists

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("shipment-events"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processEvent(record.value());     // Process the message
    }
    consumer.commitSync();                // Commit offset AFTER successful processing
}
```

**auto.offset.reset explained:**

| Setting | Behavior |
|---|---|
| `earliest` | Start from the oldest available message (process everything) |
| `latest` | Start from new messages only (skip historical) |
| `none` | Throw error if no offset found |

---

## Phase 6: RabbitMQ — Deep Dive

RabbitMQ is a traditional message broker that implements the **AMQP** (Advanced Message Queuing Protocol). It focuses on **message routing, delivery guarantees, and complex routing patterns**.

### Architecture

```
┌────────────────────────────────────────────┐
│            RabbitMQ Broker                  │
│                                             │
│  Producer → [Exchange] → Binding → [Queue] → Consumer
│                                             │
│  Exchanges route messages to queues         │
│  based on routing rules (bindings)          │
└────────────────────────────────────────────┘
```

### The Routing Model: Exchange → Binding → Queue

Unlike Kafka where producers write directly to topics, RabbitMQ has an **exchange** layer that decides WHERE messages go.

```
Producer → Exchange → (routing logic) → Queue 1 → Consumer A
                                      → Queue 2 → Consumer B
                                      → Queue 3 → Consumer C
```

### Exchange Types

#### 1. Direct Exchange

Routes messages to queues based on an **exact routing key match**.

```
Producer sends: routing_key = "shipment.created"

Exchange (Direct):
  Binding: "shipment.created" → Queue A  ✅ (match)
  Binding: "shipment.delivered" → Queue B ❌ (no match)
```

**Use case:** Task-specific routing — different queues for different event types.

#### 2. Fanout Exchange

Broadcasts messages to **ALL bound queues** regardless of routing key.

```
Producer sends: (any routing key)

Exchange (Fanout):
  → Queue A (Email Service)     ✅
  → Queue B (SMS Service)       ✅
  → Queue C (Analytics Service) ✅
```

**Use case:** Broadcasting events to all interested services.

#### 3. Topic Exchange

Routes based on **pattern matching** on the routing key using wildcards.

```
Routing key format: <entity>.<event>.<region>

Producer sends: routing_key = "shipment.delivered.asia"

Exchange (Topic):
  Binding: "shipment.*.asia"    → Queue A  ✅ (matches)
  Binding: "shipment.delivered.*" → Queue B  ✅ (matches)
  Binding: "booking.#"          → Queue C  ❌ (no match)

Wildcards:
  * = matches exactly one word
  # = matches zero or more words
```

**Use case:** Flexible routing — regional teams subscribe to their own events.

#### 4. Headers Exchange

Routes based on **message headers** instead of routing key.

```
Producer sends: headers = { "region": "asia", "type": "express" }

Exchange (Headers):
  Binding: region=asia         → Queue A  ✅
  Binding: type=express        → Queue B  ✅
  Binding: region=europe       → Queue C  ❌
```

**Use case:** Complex routing based on multiple attributes.

---

### Message Acknowledgment

RabbitMQ removes a message from the queue **only after the consumer acknowledges it**.

```java
// Manual acknowledgment
channel.basicConsume(queueName, false, (tag, delivery) -> {  // autoAck = false
    try {
        processMessage(delivery.getBody());
        channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);  // Success
    } catch (Exception e) {
        channel.basicNack(delivery.getEnvelope().getDeliveryTag(), false, true);  // Requeue
    }
});
```

| Action | Effect |
|---|---|
| `basicAck` | Message processed successfully → removed from queue |
| `basicNack` (requeue=true) | Processing failed → put back in queue for retry |
| `basicNack` (requeue=false) | Processing failed → discard or send to Dead Letter Queue |
| `basicReject` | Like nack but for a single message |

---

### RabbitMQ Prefetch (QoS)

Controls how many unacknowledged messages a consumer can hold at once.

```java
channel.basicQos(10);  // Consumer gets max 10 messages at a time
```

**Why it matters:** Without prefetch, one fast consumer grabs all messages while slow consumers sit idle. Prefetch ensures fair distribution.

---

## Phase 7: Kafka vs RabbitMQ — When to Use Which

| Aspect | Kafka | RabbitMQ |
|---|---|---|
| **Mental model** | Distributed commit log | Smart message router |
| **Message retention** | Keeps messages after consumption | Deletes after acknowledgment |
| **Replay** | ✅ Reset offset, reprocess everything | ❌ Once consumed, gone |
| **Ordering** | Per partition (strong) | Per queue (strong) |
| **Throughput** | Millions/sec | Tens of thousands/sec |
| **Routing** | Simple (topic + partition key) | Complex (exchanges, bindings, patterns) |
| **Consumer model** | Pull (consumers poll) | Push (broker delivers) |
| **Latency** | Higher (batching) | Lower (per-message delivery) |
| **Backpressure** | Natural (consumers control pace) | Prefetch-based |
| **Best for** | Event streaming, analytics, log aggregation | Task queues, complex routing, RPC |

### Decision Framework

```
"I need to stream millions of events and replay them"
  → Kafka

"I need to route messages based on complex rules"
  → RabbitMQ

"I need multiple services to independently process the same events"
  → Kafka (consumer groups)

"I need reliable task execution with retries"
  → RabbitMQ

"I need to feed real-time dashboards and analytics"
  → Kafka

"I need request-reply pattern"
  → RabbitMQ
```

**Logistics example:**
- **Shipment event streaming** (created, picked up, in transit, delivered) → **Kafka** — high volume, multiple consumers, replay needed
- **Send notification email after booking** → **RabbitMQ** — simple task, needs reliable delivery, complex routing by notification type

---

## Phase 8: Delivery Guarantees

The most critical concept in messaging — how many times is a message delivered?

### At-Most-Once Delivery

Message is delivered **zero or one time**. May be lost, but never duplicated.

```
Producer → [Queue] → Consumer
                      ↓
              Process message
              (No acknowledgment needed)
              If consumer crashes → message lost
```

**How:** Consumer auto-acknowledges before processing, or producer uses `acks=0`.
**Risk:** Message loss.
**Use case:** Metrics, logs — losing a few data points is acceptable.

---

### At-Least-Once Delivery

Message is delivered **one or more times**. Never lost, but may be duplicated.

```
Producer → [Queue] → Consumer
                      ↓
              Process message
              Acknowledge ← fails/timeout
              ↓
              Broker re-delivers the same message
              Consumer processes it AGAIN (duplicate!)
```

**How:** Consumer acknowledges AFTER processing. If ack fails, broker redelivers.
**Risk:** Duplicate processing.
**Use case:** Most business events — duplicates can be handled with idempotency.

**This is the most common delivery guarantee used in production.**

---

### Exactly-Once Delivery

Message is delivered **exactly one time**. No loss, no duplicates.

```
Producer → [Queue] → Consumer
                      ↓
              Process message (transactionally)
              Acknowledge (atomically with processing)
```

**How:** Kafka supports this via **idempotent producers + transactions + exactly-once consumers**.
**Complexity:** Very high — requires transactional processing.
**Use case:** Financial transactions, billing — duplicates cause real money problems.

**In practice:** True exactly-once is extremely hard. Most systems use **at-least-once + idempotency** instead.

---

### Comparison

| Guarantee | Message Loss | Duplicates | Complexity | Use Case |
|---|---|---|---|---|
| At-most-once | Possible | No | Low | Metrics, logs |
| At-least-once | No | Possible | Medium | Most business events |
| Exactly-once | No | No | Very high | Financial, billing |

---

## Phase 9: Idempotency — Handling Duplicates

Since **at-least-once** is the most common guarantee, your consumers **will** receive duplicates. You must handle them.

### What is Idempotency?

An operation is **idempotent** if performing it multiple times produces the **same result** as performing it once.

```
Idempotent:     SET shipment.status = "DELIVERED"   (same result every time)
NOT idempotent: INCREMENT shipment.weight BY 5      (adds 5 every time!)
```

### Strategies for Idempotent Consumers

#### Strategy 1: Idempotency Key (Message ID)

Store processed message IDs. Skip messages you've already seen.

```java
public void handleShipmentEvent(ShipmentEvent event) {
    // Check if already processed
    if (processedEventRepository.existsById(event.getEventId())) {
        log.info("Duplicate event {}, skipping", event.getEventId());
        return;
    }
    
    // Process the event
    shipmentService.updateStatus(event.getShipmentId(), event.getStatus());
    
    // Mark as processed
    processedEventRepository.save(new ProcessedEvent(event.getEventId()));
}
```

**Important:** The check + process + mark should be in a **single database transaction** to avoid race conditions.

#### Strategy 2: Database Constraints

Use unique constraints to naturally prevent duplicates.

```sql
-- Trying to insert a duplicate booking confirmation fails naturally
INSERT INTO booking_confirmations (booking_id, confirmed_at)
VALUES ('BK-001', NOW())
ON DUPLICATE KEY UPDATE confirmed_at = confirmed_at;  -- No-op on duplicate
```

#### Strategy 3: Conditional Updates (Optimistic Locking)

Only update if the current state matches expectations.

```sql
-- Only update to DELIVERED if currently IN_TRANSIT
UPDATE shipments 
SET status = 'DELIVERED', version = version + 1
WHERE shipment_id = 'SHP-001' 
  AND status = 'IN_TRANSIT'
  AND version = 5;

-- If 0 rows affected → either already delivered or wrong version → skip
```

---

## Phase 10: Message Ordering

### Why Ordering Matters

For a shipment, events must be processed in order:
```
CREATED → PICKED_UP → IN_TRANSIT → DELIVERED

If processed out of order:
  DELIVERED processed before IN_TRANSIT → Customer sees "Delivered" then "In Transit"!
```

### How Kafka Guarantees Ordering

Kafka guarantees ordering **within a single partition only**.

```
Partition key = shipmentId

All events for SHP-001 go to Partition 2 (same partition):
  Offset 0: SHP-001 CREATED
  Offset 1: SHP-001 PICKED_UP
  Offset 2: SHP-001 IN_TRANSIT
  Offset 3: SHP-001 DELIVERED
→ Consumed in order ✅

Events for SHP-002 go to Partition 0 (different partition):
  Offset 0: SHP-002 CREATED
  Offset 1: SHP-002 PICKED_UP
→ Also consumed in order ✅

But NO ordering guarantee between SHP-001 and SHP-002 (different partitions).
→ This is usually fine — you rarely need cross-entity ordering.
```

**Rule:** Use the **entity ID as partition key** to guarantee per-entity ordering.

### How RabbitMQ Guarantees Ordering

RabbitMQ guarantees ordering **per queue** — messages are delivered in FIFO order to a single consumer. But with multiple consumers on one queue, ordering is NOT guaranteed because consumers process at different speeds.

**Solution:** Use a **single consumer per queue** if strict ordering is needed (sacrifices throughput).

---

## Phase 11: Dead Letter Queues (DLQ)

### What is a Dead Letter Queue?

A DLQ is a **special queue** where messages go when they **can't be processed** after multiple attempts.

```
Normal flow:
  [Main Queue] → Consumer → Process ✅

Failure flow:
  [Main Queue] → Consumer → Process ❌ (fail)
                           → Retry 1 ❌
                           → Retry 2 ❌
                           → Retry 3 ❌
                           → [Dead Letter Queue] (parked for investigation)
```

### Why Not Just Retry Forever?

- **Poison messages** — some messages will NEVER succeed (bad data, bug in consumer)
- Infinite retry blocks the queue — other messages can't be processed
- DLQ lets you **isolate failures** without losing data

### Implementation in RabbitMQ

```java
// Declare DLQ
channel.queueDeclare("shipment-events-dlq", true, false, false, null);

// Main queue with DLQ configuration
Map<String, Object> args = new HashMap<>();
args.put("x-dead-letter-exchange", "");                      // Default exchange
args.put("x-dead-letter-routing-key", "shipment-events-dlq"); // Route to DLQ
args.put("x-message-ttl", 30000);                            // Message TTL: 30 sec
channel.queueDeclare("shipment-events", true, false, false, args);
```

### Implementation in Kafka

Kafka doesn't have built-in DLQ. You implement it manually:

```java
@KafkaListener(topics = "shipment-events")
public void consume(ConsumerRecord<String, String> record) {
    try {
        processEvent(record.value());
    } catch (Exception e) {
        int retryCount = getRetryCount(record);
        if (retryCount >= MAX_RETRIES) {
            // Send to DLQ topic
            kafkaTemplate.send("shipment-events-dlq", record.key(), record.value());
            log.error("Message sent to DLQ after {} retries: {}", retryCount, record.key());
        } else {
            // Send to retry topic with incremented count
            kafkaTemplate.send("shipment-events-retry", record.key(), record.value());
        }
    }
}
```

### Retry Topology Pattern (Kafka)

```
[Main Topic] → Consumer → Process
                  ↓ (on failure)
            [Retry Topic 1] (5 sec delay)
                  ↓ (still failing)
            [Retry Topic 2] (30 sec delay)
                  ↓ (still failing)
            [Retry Topic 3] (5 min delay)
                  ↓ (still failing)
            [DLQ Topic] (parked, alert ops team)
```

This is called **exponential backoff with DLQ**.

### Monitoring DLQ

DLQ is **not a set-and-forget solution**. You must:
- ✅ Alert when messages arrive in DLQ
- ✅ Dashboard showing DLQ depth over time
- ✅ Tooling to inspect, fix, and replay DLQ messages
- ✅ Root cause analysis for every DLQ message

---

## Phase 12: Backpressure and Flow Control

### What is Backpressure?

When a **consumer can't keep up** with the rate of incoming messages, the queue grows unboundedly → eventually runs out of memory/disk.

```
Producer: 10,000 msgs/sec
Consumer: 2,000 msgs/sec
Queue growth: 8,000 msgs/sec → Queue explodes 💥
```

### Backpressure Strategies

| Strategy | How It Works | Tradeoff |
|---|---|---|
| **Scale consumers** | Add more consumer instances | Higher infra cost |
| **Rate limit producers** | Slow down message production | Producer latency increases |
| **Queue size limits** | Reject/drop messages when queue is full | Message loss risk |
| **Consumer prefetch** (RabbitMQ) | Limit unacked messages per consumer | Controlled flow |
| **Consumer lag monitoring** (Kafka) | Alert when consumer falls behind | Reactive, not preventive |

### Consumer Lag in Kafka

**Consumer lag** = Latest offset in partition - Consumer's committed offset

```
Partition 0: latest offset = 1,000,000
Consumer's offset = 950,000
Lag = 50,000 messages

If lag keeps growing → consumer can't keep up → scale up or optimize
```

**Monitoring:**
```bash
# Check consumer lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group tracking-service --describe
```

---

## Phase 13: Event-Driven Architecture

Message queues are the backbone of **event-driven architecture** (EDA) — one of the most important patterns in modern distributed systems.

### Event-Driven vs Request-Driven

```
Request-Driven (synchronous):
  Booking Service → calls → Rate Service → calls → Notification Service
  (Chain of synchronous calls — tightly coupled)

Event-Driven (asynchronous):
  Booking Service → publishes "BookingCreated" event
    → Rate Service reacts (subscribes)
    → Notification Service reacts (subscribes)
    → Analytics Service reacts (subscribes)
  (Loosely coupled — services are independent)
```

### Event Sourcing

Instead of storing the **current state**, store the **sequence of events** that led to the current state.

```
Traditional (state-based):
  Shipment table: { id: "SHP-001", status: "DELIVERED", location: "Chennai" }

Event Sourcing (event-based):
  Event Store:
    1. { type: "ShipmentCreated", shipmentId: "SHP-001", origin: "Mumbai" }
    2. { type: "ShipmentPickedUp", shipmentId: "SHP-001", carrier: "MAERSK" }
    3. { type: "ShipmentInTransit", shipmentId: "SHP-001", location: "Arabian Sea" }
    4. { type: "ShipmentDelivered", shipmentId: "SHP-001", location: "Chennai" }
    
  Current state = replay all events → { status: "DELIVERED", location: "Chennai" }
```

**Benefits:**
- Complete audit trail — you know exactly what happened and when
- Time travel — reconstruct state at any point in time
- Event replay — fix bugs by replaying events through corrected logic
- Natural fit with Kafka (it's already an event log)

**Drawbacks:**
- Complexity — rebuilding state from events is non-trivial
- Storage — events accumulate over time
- Querying — can't easily query current state without projections

**Perfect for:** Logistics tracking, financial systems, compliance-heavy domains.

---

### CQRS — Command Query Responsibility Segregation

Separate the **write model** (commands) from the **read model** (queries).

```
                ┌── Command (Write) ──→ Event Store (Kafka)
User Request ───┤                              ↓
                └── Query (Read) ───→ Read DB (optimized for queries)
                                        ↑
                                   Projection (builds read model from events)
```

**Example:**
```
Write side: Booking Service receives "Create Booking" command
  → Validates → Stores BookingCreated event in Kafka

Read side: Dashboard queries booking counts
  → Reads from a denormalized read table (materialized from events)
  → Optimized for the exact query needed (no joins, pre-aggregated)
```

**Benefits:**
- Write and read models can be **independently optimized and scaled**
- Read model can be rebuilt from events at any time

**When to use:** When read and write patterns are fundamentally different (e.g., writes are events, reads are complex aggregations).

---

## Phase 14: Message Queues in Spring Boot

### Spring Kafka

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: tracking-service
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.logistics.events"
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
```

#### Producer

```java
@Service
@RequiredArgsConstructor
public class ShipmentEventProducer {

    private final KafkaTemplate<String, ShipmentEvent> kafkaTemplate;

    public void publishStatusChange(String shipmentId, String newStatus) {
        ShipmentEvent event = new ShipmentEvent(shipmentId, newStatus, Instant.now());
        
        kafkaTemplate.send("shipment-events", shipmentId, event)  // key = shipmentId
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish event for {}: {}", shipmentId, ex.getMessage());
                } else {
                    log.info("Published event for {} to partition {} offset {}", 
                        shipmentId,
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}
```

#### Consumer

```java
@Service
public class ShipmentEventConsumer {

    @KafkaListener(
        topics = "shipment-events",
        groupId = "tracking-service",
        concurrency = "3"  // 3 consumer threads (match partition count)
    )
    public void handleShipmentEvent(
            @Payload ShipmentEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment acknowledgment) {
        
        try {
            log.info("Processing event: shipment={}, status={}, partition={}, offset={}",
                event.getShipmentId(), event.getStatus(), partition, offset);
            
            trackingService.updateStatus(event.getShipmentId(), event.getStatus());
            
            acknowledgment.acknowledge();  // Manual commit after successful processing
            
        } catch (Exception e) {
            log.error("Failed to process event at offset {}: {}", offset, e.getMessage());
            // Don't acknowledge — message will be redelivered
            throw e;
        }
    }
}
```

#### Error Handling with Retry + DLQ

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> kafkaTemplate) {
        // Publish to DLQ after 3 retries with exponential backoff
        DeadLetterPublishingRecoverer recoverer = 
            new DeadLetterPublishingRecoverer(kafkaTemplate);
        
        BackOff backOff = new ExponentialBackOff(1000L, 2.0);  // 1s, 2s, 4s
        ((ExponentialBackOff) backOff).setMaxAttempts(3);
        
        return new DefaultErrorHandler(recoverer, backOff);
    }
}
```

---

### Spring AMQP (RabbitMQ)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
```

#### Configuration (Exchange + Queue + Binding)

```java
@Configuration
public class RabbitMQConfig {

    @Bean
    public TopicExchange shipmentExchange() {
        return new TopicExchange("shipment-exchange");
    }

    @Bean
    public Queue notificationQueue() {
        return QueueBuilder.durable("notification-queue")
            .withArgument("x-dead-letter-exchange", "")
            .withArgument("x-dead-letter-routing-key", "notification-dlq")
            .build();
    }

    @Bean
    public Queue notificationDLQ() {
        return QueueBuilder.durable("notification-dlq").build();
    }

    @Bean
    public Binding notificationBinding() {
        return BindingBuilder
            .bind(notificationQueue())
            .to(shipmentExchange())
            .with("shipment.*.delivered");  // Only delivered events
    }
}
```

#### Producer

```java
@Service
@RequiredArgsConstructor
public class ShipmentNotificationProducer {

    private final RabbitTemplate rabbitTemplate;

    public void notifyDelivered(String shipmentId) {
        ShipmentEvent event = new ShipmentEvent(shipmentId, "DELIVERED", Instant.now());
        
        rabbitTemplate.convertAndSend(
            "shipment-exchange",                    // Exchange
            "shipment." + shipmentId + ".delivered", // Routing key
            event
        );
    }
}
```

#### Consumer

```java
@Service
public class NotificationConsumer {

    @RabbitListener(queues = "notification-queue")
    public void handleDeliveryNotification(ShipmentEvent event) {
        log.info("Sending delivery notification for {}", event.getShipmentId());
        emailService.sendDeliveryConfirmation(event.getShipmentId());
    }
}
```

---

## Phase 15: When Message Queues Are an Overhead

Just like caching, message queues aren't always the answer.

### Scenario 1: Simple, Low-Volume Synchronous Calls

**Example:** An internal admin tool where 5 users create 20 bookings/day. Each booking just saves to DB and returns.

```
With queue:    Admin → API → [Queue] → Consumer → DB → (no immediate response!)
Without queue: Admin → API → DB → Response ✅

The queue adds latency, complexity, and infra for 20 requests/day.
```

**Verdict:** Direct synchronous call is simpler and perfectly fine.

---

### Scenario 2: Need Immediate Response with Result

**Example:** A rate calculation API where the client needs the freight rate in the HTTP response.

```
Client: POST /api/rates { origin: "INMAA", dest: "USLAX" }
Expected: { "rate": 2500.00, "currency": "USD" }

With queue: Client sends request → Queue → Consumer calculates → ???
            (How does the client get the response? Request-reply pattern 
             is complex and adds latency)
```

**Verdict:** If the caller needs an immediate result, a synchronous call is more natural.

---

### Scenario 3: Tightly Coupled Two-Service Chain

**Example:** Service A always calls Service B, and only Service B. No fan-out, no retry needed, no traffic spikes.

```
Service A → Service B (direct HTTP call, 5ms response, 99.99% uptime)

Adding a queue between them:
  + No real benefit (B is always available)
  - Added latency, infra, operational overhead
  - Debugging is harder (message tracing vs HTTP tracing)
```

**Verdict:** A direct HTTP call with a retry mechanism (Resilience4j) is simpler.

---

### Scenario 4: Small Monolith

**Example:** A single Spring Boot app handling bookings, tracking, and notifications — all in one deployable.

```
BookingService.java → NotificationService.java (method call in same JVM)

Adding Kafka between them:
  - Requires running a Kafka cluster
  - Serialization/deserialization overhead
  - Operational complexity (brokers, ZooKeeper, monitoring)
  - For what? In-memory method calls are faster and simpler.
```

**Verdict:** Use Spring's `ApplicationEventPublisher` for in-process async events instead:
```java
// Simple in-process event — no external queue needed
applicationEventPublisher.publishEvent(new BookingCreatedEvent(booking));

@EventListener
@Async
public void handleBookingCreated(BookingCreatedEvent event) {
    notificationService.sendConfirmation(event.getBooking());
}
```

---

### Scenario 5: Data That Must Be Processed in Strict Global Order

**Example:** A ledger system where every transaction must be processed in exact global sequence.

```
Queue with multiple partitions → ordering only per partition, not globally
Single partition → ordering guaranteed BUT no parallelism → bottleneck
```

**Verdict:** A traditional database with sequential transactions may be more appropriate than a message queue.

---

### The Message Queue Overhead Checklist

Before adding a message queue, ask:

- [ ] **Is the operation async?** → If the caller needs an immediate result, probably no
- [ ] **Is there fan-out?** → If only one consumer, maybe a direct call is simpler
- [ ] **Are there traffic spikes?** → If traffic is steady and low, buffering isn't needed
- [ ] **Do I need resilience to consumer downtime?** → If consumer is always up, queue adds latency
- [ ] **Is my system already distributed?** → If it's a monolith, use in-process events
- [ ] **Is the operational overhead justified?** → Running Kafka is non-trivial

> **Rule of thumb:** If you can draw the message flow as a simple straight line with one producer and one consumer, and the consumer is always available, you probably don't need a queue.

---

## Phase 16: Real-World Case Studies

### Case 1: Logistics — Shipment Event Pipeline

**Problem:** A freight platform processes 50,000 shipment events/hour from carriers (GPS, status updates, milestones). These events need to update tracking, trigger notifications, feed analytics, and log for compliance — all independently.

**Solution:**
```
Carrier Webhook → Ingestion Service → [Kafka: shipment-events]
                                            ↓
                               ┌── Tracking Service (consumer group 1)
                               ├── Notification Service (consumer group 2)
                               ├── Analytics Service (consumer group 3)
                               └── Compliance Logger (consumer group 4)
```

- **Partition key:** `shipmentId` (per-shipment ordering)
- **Retention:** 30 days (replay for debugging)
- **DLQ:** Separate DLQ topic per consumer group

**Result:** Each service processes events independently. If Analytics goes down for maintenance, events accumulate in Kafka and are processed when it comes back — no data loss.

---

### Case 2: Logistics — Booking Confirmation Workflow

**Problem:** When a booking is confirmed, the system must: generate documents, send customer email, notify carrier, create invoice, and update dashboard. Doing all synchronously takes 8 seconds.

**Solution:**
```
Booking Service confirms booking (200ms)
  → Publishes "BookingConfirmed" to RabbitMQ (fanout exchange)
  → Returns success to user immediately

Fanout exchange delivers to:
  [doc-gen-queue]      → Document Service generates BOL (3 sec)
  [email-queue]        → Email Service sends confirmation (1 sec)
  [carrier-notify-queue] → Carrier API notified (2 sec)
  [invoice-queue]      → Billing Service creates invoice (1 sec)
  [dashboard-queue]    → Dashboard updated (500ms)
```

**Result:** User gets response in 200ms instead of 8 seconds. All side effects happen asynchronously.

---

### Case 3: Logistics — Rate Card Update Propagation

**Problem:** When a new freight tariff is uploaded, 15 microservices that cache rate data need to invalidate their caches.

**Solution:**
```
Admin uploads new tariff
  → Rate Service publishes "TariffUpdated" to Kafka
  → All 15 services subscribe → evict rate caches → reload on next request
```

This connects directly to what we learned in [[Caching]] — **event-based cache invalidation using message queues**.

---

## Key Takeaways

1. **Message queues decouple producers from consumers** — they don't need to know about each other
2. **Choose the right tool** — Kafka for high-volume event streaming, RabbitMQ for complex routing and task queues
3. **Partitioning = parallelism + ordering** — use entity ID as partition key
4. **At-least-once + idempotency** is the practical sweet spot for most systems
5. **Dead Letter Queues are essential** — never let poison messages block your pipeline
6. **Consumer lag is your #1 metric** — if lag grows, you have a problem
7. **Event-driven architecture** (EDA) with Event Sourcing and CQRS are powerful patterns for logistics
8. **Don't queue everything** — synchronous calls are simpler when traffic is low and consumers are reliable
9. **Spring Boot has first-class support** — Spring Kafka and Spring AMQP make integration straightforward

---

**Next up:** [[Microservices Patterns]]
