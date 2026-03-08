# ADR-003 — API Gateway as Unified Entry Point

| Field         | Value                                              |
|---------------|----------------------------------------------------|
| **ID**        | ADR-003                                            |
| **Title**     | API Gateway Pattern — Unified Integration for Mobile and Web Clients |
| **Status**    | Accepted                                           |
| **Date**      | 2026-03-08                                         |
| **Authors**   | Architecture Team — E-Commerce Platform            |
| **Reviewers** | Mobile Engineering, Frontend Engineering, Security |

---

## Context

The platform serves clients via a native mobile app (iOS/Android) and a web browser (React SPA). Without a unified entry point, each client would need to discover and directly address the internal microservices, creating five critical problems:

1. **Topological coupling:** the mobile app would need to know the internal addresses of each service. Any infrastructure refactoring would break clients.
2. **Duplicated authentication:** each microservice would validate JWTs independently, expanding the attack surface and creating policy inconsistencies.
3. **Fragmented CORS management:** each service would manage CORS headers separately for web clients.
4. **No centralized rate limiting:** without per-IP/client request limit enforcement, the platform would be vulnerable to DDoS and API abuse during flash sale peaks.
5. **Protocol mismatch:** mobile clients prefer REST/JSON; internal services may evolve to gRPC without impacting external contracts.

---

## Decision

A dedicated **API Gateway** (Kong Gateway) is deployed as the sole external entry point to the platform. No internal microservice is directly exposed to the public network.

The Gateway centralizes the following cross-cutting concerns:

- **Authentication and authorization:** JWT validation; identity propagated via `X-User-Id` and `X-User-Roles` headers to internal services.
- **Rate limiting:** per IP and per API key; differentiated limits by plan (anonymous, authenticated, admin).
- **Routing and versioning:** path-based routing with `/api/v1/` prefix; canary release support via traffic splitting plugin.
- **TLS termination:** HTTPS mandatory externally; internal cluster traffic via mTLS (Istio).
- **Observability:** Prometheus metrics on all inbound requests; distributed traces via Jaeger injected as `X-Trace-Id` header.

**Mobile endpoint mapping by domain:**

| Domain               | Exposed Endpoints                                                                  |
|----------------------|------------------------------------------------------------------------------------|
| Users MS             | `POST /api/v1/users` · `POST /api/v1/auth/login` · `POST /api/v1/auth/logout` · `PUT /api/v1/users/me` |
| Catalog MS           | `GET /api/v1/products/search` · `GET /api/v1/products/{id}` · `GET /api/v1/products/compare` |
| Cart MS              | `POST /api/v1/cart/items` · `DELETE /api/v1/cart/items/{id}` · `GET /api/v1/cart` · `POST /api/v1/cart/checkout` |
| Orders/Payments MS   | `POST /api/v1/orders` · `GET /api/v1/orders/{id}` · `GET /api/v1/orders` · `POST /api/v1/orders/{id}/cancel` |

---

## Alternatives Considered

### 1. Direct Client-to-Service Access (no gateway)

- **Pros:** Zero additional hop latency; simpler architecture initially.
- **Cons:** Clients coupled to internal topology; no centralized authentication; no rate limiting; per-service CORS; mobile app cannot aggregate multiple resources in a single call; impossibility of versioning APIs independently from the service.
- **Decision:** Rejected — not viable for production mobile apps with multiple simultaneous store releases.

### 2. BFF — Backend for Frontend (per platform)

- **Pros:** Payloads optimized per client type (mobile vs web); per-platform aggregation layer that reduces round-trips in the app.
- **Cons:** Additional service to maintain per platform; coordination overhead between teams for each new feature; duplication of authentication and rate limiting logic if not combined with a gateway.
- **Decision:** Considered as a **Phase 2** evolution — add a mobile BFF on top of the API Gateway for payload-specific optimizations, without replacing it.

### 3. API Gateway (Kong) *(chosen)*

- **Pros:** Centralized authentication, rate limiting, routing, and TLS termination; rich plugin ecosystem; decouples internal topology from clients; ingress observability; API versioning without breaking changes in the app.
- **Cons:** Single point of failure if not configured in HA; additional ~2ms latency per hop; team needs to learn Kong administration (plugins, upstreams, routes).
- **Decision:** Selected.

---

## Consequences

### Positive

- Mobile and web app interact with a stable API surface, independent of internal service topology refactoring.
- Centralized JWT validation at the Gateway; internal services trust the propagated identity headers without re-validation.
- Rate limiting uniformly enforced: 100 req/min for anonymous users; 1,000 req/min for authenticated users; protects backends during high-demand events.
- All inbound traffic telemetry captured at the Gateway: Prometheus metrics, Jaeger traces, structured logs centralized in ELK.

### Negative / Risks

- **Gateway as critical path:** deployed in HA mode with a minimum of 3 Kubernetes replicas; health checks with automatic failover in < 10s.
- **API version management:** versioning via path prefix (`/api/v1/`, `/api/v2/`) prevents breaking changes in mobile clients between releases — the Gateway routes versions in parallel during migration windows.
- **Kong operational learning curve:** mitigated with IaC (declarative Kong via `deck`) and documented runbooks for common operations (add route, configure rate limit, update certificate).

---

## Additional Notes

- Kong is configured via **deck** (declarative config) — all configuration versioned in the repository under `infra/kong/`.
- Globally enabled plugins: `jwt`, `rate-limiting`, `prometheus`, `request-transformer`, `correlation-id`.
- Per-route enabled plugins: `cors` (only for routes accessed by the web SPA), `request-size-limiting` (10MB limit for product image uploads).
- TLS certificate managed via **cert-manager** (Let's Encrypt) with automatic renewal.
- The Phase 2 evolution to a mobile BFF will be implemented as an additional Node.js service positioned between the Gateway and the microservices, without altering existing external contracts.
