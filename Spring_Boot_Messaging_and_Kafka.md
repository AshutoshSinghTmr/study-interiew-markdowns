# Spring Boot Messaging and Kafka

## Why Messaging

Asynchronous messaging decouples producers from consumers, smooths load spikes, and enables event-driven architectures. Spring Boot integrates with Apache Kafka (log-based streaming) and RabbitMQ/JMS (broker-based queues) through auto-configured templates and listener containers.

## Spring for Apache Kafka

### Producing

`KafkaTemplate` sends records; Boot auto-configures it from `spring.kafka.*`. Sends are asynchronous and return a future.

```java
@Service
public class OrderEventPublisher {
    private final KafkaTemplate<String, OrderEvent> template;

    public OrderEventPublisher(KafkaTemplate<String, OrderEvent> template) {
        this.template = template;
    }

    public void publish(OrderEvent event) {
        template.send("orders", event.orderId(), event); // key = orderId preserves per-order ordering
    }
}
```

### Consuming

`@KafkaListener` methods run inside a listener container that polls the broker.

```java
@Component
public class OrderEventConsumer {

    @KafkaListener(topics = "orders", groupId = "billing")
    public void onOrder(OrderEvent event) {
        billing.charge(event);
    }
}
```

### Consumer groups, partitions, and offsets

* A **consumer group** shares a topic's partitions; each partition is consumed by exactly one member, so parallelism is capped by the partition count.
* The **key** decides the partition, which preserves ordering per key.
* **Offsets** track progress per partition. Prefer container-managed acknowledgment (`spring.kafka.listener.ack-mode`) over auto-commit for reliable at-least-once processing.

## Serialization

Configure key/value serializers and deserializers (`spring.kafka.producer.value-serializer`, `...consumer.value-deserializer`). JSON (`JsonSerializer`/`JsonDeserializer`) is common; add trusted packages for deserialization. For evolving schemas, use Avro/Protobuf with a schema registry.

## Error Handling and Retries

The listener container's `DefaultErrorHandler` retries with a back-off and then routes failures to a **dead-letter topic** (DLT) via `DeadLetterPublishingRecoverer`, so poison messages don't block a partition.

```java
@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template); // -> <topic>.DLT
    return new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3));
}
```

`@RetryableTopic` offers the same non-blocking retry/DLT flow declaratively.

## Delivery Semantics and Idempotency

* Kafka defaults to **at-least-once**, so consumers must be **idempotent** (dedupe on a message id or use upserts).
* The **idempotent producer** (`enable.idempotence=true`, default in modern clients) avoids duplicate writes on retries.
* **Transactions** (`spring.kafka.producer.transaction-id-prefix`) give exactly-once *within* a Kafka read-process-write flow, coordinated with the listener container.

## RabbitMQ / Spring AMQP

For broker-based queuing, Spring AMQP provides `RabbitTemplate` (send) and `@RabbitListener` (receive). Messages flow **producer → exchange → (binding) → queue → consumer**; the exchange type (direct, topic, fanout, headers) and routing key decide delivery. Acknowledgment modes and dead-letter exchanges mirror the reliability concerns above.

```java
@RabbitListener(queues = "orders.q")
public void handle(OrderEvent event) {
    billing.charge(event);
}
```

## Transactional Outbox

To publish an event atomically with a database change, write the event to an **outbox** table in the same local transaction, then a relay (polling or change-data-capture) publishes it to the broker. This avoids the dual-write problem where a DB commit succeeds but the broker send fails (or vice versa).

## Spring Cloud Stream

Spring Cloud Stream adds a binder abstraction over Kafka/RabbitMQ using functional `Supplier`/`Function`/`Consumer` beans, so the same code targets different brokers by swapping the binder — useful for portable event-driven microservices.

## Producer Internals

`KafkaTemplate` wraps a `KafkaProducer`. Durability and throughput are tuned with: `acks` (`0`/`1`/`all`), `retries` + `delivery.timeout.ms`, and `enable.idempotence=true` (dedupes retries and preserves per-partition order when `max.in.flight.requests<=5`). `linger.ms` + `batch.size` trade a little latency for much higher throughput; `compression.type` reduces bandwidth. The default **sticky partitioner** batches to one partition until it fills; a record **key** routes by hash to keep same-key messages ordered.

## Consumer Internals

Consumers run a **poll loop**. `max.poll.records` bounds each batch, and processing must finish within `max.poll.interval.ms` or the broker considers the consumer dead and triggers a rebalance. Heartbeats (`session.timeout.ms`) run separately from polling. Prefer the **cooperative-sticky** rebalance protocol (incremental) over the old eager (stop-the-world) one; use `group.instance.id` for **static membership** to avoid rebalances on rolling restarts.

## Offset Management

