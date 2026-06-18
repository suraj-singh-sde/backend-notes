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

## How to read these

- **New to backend?** Read in order 1 → 7; each chapter assumes the ones before it.
- **Reference use?** Jump straight to a chapter from the table above.

> Chapters are numbered 1–7 for this file set. (The original combined notes carried over an earlier 5–11 numbering; that has been normalized here.)
