# Chapter 1 — Understanding HTTP

> Where it all starts. Almost every backend you build speaks HTTP, so a solid mental model of the protocol pays off in every other chapter.

HTTP (Hypertext Transfer Protocol) is an **application-layer protocol** (Layer 7 in the OSI model) that clients and servers use to communicate. Everything else — routing, REST, auth, caching — is built on top of the ideas below.

---

## 1. Core Principles of HTTP

HTTP rests on two fundamental ideas:

- **Statelessness:** The server keeps no memory of past interactions. Every request is entirely self-contained and must carry all the information the server needs to process it (authentication tokens, cookies, etc.).
  - *Why it matters:* Statelessness simplifies server architecture and improves scalability. Any server in a fleet can handle any request because none of them rely on locally stored session state, and a crash doesn't destroy a client's progress.
- **Client-Server Model:** Communication is always initiated by the client (e.g., a browser). The server waits passively for requests and then responds.

**Statelessness in practice** — the server can't "remember" you between requests, so each request re-proves who you are:

```
Request 1:  GET /cart       Authorization: Bearer abc...   → server has no idea who you are
                                                              UNTIL it reads the token in THIS request
Request 2:  GET /orders     Authorization: Bearer abc...   → token sent again; server re-identifies you
```

Drop the token from request 2 and the server treats you as a stranger — it retained nothing from request 1.

---

## 2. Transport Protocol & HTTP Versions

HTTP needs a reliable, connection-based transport underneath it — almost universally **TCP (Transmission Control Protocol)**. Over the years, HTTP evolved mainly to use those TCP connections more efficiently:

- **HTTP/1.0:** Opened a brand-new TCP connection for every single request/response. Highly inefficient — you pay the connection-setup ("handshake") cost every time.
- **HTTP/1.1:** Introduced **persistent connections** (`keep-alive`) as the default, so multiple requests can reuse one connection.
- **HTTP/2.0:** Added **multiplexing** (many requests/responses concurrently on one connection), binary framing, header compression, and server push.
- **HTTP/3.0:** Replaced TCP with **QUIC** (built on UDP) for faster connection setup and better packet-loss handling, eliminating head-of-line blocking at the transport level.

**How connection handling evolved:**

```
HTTP/1.0   New TCP connection for every request — pay the handshake tax each time.
           [SYN→SYN/ACK→ACK] GET /a → 200 [close]
           [SYN→SYN/ACK→ACK] GET /b → 200 [close]

HTTP/1.1   One persistent connection, but responses come back in order.
           ── GET /a → 200 ── GET /b → 200 ── GET /c → 200 ──  (keep-alive)
                            ▲ /b must wait for /a to finish = head-of-line blocking

HTTP/2.0   One connection, many streams interleaved (multiplexing).
           ── /a /b /c sent together → each response returns as soon as it is ready ──
```

---

## 3. Anatomy of HTTP Messages

Client–server communication happens through structured text messages.

- **Request Message (client → server):** a request method (e.g., `GET`/`POST`), the resource path, the HTTP version, the `Host`, headers, a blank line, and an optional body.
- **Response Message (server → client):** the HTTP version, a status code (e.g., `200`), a status text (e.g., `OK`), headers, a blank line, and the response body.

**Example — a raw request and the response it produces:**

```
GET /api/books/42 HTTP/1.1          ← request line: method, path, version
Host: api.example.com               ← required in HTTP/1.1
Accept: application/json
Authorization: Bearer eyJhbG...
                                    ← blank line ends the headers (GET has no body)

HTTP/1.1 200 OK                     ← status line: version, code, status text
Content-Type: application/json
Content-Length: 48

{"id": 42, "title": "Dune", "author": "Herbert"}
```

The single blank line is structurally important: it's how the parser knows the headers have ended and the body (if any) begins.

---

## 4. HTTP Headers

Headers are key-value pairs that act as **metadata** for the message. They make the protocol extensible — think of them as a "remote control" the client and server use to dictate behavior.

- **Request Headers:** sent by the client to provide context (e.g., `User-Agent` identifies the client, `Authorization` carries credentials).
- **General Headers:** apply to both requests and responses (e.g., `Date`, `Connection`, `Cache-Control`).
- **Representation Headers:** describe the body (e.g., `Content-Type` for the media format, `Content-Length` for byte size, `Content-Encoding` for compression like gzip).
- **Security Headers:** protect against attacks (e.g., `Strict-Transport-Security` forces HTTPS, `Content-Security-Policy` mitigates cross-site scripting, `Set-Cookie` with the `HttpOnly` flag hides the cookie from JavaScript).

