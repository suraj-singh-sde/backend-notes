# Chapter 2 — Routing in Backend

> How requests find their way home. If [Chapter 1](01-http.md) is about *how* a client talks to a server, routing is about *where* in the server a given request lands.

---

## 1. What is Routing?

- **The "What" vs. the "Where":** HTTP methods (`GET`, `POST`, `DELETE`, …) express the **what** — the intent of a request. Routing expresses the **where** — the specific resource or destination that intent applies to.
- **Definition:** Routing maps a combination of an **HTTP method + URL path** to a specific server-side **handler** (the function holding the business logic).
- **Uniqueness:** The server treats the *method + path* pair as a unique key. A `GET /api/books` and a `POST /api/books` share a path but trigger completely different handlers without clashing.

---

## 2. Types of Routes

There are two ways to structure a route path:

- **Static routes:** constant strings with no variable parts. `/api/books` always points to the same general resource.
- **Dynamic routes:** include variable slots the server extracts as data. Most frameworks denote these with a colon, e.g., `/api/users/:id`. A request to `/api/users/123` extracts `123` as the `id` and fetches that specific user.

**Example — a small routing table (method + path → handler):**

```
GET     /api/books          → listBooks()
POST    /api/books          → createBook()
GET     /api/books/:id      → getBook(id)
PATCH   /api/books/:id      → updateBook(id)
DELETE  /api/books/:id      → deleteBook(id)
```

`GET /api/books` and `POST /api/books` share a path but map to different handlers — the **method + path pair** is the unique key.

---

## 3. Path Parameters vs. Query Parameters

Two distinct ways to pass data through a URL:

- **Path parameters (route parameters):** variables placed directly inside the path, after a `/` (e.g., the `123` in `/api/users/123`). They carry **semantic meaning** — they identify *which specific resource* you want.
- **Query parameters:** key-value metadata appended after a `?`. Because `GET` requests have no body, query params are how a client passes options to a read request.
  - *Syntax:* `/api/search?query=some+value`, with multiple pairs joined by `&`.
  - *Use cases:* pagination (`page=2&limit=20`), filtering, and sort order.

**Example — same resource, different roles for the two kinds of parameter:**

```
GET /api/users/123/orders?status=shipped&sort=date&limit=20
              └─┬─┘        └──────────────┬───────────────┘
         path param:                  query params:
     WHICH user (identity)     HOW to filter/shape the result set
```

**Rule of thumb:** if removing the value changes *which resource* you're addressing, it's a path param. If it only *filters or shapes* the result, it's a query param.

---

## 4. Nested Routing

Nesting expresses a **hierarchy** between resources — a standard REST practice that makes URLs read like a sentence.

```
/api/users                    → list all users
/api/users/123                → one specific user
/api/users/123/posts          → all posts belonging to user 123
/api/users/123/posts/456      → one specific post (456) belonging to user 123
```

Each `/` step drills one level deeper into the data hierarchy.

---

## 5. Route Versioning and Deprecation

As an app grows, business needs change — you might need to change the shape of the data a route returns (e.g., renaming the key `name` to `title`).

- **The problem:** changing the response shape on a live route **breaks** every frontend already relying on it (iOS apps, React apps, third-party integrations).
- **The solution — versioning:** put a version in the path, e.g., `/api/v1/products` and `/api/v2/products`.
- **Deprecation:** the server can serve both versions at once. This gives frontend teams a safe window to migrate to `v2` before the backend team removes `v1`.

```
/api/v1/products  → { "name": "Pen" }     ← old clients keep working
/api/v2/products  → { "title": "Pen" }    ← new clients adopt the new shape
                                              v1 is deprecated, then removed later
```

---

## 6. Catch-All Routes

- **Purpose:** a safety net for requests that match no defined route.
- **How it works:** placed at the very *end* of the routing logic, often via a wildcard like `/*`. If a request falls through all earlier matches, it lands here.
- **Benefit:** instead of a broken or null response, the catch-all returns a clean, user-friendly `404 Route Not Found`.

> Order matters: a catch-all defined too early would swallow requests meant for real routes. It must always be last.

---

← Previous: [« Chapter 1 — Understanding HTTP](01-http.md) · Back to the [index](../README.md) · Next: [Chapter 3 — Serialization »](03-serialization.md)
