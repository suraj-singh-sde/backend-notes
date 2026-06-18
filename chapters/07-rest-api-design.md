# Chapter 7 — Complete REST API Design

> The conventions that turn a pile of routes into a coherent, predictable API. This chapter ties together everything from the previous six.

---

## 1. What Is REST? (History & Definition)

- **The origin:** In the 1990s, Tim Berners-Lee invented the World Wide Web. As it grew exponentially, it hit a scalability crisis. In 2000, Roy Fielding proposed a standardized architectural style to address it: **REST (Representational State Transfer)**.
- **Breaking down the name:**
  - **Representational:** a resource can have multiple representations depending on who's asking — JSON for an API client, HTML for a browser.
  - **State:** the current condition/attributes of a resource (e.g., the items and total price in a shopping cart right now).
  - **Transfer:** moving those representations between client and server using standard HTTP methods.

---

## 2. The 6 Constraints of REST Architecture

To be truly "RESTful" and scalable, a system follows Fielding's six constraints:

1. **Client-Server:** strict separation of concerns — client handles UI/UX, server handles data and logic.
2. **Uniform Interface:** a standardized, consistent way for all components to communicate.
3. **Layered System:** hierarchical layers (load balancers, proxies); each layer only sees the one directly below it, improving security and scaling.
4. **Cache:** responses must label themselves cacheable or not, reducing server load and improving speed.
5. **Stateless:** the server keeps no memory of past requests; each request carries everything needed to process it.
6. **Code on Demand (optional):** servers may extend client functionality by sending executable code (e.g., JavaScript).

---

## 3. Anatomy of a RESTful Route

Follow standard, hierarchical naming conventions:

- **The structure:** a typical URL looks like `https://api.example.com/v1/books` — a secure scheme (`https`), an API subdomain (`api.`), a version (`v1`), and the resource path (`books`).
- **Always use plural nouns:** the resource segment should be plural (`/books`, `/organizations`), even when fetching one item by ID (`/books/123`).
- **Formatting:** never use spaces or underscores. Convert readable slugs to lowercase with hyphens (e.g., `/books/harry-potter`).
- **Hierarchical nesting:** `/` denotes hierarchy. `/organizations/123/projects` means "the projects belonging to organization 123."

```
https://api.example.com/v1/organizations/123/projects
└─┬─┘   └──────┬──────┘ └┬┘ └─────┬──────┘ └┬┘ └──┬───┘
scheme    api subdomain  ver   resource     id   sub-resource
                                (plural)          (plural)
```

---

## 4. HTTP Methods and Idempotency

**Idempotency:** an operation is idempotent if performing it many times has the same effect on server state as performing it once.

- **`GET` (idempotent):** fetch data; calling it a thousand times changes nothing.
- **`PATCH` / `PUT` (idempotent):** update data; re-applying the same payload yields the same final state.
  - **`PATCH`** — update *partial* fields (e.g., just a user's status).
  - **`PUT`** — *completely replace* the resource representation.
- **`DELETE` (idempotent):** the first call removes the resource; later calls return `404`, but cause no further state change.
- **`POST` (non-idempotent):** create resources. Ten identical `POST`s create ten distinct resources with ten IDs.

> Idempotency is why clients can safely **retry** `GET`/`PUT`/`DELETE` after a network blip, but must be careful retrying `POST` (which is where idempotency keys and deduplication come in).

---

## 5. Custom Actions (the Exception to CRUD)

Some actions don't fit neatly into Create/Read/Update/Delete — e.g., "cloning" a project or "archiving" an organization, which trigger background work beyond a simple update.

- **The rule:** when an action doesn't map to a standard method, REST conventions say use **`POST`**.
- **Route design:** append the action as a verb at the end of a specific resource route.

```
POST /projects/123/clone
POST /organizations/5/archive
```

---

## 6. Designing Robust "List" APIs (GET)

A list endpoint must handle heavy data loads via three features:

1. **Pagination:** sending thousands of records at once crashes clients and clogs networks. Send data in pages. A paginated response should include:
   - `data` — the array of objects for this page.
   - `total` — total count in the database (useful for UI).
   - `page` — the current page number.
   - `totalPages` — the number of pages available.
2. **Sorting:** clients use query params like `sortBy=name` and `sortOrder=ascending`.
3. **Filtering:** clients pass properties to filter (e.g., `?status=active&name=org`).

**Example — a paginated list response:**

```
GET /v1/books?status=active&sortBy=title&sortOrder=asc&page=2&limit=20

200 OK
{
  "data":       [ { "id": 21, "title": "Dune" }, ... 20 items ... ],
  "total":      137,
  "page":       2,
  "totalPages": 7
}
```

---

## 7. Handling HTTP Status Codes Properly

- `200 OK` — standard success for fetching, updating, or custom actions.
- `201 Created` — a `POST` successfully created an entity.
- `204 No Content` — a successful `DELETE` with an intentionally empty body.
- **The 404 rule:**
  - Return `404` only when a client requests a *specific, single* resource ID that doesn't exist (e.g., `GET /users/999`).
  - For a *list* request that finds no matches (e.g., all users named "Zack"), do **not** return `404`. Return `200 OK` with an empty array `[]`.

```
GET /users/999          → 404 Not Found        (a specific resource that doesn't exist)
GET /users?name=Zack    → 200 OK  { "data": [] }  (a valid search with zero results)
```

---

## 8. Golden Rules & Best Practices for API Engineers

- **Extract nouns from the UI.** Before coding, study the frontend wireframes (Figma) to see what data users touch. The "nouns" (projects, users, tasks) become your core resources.
- **Implement sane defaults.** Don't crash when a client omits optional data. No pagination limit? Default to `10`. No sort order? Default to `created_at` descending. No status on a new org? Default to `active`.
- **Be ruthlessly consistent.** Use `camelCase` everywhere in JSON. If one endpoint calls a field `description`, never abbreviate it to `desc` elsewhere. Inconsistency forces other developers to guess and breeds bugs.
- **Provide interactive documentation.** Use Swagger/OpenAPI to generate a live playground — it doubles as documentation for frontend developers and a testing ground for you.

---

← Previous: [« Chapter 6 — Layered Architecture](06-layered-architecture.md) · Back to the [index](../README.md) · Next: [Chapter 8 — Databases »](08-databases.md)
