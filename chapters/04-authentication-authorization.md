# Chapter 4 — Authentication and Authorization

> Two words that sound alike and are constantly confused. Authentication is **who you are**; authorization is **what you're allowed to do**. You always authenticate *first*, then authorize.

---

## 1. Core Definitions

- **Authentication (the "Who"):** the mechanism that assigns an identity to a subject. It answers *"Who are you in this context?"*
- **Authorization (the "What"):** the process of determining a subject's permissions. It answers *"What are you allowed to do in this context?"*

---

## 2. The Evolution of Authentication

A quick history helps explain why modern systems look the way they do:

- **Pre-industrial (implicit trust):** human, contextual trust — handshakes, a village elder vouching for someone.
- **Medieval era (explicit authentication):** wax seals as physical tokens based on **possession**. Also the first "bypass attacks" — forgery.
- **Industrial revolution:** telegraph operators used pre-agreed passphrases — a shift to **something you know**.
- **1960s (mainframes):** MIT introduced passwords for multi-user systems. Initially stored in plain text; after an incident where the password file was accidentally printed, **hashing** was invented (turning passwords into irreversible, fixed-length strings).
- **1970s:** Diffie–Hellman introduced **asymmetric cryptography**, the backbone of modern Public Key Infrastructure (PKI) and ticket-based auth (Kerberos).
- **1990s (MFA):** **Multi-Factor Authentication** combined three principles — something you *know* (password), something you *have* (OTP), something you *are* (biometrics).
- **Future trends:** post-quantum cryptography, decentralized identity (blockchain), and behavioral biometrics.

---

## 3. Three Core Components of Modern Authentication

### A. Sessions

- **The problem:** HTTP is stateless — it remembers nothing between requests. But real apps (e.g., e-commerce) need to remember who's logged in.
- **The solution:** on login, the server creates a unique **Session ID**, stores the user's data against that ID in a fast store like **Redis**, and sends the ID to the client. The client returns the ID on every future request, giving the server "memory."

### B. JWTs (JSON Web Tokens)

- **The problem:** at global scale, storing and synchronizing millions of sessions across many servers causes latency and storage cost.
- **The solution:** JWTs are a **stateless** mechanism that moves storage off the server and into the token itself.
- **Structure — three base64url parts:**
  1. **Header:** metadata such as the signing algorithm.
  2. **Payload:** "claims" such as the user ID (`sub`), issued-at time (`iat`), and roles.
  3. **Signature:** computed with a secret key held only by the server, proving the token wasn't tampered with.

**Example — a JWT is three base64url parts joined by dots (`header.payload.signature`):**

```
eyJhbGciOiJIUzI1NiJ9 . eyJzdWIiOiIxMjMiLCJyb2xlIjoiYWRtaW4ifQ . s5xK_8r...signature
└───── header ─────┘   └─────────── payload ───────────┘        └─── signature ───┘

header  decodes to:  { "alg": "HS256", "typ": "JWT" }
payload decodes to:  { "sub": "123", "role": "admin", "iat": 1718700000, "exp": 1718703600 }
```

**Key insight:** the payload is only **encoded**, not **encrypted** — anyone can read it. Security comes from the signature, `HMAC(secret, header + "." + payload)`. Only the server knows the secret, so a tampered token fails verification. **Never put secrets in a JWT payload.**

### C. Cookies

- **Definition:** a mechanism for the server to store small pieces of data in the user's browser.
- **Workflow:** the server sets an `HttpOnly` cookie (so JavaScript can't read it) holding the auth token. The browser then automatically attaches that cookie to every subsequent request to that server.

```
Server →  Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
Browser →  (every later request) Cookie: session=abc123   ← attached automatically
```

`HttpOnly` blocks XSS from stealing it; `Secure` sends it only over HTTPS; `SameSite` mitigates CSRF.

---

## 4. Major Types of Authentication

### 1. Stateful Authentication

- **Workflow:** user logs in → server stores session in Redis → server sends Session ID via cookie → client sends cookie next time → server looks up the ID in Redis.
- **Pros/cons:** centralized control and easy revocation, but lower scalability and more operational complexity. Ideal for standard web apps.

### 2. Stateless Authentication

- **Workflow:** user logs in → server signs a JWT with a secret → client sends the JWT in the `Authorization` header → server verifies the signature **without a database lookup**.
- **Pros/cons:** highly scalable and portable, but revoking access *before* expiry is hard (short of rotating the secret, which logs everyone out).

**Stateful vs. stateless, side by side:**

```
Stateful (session)                      Stateless (JWT)
──────────────────                      ───────────────
login   → store session in Redis,       login   → sign a JWT with the secret key,
          send Session ID (cookie)                 store nothing, send the JWT
request → look up ID in Redis (I/O)      request → verify signature (math, no I/O)
revoke  → delete from Redis (easy)       revoke  → hard; token lives until it expires
```

### 3. API Key-Based Authentication

- **Purpose:** for programmatic, **machine-to-machine** communication (e.g., your backend calling OpenAI's API).
- **Workflow:** a user generates a cryptographically random string in a UI; their server includes that key in automated requests, bypassing human login forms.

### 4. OAuth 2.0 & OpenID Connect (OIDC)

- **The delegation problem:** historically, users shared their password so one app could access another's data (e.g., a travel app reading Gmail). A huge security risk, and revocation was impossible.
- **OAuth 2.0:** lets a client app request a **token with limited permissions** (e.g., read-only contacts) from an Authorization Server. This handles **authorization**.
- **OpenID Connect (OIDC):** OAuth alone doesn't verify *identity*, so OIDC was layered on top. It adds the **ID Token** (a JWT) that securely shares profile data (email, name). This powers "Sign in with Google."

---

## 5. Authorization: Role-Based Access Control (RBAC)

- **The concept:** not all users get the same capabilities (a standard user vs. an admin who can access deleted-file "dead zones").
- **How it works:** users are assigned **roles** (User, Admin, Moderator); each role maps to specific resource permissions (Read, Write, Delete).
- **Execution:** during the request, the server derives the user's role from their token. If an unauthorized user attempts an action, the server rejects it with `403 Forbidden`.

**Example — a simple role → permission matrix:**

```
Role        | Read | Write | Delete
------------|------|-------|-------
User        |  ✔   |   ✗   |   ✗
Moderator   |  ✔   |   ✔   |   ✗
Admin       |  ✔   |   ✔   |   ✔
```

A `User` calling `DELETE /posts/42` → server reads role from token → role lacks Delete → `403 Forbidden`.

---

## 6. Critical Security Best Practices

- **Use generic error messages.** Never respond "User not found" or "Incorrect password." Attackers use specific messages to confirm which usernames exist. Always reply generically, e.g., *"Authentication failed."*

**Example — why specificity leaks information:**

```
✗ Leaky:
   POST /login (unknown user) → 404 "User not found"
   POST /login (wrong pass)   → 401 "Incorrect password"   ← attacker learns the username is real

✓ Safe — identical response either way:
   POST /login                → 401 "Authentication failed"
```

- **Prevent timing attacks.** Attackers can measure response time. If a username is invalid, the server might fail instantly; if valid but the password is wrong, the server takes longer because it runs the heavy password-hashing algorithm. That timing difference leaks which usernames exist.
  - *Solution:* equalize response times with **constant-time comparisons**, or run the hash even for unknown users / add a fixed delay, so the attacker can't tell the two cases apart.

---

← Previous: [« Chapter 3 — Serialization](03-serialization.md) · Back to the [index](../README.md) · Next: [Chapter 5 — Validations & Transformations »](05-validations-transformations.md)