**Example — a response header block in practice:**

```
Content-Type: application/json; charset=utf-8
Cache-Control: max-age=3600
Content-Encoding: gzip
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
Strict-Transport-Security: max-age=31536000
```

---

## 5. HTTP Methods and Idempotency

Methods express the **intent** of a request.

| Method | Intent |
|--------|--------|
| `GET` | Fetch data without modifying anything |
| `POST` | Submit new data (has a body) |
| `PATCH` | Partially update an existing resource |
| `PUT` | Completely replace an existing resource |
| `DELETE` | Remove a resource |
| `OPTIONS` | Ask about server capabilities (used heavily in CORS) |

**Idempotency** is the key concept here:

- **Idempotent methods** can be run multiple times and leave the server in the same final state (e.g., `GET`, `PUT`, `DELETE`).
- **Non-idempotent methods** produce a different result each time they run (e.g., sending the same `POST` twice creates two separate resources).

**Example — idempotent vs. not:**

```
PUT /books/42  {"title": "Dune"}     run 1 → book 42 title = "Dune"
PUT /books/42  {"title": "Dune"}     run 2 → book 42 title = "Dune"   (identical end state) ✔ idempotent

POST /books    {"title": "Dune"}     run 1 → creates book #42
POST /books    {"title": "Dune"}     run 2 → creates book #43         (duplicate!)        → NOT idempotent
```

This is why a double-clicked "Submit" button can create two orders, while a retried `PUT` is safe.

---

## 6. Cross-Origin Resource Sharing (CORS)

Browsers enforce a **Same-Origin Policy**, which blocks web pages from making requests to a different origin (scheme + domain + port). CORS is the mechanism that lets servers safely opt in to cross-origin requests.

- **Simple requests** (typically `GET`/`POST` with standard headers and content types): the browser automatically adds an `Origin` header. If the server allows it, it replies with `Access-Control-Allow-Origin` containing the client's origin (or a `*` wildcard). If that header is missing, the browser blocks the response from reaching the page.
- **Pre-flight requests:** triggered when the request uses a non-simple method (`PUT`/`DELETE`), sends authorization headers, or uses an `application/json` content type.
  - The browser first fires an `OPTIONS` request asking the server whether the route permits the intended method and headers.
  - The server replies with `204 No Content`, explicitly listing the allowed origins, methods, headers, and a `max-age` to cache this answer.
  - If approved, the browser then sends the real request.

**The pre-flight handshake, step by step:**

```
Browser                                          Server
   │  OPTIONS /api/books                            │
   │  Origin: https://app.example.com               │
   │  Access-Control-Request-Method: PUT            │
   │ ───────────────────────────────────────────►  │  "Do you allow this?"
   │                                                │
   │  204 No Content                                │
   │  Access-Control-Allow-Origin: https://app...   │
   │  Access-Control-Allow-Methods: PUT, DELETE     │
   │  Access-Control-Max-Age: 86400  (cache this)   │
   │ ◄───────────────────────────────────────────  │
   │                                                │
   │  PUT /api/books   ← the real request now fires │
   │ ───────────────────────────────────────────►  │
   │  200 OK                                        │
   │ ◄───────────────────────────────────────────  │
```

> **Common gotcha:** CORS is enforced by the *browser*, not the server. A request from `curl` or Postman ignores CORS entirely — so CORS is not a substitute for authentication or authorization.

---

## 7. Standardized Status Codes

Status codes are three-digit numbers that act as a universal language for the outcome of a request.

- **1xx (Informational):** headers received, client can proceed (e.g., `100 Continue` for large uploads).
- **2xx (Success):**
  - `200 OK` — successful operation.
  - `201 Created` — usually follows a `POST` that created a resource.
  - `204 No Content` — success with no body (common for `OPTIONS` and `DELETE`).
- **3xx (Redirection):**
  - `301 Moved Permanently` — the resource has a new URL.
  - `302 Found` — temporary redirect to a new route.
  - `304 Not Modified` — tells the client to use its cached copy.
