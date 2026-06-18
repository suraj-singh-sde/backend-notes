# Chapter 3 — Serialization and Deserialization

> How two programs written in different languages manage to understand each other over a network.

---

## 1. The Core Problem: Language Barriers in Networking

- **Different environments:** a typical web app has a client (say, a JavaScript frontend) and a server (say, a Rust backend).
- **Incompatible data types:** these languages model data very differently. JavaScript is dynamic and interpreted; Rust is compiled with strict, static types.
- **The challenge:** when the JavaScript client sends an in-memory object across the internet, the Rust server can't natively understand a JavaScript object. They need a **shared, neutral language** to exchange data.

---

## 2. What Are Serialization and Deserialization?

- **The solution:** both sides agree on a single common format for data on the wire.
- **Serialization:** converting native data (a JavaScript object, a Rust struct) *into* that common format before sending it.
- **Deserialization:** the reverse — taking the received common format and converting it *back* into the receiver's native type so it can run business logic.
- **Purpose:** this makes communication **language-agnostic** and **domain-agnostic** — any machine can talk to any other machine regardless of the tech each one runs.

---

## 3. Types of Serialization Standards

Formats fall into two broad categories:

1. **Text-based formats** — human-readable. Examples: **JSON**, YAML, XML.
2. **Binary formats** — compiled to compact raw bytes for efficient transmission. Examples: **Protobuf**, Avro.

| | Text-based (e.g., JSON) | Binary (e.g., Protobuf) |
|---|---|---|
| Readability | Human-readable | Not human-readable |
| Size | Larger | Smaller / faster |
| Typical use | Public REST APIs, configs, logs | High-throughput internal/microservice traffic |

---

## 4. Deep Dive into JSON (the Industry Standard)

For traditional HTTP REST APIs, **JSON (JavaScript Object Notation)** is the most popular format — used roughly 80% of the time.

- **Use cases:** beyond HTTP, JSON is heavily used for application logs and configuration files.
- **Characteristics:** despite the name, it isn't tied to JavaScript, and it's prized for being human-readable.
- **Strict syntax rules:**
  - An object starts with `{` and ends with `}`.
  - **Keys** must be strings wrapped in double quotes (e.g., `"name":`).
  - **Values** are limited to a fixed set of types: strings, numbers, booleans, arrays, `null`, or nested objects.

**Example — one JSON object showing every value type:**

```json
{
  "id": 42,
  "title": "Dune",
  "inStock": true,
  "tags": ["sci-fi", "classic"],
  "author": { "name": "Frank Herbert", "born": 1920 },
  "discount": null
}
```

> Note what JSON *can't* represent directly: dates, binary blobs, comments, or trailing commas. Dates are conventionally sent as ISO-8601 strings (`"2026-06-18T10:00:00Z"`); binary data is base64-encoded into a string.

---

## 5. The Backend Engineer's Mental Model (OSI Layers)

Data crosses many network layers (the OSI model), but you don't have to think about most of them.

- **The abstraction:** as a backend engineer, focus on the **Application Layer**. Assume the client serializes data into JSON and sends it.
- **The intermediary steps:** under the hood, the network turns that JSON into data frames, IP packets, and finally physical bits (`0`s and `1`s) carried by electrical/optical signals.
- **The receiving end:** before your server touches anything, the network reassembles those bits back into the exact JSON the client sent. You never deal with the intermediate conversions — only the JSON.

---

## 6. The Complete End-to-End Workflow

1. **Client prepares data:** the frontend gathers user input.
2. **Serialization (client):** the client converts its internal data into a JSON string and attaches it to the request body.
3. **Transmission:** the data is broken into bits, travels the internet, and reaches the server.
4. **Deserialization (server):** the server parses the JSON string into its native type (e.g., a Rust struct).
5. **Processing:** the server runs business logic (e.g., saving to a database).
6. **Serialization (server):** the server packages its response back into JSON and sends the HTTP response.
7. **Deserialization (client):** the frontend parses the JSON response back into a native object and updates the UI.

**The round trip at a glance:**

```
Client (JavaScript)                                   Server (Rust)
  {id: 42}  ──serialize──►  "{\"id\":42}"  ──bits over network──►
                                              "{\"id\":42}"  ──deserialize──►  struct {id: 42}
                                                                                     │ business logic
  object  ◄──deserialize──  "{\"ok\":true}"  ◄──bits over network──  serialize──◄  {ok: true}
```

Each side only ever touches its own native type and the shared JSON text — the bits-on-the-wire conversion is handled invisibly by the lower network layers.

---

← Previous: [« Chapter 2 — Routing](02-routing.md) · Back to the [index](../README.md) · Next: [Chapter 4 — Authentication & Authorization »](04-authentication-authorization.md)