Offsets track progress per partition. Avoid `enable.auto.commit` for reliability; instead let the container commit via an `AckMode` (`RECORD`, `BATCH`, `MANUAL`, `MANUAL_IMMEDIATE`). Committing **after** processing gives at-least-once (possible duplicates); committing before gives at-most-once (possible loss). At-least-once plus idempotent consumers is the common, robust choice.

## Error Handling and Retries

The container's `DefaultErrorHandler` retries with a `FixedBackOff`/`ExponentialBackOff`, then hands failures to a `DeadLetterPublishingRecoverer` (-> `<topic>.DLT`). Prefer **non-blocking** retries via `@RetryableTopic` (separate retry topics + DLT), because in-line blocking retries stall the whole partition.

## Exactly-Once Semantics

Kafka transactions (`transactional.id`, `KafkaTransactionManager`) plus `sendOffsetsToTransaction` make a **read-process-write** flow atomic; downstream consumers set `isolation.level=read_committed`. This yields exactly-once **within Kafka**; when a side effect touches an external system, keep the consumer idempotent or use the outbox pattern.

## Serialization and Schema Evolution

Configure key/value `Serializer`/`Deserializer`. Wrap deserializers in `ErrorHandlingDeserializer` so a single poison record does not crash the container. For `JsonDeserializer`, set trusted packages/type mappings. For evolving contracts, use **Avro/Protobuf with a Schema Registry** and a compatibility mode (backward/forward) so producers and consumers can deploy independently.

## Concurrency and Containers

`@KafkaListener(concurrency = "N")` runs N consumer threads in a `ConcurrentMessageListenerContainer`, effective up to the partition count. **Batch listeners** process a list per poll for higher throughput; containers support `pause()`/`resume()` for backpressure and flow control.

## Kafka Streams, RabbitMQ, and the Outbox

**Kafka Streams** (`KStream`/`KTable`) does stateful stream processing with its own exactly-once support. **RabbitMQ** routes producer -> exchange (direct/topic/fanout/headers) -> queue, with ack/nack/requeue, dead-letter exchanges, and consumer `prefetch` (QoS). The **transactional outbox** (often with CDC via Debezium) publishes events reliably by writing them in the same DB transaction as the state change.

## Interview Q&A

**Q: How does Kafka scale consumers, and what limits parallelism?**
A: Consumers in a group split a topic's partitions, one consumer per partition at a time. Maximum parallelism equals the partition count; adding consumers beyond that leaves some idle.

**Q: How do you guarantee ordering for related messages?**
A: Give related records the same key so they land on the same partition; Kafka preserves order within a partition (not across partitions).

**Q: What delivery guarantee does Kafka give by default and what does that imply?**
A: At-least-once, so duplicates are possible on retries. Consumers must be idempotent, or you use transactions for exactly-once read-process-write.

**Q: How do you stop a bad ("poison") message from blocking a partition?**
A: Configure a `DefaultErrorHandler` with back-off and a `DeadLetterPublishingRecoverer` (or `@RetryableTopic`) to retry and then route the record to a dead-letter topic.

**Q: Why is committing offsets automatically risky?**
A: Auto-commit can advance the offset before processing succeeds, losing messages on failure. Use container ack-modes so the offset commits only after successful handling.

**Q: What problem does the transactional outbox solve?**
A: The dual-write problem — it makes the database change and the "event to publish" atomic in one local transaction, with a separate relay delivering the event reliably.

**Q: When would you pick RabbitMQ over Kafka?**
A: RabbitMQ suits per-message routing, work queues, and complex routing topologies with lower retention needs; Kafka suits high-throughput, replayable event streams with long retention and partition-based ordering.

**Q: What does `acks=all` plus the idempotent producer guarantee?**
A: `acks=all` waits for all in-sync replicas, avoiding data loss on broker failure; `enable.idempotence=true` prevents duplicate writes on producer retries and preserves per-partition order.

**Q: What is a consumer rebalance and how do you minimize disruption?**
A: When group membership or partitions change, partitions are reassigned. Use the cooperative-sticky assignor (incremental) and `group.instance.id` static membership so rolling restarts do not trigger full stop-the-world rebalances.

**Q: How do you achieve exactly-once processing in Kafka?**
A: Use Kafka transactions (`transactional.id`) with `sendOffsetsToTransaction` for a read-process-write flow, and set consumers to `isolation.level=read_committed`; keep external side effects idempotent.

**Q: How do you handle deserialization errors without crashing the consumer?**
A: Wrap the deserializer in `ErrorHandlingDeserializer` so a bad record becomes a handled failure (routed to retry/DLT) instead of an infinite poison-pill loop.

**Q: What does a schema registry give you?**
A: A central store of Avro/Protobuf schemas with compatibility rules (backward/forward) so producers and consumers can evolve message formats independently without breaking each other.

## Interview Notes

* Explain consumer groups, partitions, keys, and offset management.
* Know at-least-once delivery, idempotent consumers, DLT-based error handling, and exactly-once via transactions.
* Contrast Kafka vs RabbitMQ, and describe the transactional outbox pattern.
