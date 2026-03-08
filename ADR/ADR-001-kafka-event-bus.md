# ADR-001 — Apache Kafka as Asynchronous Event Bus

| Field         | Value                                              |
|---------------|----------------------------------------------------|
| **ID**        | ADR-001                                            |
| **Title**     | Use of Apache Kafka as Asynchronous Messaging Backbone |
| **Status**    | Accepted                                           |
| **Date**      | 2026-03-08                                         |
| **Authors**   | Architecture Team — E-Commerce Platform            |
| **Reviewers** | Backend Engineering, DevOps, Security              |

---

## Context

The e-commerce platform consists of six independent microservices — Users, Catalog, Cart, Orders/Payments, Recommendations, and Notifications — that need to propagate domain events to each other without creating direct coupling.

During high-demand events (flash sales, product launches), synchronous chains of REST calls between Cart, Order, Notification, and Recommendation create fragile dependencies: slowness in the Notifications service cannot block or bring down the checkout flow.

Requirements driving this decision:

- **At-least-once delivery** for critical business events (order confirmation, payment receipt).
- **Event replay**: new microservices must be able to subscribe to existing topics and reprocess history without impacting producers.
- **Independent consumer scalability**: the Notifications service needs to scale its processing capacity decoupled from the Cart.
- **Full decoupling**: producers must not know the consumers of their events.

---

## Decision

We will adopt **Apache Kafka** (managed via Confluent Cloud) as the central event bus for all asynchronous communication between microservices.

Each domain publishes events to domain-named topics. Each consumer service maintains its own _consumer group_, processing events at its own pace, with no coupling to other consumers on the same topic.

**Defined topics:**

| Topic              | Producer                  | Primary Consumers                    |
|--------------------|---------------------------|--------------------------------------|
| `order-events`     | Orders/Payments MS        | Notifications, Recommendations       |
| `cart-events`      | Cart MS                   | Recommendations, Orders MS           |
| `catalog-updates`  | Catalog MS                | Cart, Search, Recommendations        |
| `user-events`      | Users MS                  | Notifications, Recommendations       |
| `payment-events`   | Orders/Payments MS        | Notifications, Audit                 |

---

## Alternatives Considered

### 1. Synchronous Chained REST (point-to-point)

- **Pros:** No infrastructure overhead; easy to trace synchronously.
- **Cons:** Strong temporal coupling. A failure in the Notifications service propagates to the checkout flow. Producer must know all consumers. No replay.
- **Decision:** Rejected — unacceptable risk of cascading failure during peak load.

### 2. RabbitMQ (AMQP)

- **Pros:** Simpler operation; flexible routing via exchanges; native dead-letter queues; lower learning curve.
- **Cons:** No message retention by default — consumed messages are removed from the queue. Historical event replay is not natively supported. Lower throughput than Kafka in high-concurrency scenarios.
- **Decision:** Rejected — replay capability is a functional requirement for the Recommendations service onboarding.

### 3. Apache Kafka *(chosen)*

- **Pros:** Log-based with configurable retention; event replay; multiple independent consumer groups; high throughput; mature ecosystem (Spring Kafka, Schema Registry, Kafka Streams).
- **Cons:** Higher operational complexity (partitioning, replication, offsets); at-least-once semantics requires idempotent consumers; debugging asynchronous flows is more complex.
- **Decision:** Selected.

---

## Consequences

### Positive

- Full producer-consumer decoupling: new services subscribe to existing topics with no changes to producers.
- Resilience: events persisted in Kafka for 7 days, surviving transient consumer failures.
- Replay capability: the Recommendations service can rebuild its model by training on 90 days of historical `order-events`.
- Independent scalability: consumers scale their replicas based on consumer group lag (monitored via Prometheus).

### Negative / Risks

- **Mandatory idempotency:** all consumers must handle duplicate delivery; enforced via `idempotencyKey` in the event schema.
- **Operational complexity:** mitigated by adopting Confluent Cloud (managed Kafka) instead of a self-hosted cluster.
- **Asynchronous observability:** Kafka consumer lag monitored via Prometheus + Grafana; alerts configured for lag > 10,000 messages.

---

## Additional Notes

- Minimum retention of **7 days** on all production topics.
- Event schemas versioned via **Schema Registry** (Confluent) with `BACKWARD` compatibility.
- Each consumer team is responsible for its own _consumer group_ and offset management.
- In case of persistent failure, messages are routed to a **Dead Letter Topic** (`{topic}.dlq`) for manual reprocessing.
