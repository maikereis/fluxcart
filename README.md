# Flux Cart

Adaptive e-commerce platform for tech markets, built on a microservices architecture designed for flash-sale-grade scalability, real-time inventory sync, and procurement-grade catalog search.

---

## Architecture Overview

The system decomposes the e-commerce domain into six independent bounded contexts, each owned by a dedicated team with full autonomy over its data model and deployment lifecycle.

| Service | Responsibility | Primary Store |
|---|---|---|
| User Management | Registration, auth, OAuth/SSO | PostgreSQL + Redis |
| Product Catalog | Dynamic catalog, pricing engine, search | MongoDB + Elasticsearch + Redis |
| Shopping Cart | Cart lifecycle, pricing snapshots, reservations | Redis + PostgreSQL |
| Order & Payment | Order orchestration, payment gateway, fraud detection | PostgreSQL |
| Recommendation Engine | ML personalization, collaborative filtering, A/B testing | MongoDB + Redis |
| Notification Service | Email, push, SMS delivery, event-driven templates | PostgreSQL |

---

## Diagrams

<image src="diagrams/e-commerce-level1.svg" width="800" height="600"></image>

<image src="diagrams/e-commerce-level2.svg" width="800" height="600"></image>

---

## Key Architectural Decisions

### ADR-001 — Apache Kafka as Asynchronous Event Bus

All cross-domain state propagation runs through Kafka (managed via Confluent Cloud). Producers publish to domain-named topics; consumers subscribe independently through dedicated consumer groups. This eliminates synchronous coupling between services during high-demand events such as flash sales.

Key topics: `order-events`, `cart-events`, `catalog-updates`, `user-events`, `payment-events`.

Event schemas are versioned via Confluent Schema Registry with `BACKWARD` compatibility. All topics retain messages for a minimum of 7 days. Failed messages are routed to a dead-letter topic (`{topic}.dlq`).

### ADR-002 — Polyglot Persistence

Each microservice owns its exclusive database. No service may directly access another service's store; integration occurs exclusively via REST or Kafka events. Database technology is selected per domain access pattern:

- **PostgreSQL** — transactional consistency for Orders, Users, and Notifications
- **Redis** — sub-millisecond cart access and session token storage
- **MongoDB** — flexible document schema for Catalog and Recommendations
- **Elasticsearch** — full-text and faceted product search

CAP trade-offs are applied deliberately: Orders and Users operate in **CP** mode; Catalog, Cart, Search, and Recommendations operate in **AP** mode.

### ADR-003 — API Gateway (Kong) as Unified Entry Point

Kong Gateway is the sole external entry point for both mobile (iOS/Android) and web (React SPA) clients. No internal microservice is exposed to the public network.

The gateway centralizes: JWT validation, rate limiting (100 req/min anonymous / 1,000 req/min authenticated), path-based routing with `/api/v1/` versioning, TLS termination, and observability via Prometheus + Jaeger.

---

## Communication Patterns

**Synchronous (REST):** client-facing and latency-sensitive service-to-service calls. All traffic enters through the API Gateway; internal communication uses HTTP with circuit breakers.

**Asynchronous (Kafka):** cross-domain event propagation for eventual consistency. Producers define canonical event schemas; consumers process at their own pace via independent consumer groups.



## Architecture Decision Records

Full decision rationale, alternatives considered, and consequences are documented in the `/docs/adr/` directory:

- [ADR-001 — Apache Kafka as Asynchronous Event Bus](docs/adr/ADR-001-kafka-event-bus.md)
- [ADR-002 — Polyglot Persistence by Domain](docs/adr/ADR-002-persistencia-poliglota.md)
- [ADR-003 — API Gateway as Unified Entry Point](docs/adr/ADR-003-api-gateway.md)