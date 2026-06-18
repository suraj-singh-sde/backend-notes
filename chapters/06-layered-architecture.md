# Chapter 6 — Controllers, Services, Repositories, Middlewares & Request Context

> The standard way to organize a backend so it stays maintainable as it grows. Each layer has one job and knows as little as possible about the others.

---

## 1. The Backend Architecture Pattern

When a client sends a request, splitting your server code into **Controllers, Services, and Repositories** isn't strictly mandatory — but it's a vital design pattern. This **separation of concerns** makes the codebase scalable, maintainable, easier to debug, and easier to extend.

```
        Request
           │
   ┌───────▼────────┐
   │   Controller   │  HTTP in/out: parse, validate, status codes
   └───────┬────────┘
   ┌───────▼────────┐
   │    Service     │  business logic (knows NOTHING about HTTP)
   └───────┬────────┘
   ┌───────▼────────┐
   │   Repository   │  database queries only
   └───────┬────────┘
        Database
```

---

## 2. The Controller (Handler) Layer

The controller is the **entry point** for a route once routing matches the URL. Its job is to manage data flow from the client to the server and back.

**Step-by-step workflow of a controller:**

1. **Data extraction:** receive the `request`/`response` objects from the runtime and pull out what you need (query params for a `GET`, JSON body for a `POST`).
2. **Binding (deserialization):** the body arrives as a JSON string; the controller deserializes it into a native type (Go struct, Python dict, JS object). If this fails → `400 Bad Request`.
3. **Validation:** confirm the data matches the expected format and reject malicious payloads (see [Chapter 5](05-validations-transformations.md)).
4. **Transformation:** reshape data for convenience — e.g., inject a default `sort` value if the client omitted it.
5. **Delegation:** hand the clean, validated data to the **Service** layer.
6. **Sending the response:** once the service returns, choose the right status code (`200`, `201`, `400`, `500`, …) and send the final response.

**Example — a thin controller (Express-style pseudocode):**

```js
async function createBook(req, res) {
  const body = req.body;                       // 1. extract  (already deserialized by middleware)
  const errors = validateBook(body);           // 3. validate
  if (errors) return res.status(400).json(errors);

  body.title = body.title.trim();              // 4. transform
  const book = await bookService.create(body); // 5. delegate to the service
  return res.status(201).json(book);           // 6. respond
}
```

Notice what the controller does *not* do: no SQL, no business rules. It's deliberately thin.

---

## 3. The Service Layer

This is where the actual **business logic** lives.

- **Complete HTTP isolation:** the service should know *nothing* about HTTP — no request/response objects, no status codes, no validation. It's a plain function: data in, result out.
- **Orchestration:** one service method can do a lot — call several repositories, merge datasets, send emails, hit external APIs.

```js
// service: pure logic, fully testable without any HTTP
async function create(bookData) {
  if (await bookRepo.existsByTitle(bookData.title)) {
    throw new ConflictError("Book already exists");   // throws a domain error, NOT an HTTP code
  }
  const book = await bookRepo.insert(bookData);
  await emailService.notifyLibrarians(book);
  return book;
}
```

Because the service has no HTTP dependencies, you can unit-test it without spinning up a web server.

---

## 4. The Repository (Database) Layer

The repository's sole job is **database operations**.

- **Workflow:** take data from the service, build the specific query (insert, filter, sort), and return the raw results.
- **Single Responsibility:** each method does exactly one thing and returns one shape of data. Don't write one method that returns *either* a single book *or* an array depending on a flag — write two methods.

```js
// Good: two focused methods
bookRepo.findById(id)      // → one book or null
bookRepo.findAll(filter)   // → array of books

// Avoid: one method that branches on a flag to return different shapes
bookRepo.find(id, returnArray)  // ✗ ambiguous return type
```

---

## 5. Middlewares

Middlewares run **in the middle** of the request lifecycle — between receiving the request and reaching the final controller.

- **Why use them?** to avoid duplicating cross-cutting logic (security, logging, parsing) across hundreds of endpoints. Middlewares centralize it.
- **The `next()` function:** middlewares receive `request`, `response`, and a `next` function. Calling `next()` passes control to the next middleware/handler. A middleware can choose *not* to call `next()` and instead respond immediately, terminating the request early.
- **Execution order matters:** the request flows through middlewares sequentially, so their order is part of the design.

**Common middlewares (in typical order):**

```
Request
  → CORS & Security Headers   reject unauthorized origins early → (block or next())
  → Rate Limiting             too many requests from this IP? → 429, else next()
  → Authentication            verify token; invalid → 401, else attach user & next()
  → Logging                   record method/path/params, then next()
  → [ Controller ]            the actual handler runs
  → Global Error Handler      (at the very END) catches anything thrown above,
                              formats a clean error response
```

- **CORS & security headers:** placed early; block disallowed origins instantly.
- **Rate limiting:** check the IP; if spamming → `429 Too Many Requests`.
- **Authentication:** verify the token; invalid → `401`; valid → extract user data and pass it downstream.
- **Logging:** record request details for debugging.
- **Global error handling:** placed at the **very end**; catches unstructured errors from controllers/services and formats them into clean, standard error responses.

---

## 6. Request Context

Request Context is a **shared, per-request key-value store**, scoped to a single HTTP request.

- **Purpose:** the request passes through many isolated function boundaries (several middlewares, then the controller). Context lets them share state *without* tightly coupling the code.

**Common use cases:**

1. **Passing authentication data:** the auth middleware validates the token, extracts `user_id` and `role`, and stores them in the context. Later, the controller pulls `user_id` from the context when saving a record.
   - *Security note:* **never** trust a `user_id` sent in the client's JSON payload — a malicious user could spoof another person's ID. Always take it from the context, where it was derived from the verified token.
2. **Request tracing:** an early middleware generates a unique ID (UUID) and stores it in the context. As the request flows through services and logs, that ID is attached everywhere, letting engineers trace one request's exact path across systems.
3. **Cancellations:** store abort signals and timeout deadlines so external calls don't hang forever.

```
Auth MW:   ctx.set("userId", "123")   ← derived from the VERIFIED token, not the body
              │
Controller:  const id = ctx.get("userId")   ← trusted; use this to save the record
              │                                 (ignore any userId the client sent)
Logger:      ctx.get("traceId")        ← same UUID attached to every log line for this request
```

---

← Previous: [« Chapter 5 — Validations & Transformations](05-validations-transformations.md) · Back to the [index](../README.md) · Next: [Chapter 7 — REST API Design »](07-rest-api-design.md)
