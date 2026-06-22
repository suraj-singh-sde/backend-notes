# Backend Engineering Notes

A working set of notes on backend fundamentals, organized one topic per chapter. Each chapter is self-contained but they build on each other in order.

## Chapters

| # | Chapter | What it covers |
|---|---------|----------------|
| 1 | [Understanding HTTP](chapters/01-http.md) | Statelessness, HTTP versions, message anatomy, headers, methods & idempotency, CORS, status codes, caching, content negotiation, large transfers, TLS/HTTPS |
| 2 | [Routing in Backend](chapters/02-routing.md) | Method + path mapping, static vs. dynamic routes, path vs. query params, nested routes, versioning, catch-all routes |
| 3 | [Serialization & Deserialization](chapters/03-serialization.md) | Why a shared wire format is needed, text vs. binary formats, JSON deep dive, the end-to-end round trip |
| 4 | [Authentication & Authorization](chapters/04-authentication-authorization.md) | Sessions, JWTs, cookies, stateful vs. stateless, API keys, OAuth/OIDC, RBAC, security best practices |
| 5 | [Validations & Transformations](chapters/05-validations-transformations.md) | Where validation runs, the validation pipeline, the four validation types, type casting & normalization, frontend vs. backend |
| 6 | [Controllers, Services, Repositories, Middlewares & Request Context](chapters/06-layered-architecture.md) | Layered architecture, the controller/service/repository split, middleware chains, request-scoped context |
| 7 | [Complete REST API Design](chapters/07-rest-api-design.md) | REST history & constraints, route anatomy, idempotency, custom actions, list APIs, status codes, golden rules |
| 8 | [Mastering Databases with Postgres](chapters/08-databases.md) | Persistence, SQL vs. NoSQL, Postgres types, migrations, data modeling, constraints, transactions & ACID, indexes, SQL injection, N+1, query design |
| 9 | [Caching](chapters/09-caching.md) | Hit ratio, latency ladder, network/hardware/software levels, cache topologies (L1/L2/near-cache), read & write strategies, eviction policies, invalidation & consistency, production pitfalls (stampede/avalanche/penetration/hot & big keys), operating a cache (HA, persistence, observability), how large companies cache, Redis use cases |

## How to read these

- **New to backend?** Read in order 1 → 9; each chapter assumes the ones before it.
- **Reference use?** Jump straight to a chapter from the table above.
