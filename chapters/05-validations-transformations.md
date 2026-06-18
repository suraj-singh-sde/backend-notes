# Chapter 5 — Validations and Transformations

> The guard at the front door. Before any business logic runs, you confirm the incoming data is shaped the way you expect — and reshape it when convenient.

---

## 1. What Are Validations and Transformations?

- **Purpose:** ensure **data integrity and security**. They guarantee that any data entering your server matches the exact format, type, and logical constraints your application requires.
- **Scope:** applies to *all* incoming client data — JSON body payloads, query parameters, path parameters, and headers.

---

## 2. Where Do They Fit in Backend Architecture?

To know *where* validation happens, recall the standard backend layering (covered in [Chapter 6](06-layered-architecture.md)):

- **Repository layer (bottom):** direct database connections and queries.
- **Service layer (middle):** core business logic.
- **Controller layer (top / entry point):** HTTP-specific logic — reading requests, sending status codes.

**Validation and transformation happen in the Controller layer** — immediately after the route is matched, but **before** any service or database method is called.

```
Request → [Route matched] → [Validate + Transform] → Controller → Service → Repository
                                     ▲
                            the guard runs HERE, before any real work
```

---

## 3. Why Is Backend Validation Critical?

If you don't validate at the entry point, bad data flows downstream and your server breaks in confusing ways.

- **The 500 problem:** suppose the database expects `name` to be text. If a client sends the number `0` and you don't validate, the bad value travels all the way to the database, which rejects the type mismatch and throws a `500 Internal Server Error`. From the client's side, it looks like *your* server is broken.
- **The 400 solution:** with validation at the entry point, the server catches the mistake instantly, never makes the database call, and returns a clear `400 Bad Request` (e.g., "Name must be a string").

```
Without validation:  client sends name=0 → service → DB rejects type → 💥 500 (looks like our bug)
With validation:     client sends name=0 → caught at the door → 400 "Name must be a string" ✔
```

---

## 4. How the Validation Pipeline Works (Step by Step)

A robust validation step processes data in layers, each gating the next:

1. **Existence check:** does the expected field (e.g., `name`) exist? If not → "Field is required."
2. **Type check:** if it exists, is it the right type? (string vs. array vs. boolean)
3. **Constraint check:** if the type is right, does it meet the rules? (e.g., length between 5 and 100)

**Example — the pipeline as pseudocode for a `name` field:**

```js
function validateName(body) {
  if (body.name === undefined)        return error("name is required");      // 1. existence
  if (typeof body.name !== "string")  return error("name must be a string"); // 2. type
  if (body.name.length < 5 ||
      body.name.length > 100)         return error("name must be 5–100 chars"); // 3. constraint
  return ok();
}
```

Order matters: you can't check the *length* of something until you know it *exists* and is a *string*.

---

## 5. The Four Main Types of Validation

1. **Type validation:** check basic data types (string, number, boolean, array). Can be recursive — e.g., confirm an array contains only strings.
2. **Syntactic validation:** check that a string follows a structural pattern — email format (`@` + domain), phone numbers, dates (`YYYY-MM-DD`).
3. **Semantic validation:** check that data makes **logical sense** in the real world, even when type and syntax are valid — a date of birth can't be in the future; an age can't be 430.
4. **Complex (dependent) validation:** validate a field based on *other* fields — e.g., "Confirm Password" must equal "Password"; a "Partner Name" is required only if "Married" is `true`.

```
Type:       age must be a number
Syntactic:  email must match  ^[^@]+@[^@]+\.[^@]+$
Semantic:   dateOfBirth must be in the past
Dependent:  confirmPassword must equal password
```

---

## 6. What Is Transformation (Type Casting)?

- **Definition:** modifying incoming data to put it into a convenient shape *before* the service layer uses it.
- **Handling query parameters:** query params arrive as **strings, always**. A classic case is pagination (`?page=2&limit=20`): if your validation expects numbers, the raw strings fail. Transformation **casts** the strings to numbers so they pass validation.
- **Data normalization:** transformation also cleans data — e.g., lowercasing an email, or prepending `+` to a phone number, before saving.

**Example — transform *then* validate:**

```
Incoming:   ?page=2&limit=20        →  { page: "2", limit: "20" }   (strings!)
Transform:  cast to numbers         →  { page: 2,   limit: 20 }
Normalize:  email "Bob@X.COM"       →  "bob@x.com"
Validate:   page ≥ 1, limit ≤ 100   →  ✔ passes, hand clean data to the service
```

---

## 7. Frontend vs. Backend Validation (the Golden Rule)

A common and dangerous mistake: assuming that frontend validation means you can skip backend validation.

- **Frontend validation is for UX:** instant, friendly feedback in a form improves the user experience.
- **Backend validation is for security:** the frontend can be bypassed entirely. Anyone can hit your API directly with Postman, Insomnia, or `curl`, skipping the UI. Therefore **backend validation is mandatory** — it's the only validation you actually control.

> Treat the frontend's checks as a courtesy to honest users, and the backend's checks as the real defense against everyone else.

---

← Previous: [« Chapter 4 — Authentication & Authorization](04-authentication-authorization.md) · Back to the [index](../README.md) · Next: [Chapter 6 — Layered Architecture »](06-layered-architecture.md)