- **4xx (Client Errors):**
  - `400 Bad Request` — invalid/malformed data from the client.
  - `401 Unauthorized` — missing or invalid authentication.
  - `403 Forbidden` — authenticated, but lacks permission.
  - `404 Not Found` — wrong URL or deleted resource.
  - `405 Method Not Allowed` — wrong method for a route.
  - `409 Conflict` — business-logic violation (e.g., duplicate username).
  - `429 Too Many Requests` — client hit a rate limit.
- **5xx (Server Errors):**
  - `500 Internal Server Error` — an unhandled exception.
  - `501 Not Implemented` — feature not supported.
  - `502 Bad Gateway` / `504 Gateway Timeout` — a proxy/load balancer couldn't reach the upstream server.
  - `503 Service Unavailable` — server down or under maintenance.

> **Rule of thumb:** `4xx` means "you (the client) made a mistake"; `5xx` means "we (the server) made a mistake." Picking the right class is the difference between a frontend developer fixing their payload vs. paging your on-call engineer.

---

## 8. HTTP Caching

Caching reuses previously downloaded responses to save bandwidth and load time.

- On the first fetch, the server returns the payload plus three headers: `Cache-Control` (sets the max duration), `ETag` (a unique hash of the payload), and `Last-Modified`.
- On later requests, the client sends conditional headers: `If-None-Match` (carrying the `ETag`) or `If-Modified-Since`.
- If the resource hasn't changed, the server returns an empty `304 Not Modified`, telling the browser to reuse its cached copy. If it *has* changed, the server returns `200 OK` with the new data and a new `ETag`.

**Example — the conditional request flow:**

```
First request                     Next request (after cache expires)
─────────────                     ──────────────────────────────────
GET /logo.png                     GET /logo.png
                                  If-None-Match: "abc123"
200 OK                            304 Not Modified   ← empty body
ETag: "abc123"                    (browser reuses its cached copy)
Cache-Control: max-age=3600
<image bytes>
```

The win: the `304` response carries no body, so a 2 MB image costs only a few bytes of headers when it hasn't changed.

---

## 9. Content Negotiation and Compression

Clients and servers can negotiate the best representation to exchange.

- The client sends preferences via `Accept` (e.g., `application/json` vs. `application/xml`), `Accept-Language` (e.g., `en` vs. `es`), and `Accept-Encoding` (e.g., `gzip`).
- The server responds with the most appropriate format it supports.
- **Compression:** by negotiating an encoding like `gzip`, the server can drastically shrink text responses (e.g., a 26 MB JSON payload compressed to ~3.8 MB), saving large amounts of bandwidth.

```
Client →  Accept: application/json
          Accept-Encoding: gzip
Server →  Content-Type: application/json
          Content-Encoding: gzip          ← body is gzipped; the browser transparently decompresses it
```

---

## 10. Handling Large Data Transfers

- **Large client uploads (images/video):** plain JSON is terrible for binary data. Clients use a `multipart/form-data` request, which splits the payload into parts separated by a unique delimiter declared in the `boundary` parameter.
- **Large server downloads:** to avoid timeouts, the server **streams** the response in chunks (e.g., `Content-Type: text/event-stream` with `Connection: keep-alive`). The browser appends chunks as they arrive until the transfer completes.

**Example — a `multipart/form-data` upload body (a text field + a file in one request):**

```
POST /upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----X9Zk

------X9Zk
Content-Disposition: form-data; name="title"

My Vacation Photo
------X9Zk
Content-Disposition: form-data; name="file"; filename="beach.jpg"
Content-Type: image/jpeg

<raw binary bytes of the image>
------X9Zk--
```

The `boundary` string marks where each part begins and ends; the final delimiter adds a trailing `--`.

---

## 11. Security (SSL/TLS & HTTPS)

- **TLS (Transport Layer Security):** the modern, secure replacement for the older SSL protocol.
- It **encrypts data in transit** to prevent eavesdropping and tampering, and uses **certificates** to verify the server's identity.
- **HTTPS:** simply HTTP wrapped inside a TLS connection.

> Without TLS, anyone on the network path (a coffee-shop Wi-Fi router, an ISP, a malicious proxy) can read and modify your traffic. The certificate is what assures the client that it's really talking to `api.example.com` and not an impostor.

---

← Back to the [index](../README.md) · Next: [Chapter 2 — Routing in Backend »](02-routing.md)
