# ADR-002 — Polyglot Persistence by Domain

| Field         | Value                                              |
|---------------|----------------------------------------------------|
| **ID**        | ADR-002                                            |
| **Title**     | Polyglot Persistence Strategy — Database per Service |
| **Status**    | Accepted                                           |
| **Date**      | 2026-03-08                                         |
| **Authors**   | Architecture Team — E-Commerce Platform            |
| **Reviewers** | Backend Engineering, Data Engineering, DBA Team    |

---

## Context

The platform manages radically different data models and access patterns across its domains. Enforcing a single shared relational database would create three critical problems:

1. **Deploy coupling:** a schema migration in the Catalog service could block a deployment of Orders.
2. **Forced compromise:** optimizations for transactional order queries conflict with the full-text search needs of Catalog and the high-speed access requirements of Cart.
3. **Violation of ownership principle:** distinct teams should not share the same schema — this creates implicit dependencies and hinders each team's autonomy.

Per-domain requirements:

| Domain               | Data Characteristics                                                      |
|----------------------|---------------------------------------------------------------------------|
| Orders / Payments    | Multi-table ACID transactions; immutable audit log                        |
| Users                | Structured relational data; ephemeral session tokens                      |
| Product Catalog      | Flexible schema (specs vary by category); full-text and faceted search    |
| Cart                 | Ephemeral per-session state; reads/writes with < 5ms latency              |
| Recommendations      | High-volume user-product interactions; batch ML training                  |
| Notifications        | Delivery audit trail; queries by status and recipient                     |

---

## Decision

Each microservice has its own exclusive database, selected based on the domain's data model and access pattern. **No other service may directly access another service's database.** Integration occurs exclusively via REST API or Kafka events.

**Definitive mapping:**

| Service              | Primary DB              | Secondary DB              | Rationale                                                                     |
|----------------------|-------------------------|---------------------------|-------------------------------------------------------------------------------|
| Orders/Payments MS   | PostgreSQL 15           | —                         | Full ACID; multi-table transactions; Flyway for versioned migrations           |
| Users MS             | PostgreSQL 15           | Redis 7                   | PostgreSQL for account data; Redis for JWT tokens and revocation list          |
| Catalog MS           | MongoDB 7               | Elasticsearch 8 + Redis   | Flexible documents per category; Elasticsearch for search; Redis for availability cache |
| Cart MS              | Redis 7                 | PostgreSQL 15             | Redis as primary store with TTL; PostgreSQL for abandoned cart recovery        |
| Recommendations MS   | MongoDB 7               | Redis 7                   | Interaction documents; Redis for serving pre-computed scores                  |
| Notifications MS     | PostgreSQL 15           | —                         | Delivery audit log; queries by status and recipient                            |

---

## Alternatives Considered

### 1. Shared PostgreSQL (single schema)

- **Pros:** Simple operation; easy cross-entity JOINs; single backup and monitoring strategy.
- **Cons:** Coupling between services via schema; impossibility of independent deployment; schema change locks; not optimized for full-text search or high-frequency cache access.
- **Decision:** Rejected — violates the service ownership principle and prevents independent scalability.

### 2. Single distributed NewSQL (CockroachDB / Spanner)

- **Pros:** Distributed SQL with global ACID; horizontal scaling; a single technology to operate.
- **Cons:** Full-text search still requires a separate Elasticsearch; catalog schema flexibility still limited; global write latency higher than Redis for the cart use case; high cost.
- **Decision:** Rejected — does not eliminate the need for specialized stores for search and cache.

### 3. Polyglot Persistence *(chosen)*

- **Pros:** Each service uses the optimal tool for its workload; independent schema evolution; clear data ownership; CAP trade-offs applied per domain (CP for orders, AP for catalog and cart).
- **Cons:** Larger operational surface; multiple backup strategies; eventual consistency requires careful event design.
- **Decision:** Selected.

---

## Consequences

### Positive

- Entirely independent schema deployment and evolution per service.
- Each service optimized for its workload: PostgreSQL for transactional consistency, Elasticsearch for search relevance, Redis for sub-millisecond latency in the cart.
- CAP trade-offs deliberately applied: Orders and Users in **CP** mode (consistency-priority); Catalog, Cart, and Recommendations in **AP** mode (availability-priority).
- Independent scalability: the Catalog's Elasticsearch can scale its shards without affecting the Orders PostgreSQL.

### Negative / Risks

- **Operational complexity:** monitoring, backup, and patching of multiple technologies. Mitigated with managed services (AWS RDS, ElasticCloud, Redis Cloud).
- **Eventual consistency across domains:** denormalized data across different stores requires synchronization via Kafka events. Transient inconsistencies are accepted where the domain allows (e.g., catalog slightly stale in cart for up to 60s).
- **Engineer onboarding:** learning curve for the team to work with multiple database technologies. Mitigated with runbook documentation and per-domain access pattern guides.

---

## Additional Notes

- PostgreSQL migrations managed via **Flyway**, with versioned scripts in each service's repository under `src/main/resources/db/migration/`.
- MongoDB migrations managed via **Mongock**.
- Automatic backups with minimum retention of **30 days** for PostgreSQL and MongoDB; **7 days** for Redis (ephemeral data).
- PostgreSQL connection pooling via **HikariCP**, configured with a maximum pool of 20 connections per service instance.
