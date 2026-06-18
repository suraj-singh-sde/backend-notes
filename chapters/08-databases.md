# Chapter 8 — Mastering Databases with Postgres

> Where the data actually lives. Everything in the earlier chapters — requests, validation, business logic — eventually comes down to reading from and writing to a database safely and quickly.

---

## 1. Fundamental Concepts

- **Persistence:** the core purpose of a database is to make data survive after the program (and the machine) stops running.
- **Disk vs. RAM:**
  - **RAM (primary memory):** very fast, but volatile and expensive. Used for caching (e.g., Redis — see [Chapter 9](09-caching.md)).
  - **Disk (secondary memory):** slower, but cheaper and persistent. Databases (Postgres, Mongo) store data here to hold large volumes (terabytes) reliably.
- **Why not just text files?** Plain files struggle with parse speed, have no enforced structure/schema, and — most importantly — can't handle **concurrency** (many users updating the same data at once) without corrupting data.

---

## 2. Choosing a Database

- **Relational (SQL):** data in tables/rows/columns with a strict schema. Best when data integrity matters (e.g., CRM, finance).
- **Non-Relational (NoSQL):** flexible, schema-less documents. Good for dynamic, unstructured content (e.g., a CMS).
- **Why Postgres?**
  - **Open source & standards-compliant:** free, and it adheres to SQL standards, easing migration.
  - **JSON support (the killer feature):** the `JSON`/`JSONB` types let you store unstructured, document-style data *inside* a relational database — much of MongoDB's flexibility without leaving Postgres.

| | Relational (SQL) | Non-Relational (NoSQL) |
|---|---|---|
| Schema | Strict, predefined | Flexible / schema-less |
| Strength | Integrity, complex queries, joins | Flexibility, horizontal scale |
| Example use | Banking, CRM | CMS, event logs, catalogs |

---

## 3. Postgres Data Types & Best Practices

- **Integers:**
  - `SERIAL` / `BIGSERIAL` — auto-incrementing integers, usually for IDs.
  - `SMALLINT`, `INTEGER`, `BIGINT` — pick based on the magnitude you need.
- **Floats vs. decimals:**
  - `DECIMAL` / `NUMERIC` — exact precision. **Always use this for money** to avoid rounding errors.
  - `REAL` / `FLOAT` — floating point. Faster but slightly imprecise; use for scientific values, never currency.
- **Strings:**
  - `CHAR(n)` — fixed length, pads with spaces. Avoid.
  - `VARCHAR(n)` — variable length with a cap. (The famous `255` limit is just a MySQL-era habit.)
  - `TEXT` — variable length, no limit. **Prefer `TEXT`** in Postgres: equally performant and saves you from migration pain when a length limit turns out to be too small.
- **Other useful types:**
  - `UUID` — universally unique identifier. Safer than sequential integer IDs in distributed systems (IDs don't leak row counts and don't collide across shards).
  - `JSONB` — binary JSON. **Always prefer it over `JSON`** because Postgres can index and query it efficiently.
  - `ENUM` — a custom type restricted to a fixed set of values (e.g., `'pending'`, `'completed'`). Great for integrity and self-documenting columns.

**Example — `DECIMAL` vs. `FLOAT` for money:**

```sql
-- ✗ Float: 0.1 + 0.2 does NOT exactly equal 0.3 in binary floating point
SELECT 0.1::float + 0.2::float;   -- → 0.30000000000000004

-- ✔ Numeric: exact decimal arithmetic
SELECT 0.1::numeric + 0.2::numeric;  -- → 0.3
```

---

## 4. Database Migrations

- **Definition:** migrations are **version control for your schema**.
- **Workflow:**
  - **Up migration:** applies a change (e.g., `CREATE TABLE`).
  - **Down migration:** reverts it (e.g., `DROP TABLE`) so you can roll back a bad release.
- **Why:** they track schema changes over time and guarantee every developer and every server runs the *exact same* structure.

```sql
-- 0007_add_priority_to_tasks.up.sql
ALTER TABLE tasks ADD COLUMN priority SMALLINT NOT NULL DEFAULT 3;

-- 0007_add_priority_to_tasks.down.sql   (the rollback)
ALTER TABLE tasks DROP COLUMN priority;
```

---

## 5. Data Modeling (Relationships)

- **Naming conventions:** use **plural** table names (`users`, `projects`) and **snake_case** columns (`full_name`), because Postgres folds unquoted identifiers to lowercase.

**One-to-One** (User ↔ Profile) — split to keep the main table lightweight; the profile's PK *is* the FK:

```sql
CREATE TABLE users (
  id    BIGSERIAL PRIMARY KEY,
  email TEXT UNIQUE NOT NULL
);

CREATE TABLE user_profiles (
  user_id BIGINT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,  -- PK and FK in one
  bio     TEXT
);
```

**One-to-Many** (Project → many Tasks) — the "many" side holds the foreign key:

```sql
CREATE TABLE projects (
  id   BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL
);

CREATE TABLE tasks (
  id         BIGSERIAL PRIMARY KEY,
  project_id BIGINT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,  -- the FK lives here
  title      TEXT NOT NULL
);
```

**Many-to-Many** (Users ↔ Projects) — needs a **linking table** with a **composite primary key**:

```sql
CREATE TABLE project_members (
  user_id    BIGINT REFERENCES users(id)    ON DELETE CASCADE,
  project_id BIGINT REFERENCES projects(id) ON DELETE CASCADE,
  role       TEXT NOT NULL DEFAULT 'member',
  PRIMARY KEY (user_id, project_id)   -- composite key prevents duplicate memberships
);
```

---

## 6. Constraints & Integrity

Constraints let the **database itself** enforce rules, so bad data can't get in even if the application has a bug.

- **Primary Key:** implicitly `UNIQUE` and `NOT NULL`.
- **Foreign Key:** you can't reference a row that doesn't exist.
- **Check Constraint:** custom logic at the DB level.
- **Referential integrity (`ON DELETE`):**
  - `RESTRICT` — block deleting a user who still owns projects.
  - `CASCADE` — deleting a project automatically deletes its tasks.

```sql
CREATE TABLE tasks (
  id         BIGSERIAL PRIMARY KEY,
  project_id BIGINT NOT NULL REFERENCES projects(id) ON DELETE RESTRICT,
  priority   SMALLINT CHECK (priority BETWEEN 1 AND 5),  -- check constraint
  status     TEXT NOT NULL DEFAULT 'todo'
);
```

> Treat constraints as a safety net *in addition to* application validation ([Chapter 5](05-validations-transformations.md)), not a replacement. Validation gives users friendly errors; constraints guarantee integrity no matter what reaches the database.

---

## 7. Transactions & ACID

A **transaction** groups several statements into one all-or-nothing unit: either every statement commits, or none of them do. This is the single most important reason to reach for a relational database.

Transactions guarantee **ACID**:

| Property | Meaning |
|----------|---------|
| **Atomicity** | All statements succeed together, or all roll back. No half-finished work. |
| **Consistency** | The database moves from one valid state to another; constraints always hold. |
| **Isolation** | Concurrent transactions don't see each other's uncommitted changes. |
| **Durability** | Once committed, data survives a crash or power loss. |

**Example — a money transfer must be atomic:**

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- debit sender
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- credit receiver
COMMIT;   -- both happen, or...
-- ROLLBACK;  ← if anything fails, the debit is undone too — money never vanishes
```

Without a transaction, a crash *between* the two `UPDATE`s would debit the sender without crediting the receiver — 100 units gone.

> **Isolation levels** (`READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`) trade strictness for concurrency. Postgres defaults to `READ COMMITTED`; raise it when you must prevent subtle race conditions like double-spending.

---

## 8. Performance & Security

### Parameterized Queries (SQL Injection)

**Never** build queries by concatenating strings. Use placeholders — the driver treats the input strictly as a value, never as executable SQL.

```js
// ✗ DANGEROUS: a value like  '; DROP TABLE users; --  becomes executable SQL
db.query(`SELECT * FROM users WHERE email = '${email}'`);

// ✔ SAFE: the value is bound separately and can never change the query's structure
db.query('SELECT * FROM users WHERE email = $1', [email]);
```

### Indexes

- **Concept:** like a book's index, it lets the database jump straight to matching rows instead of scanning every row (a *sequential scan*).
- **When to index:** columns used in `WHERE`, `JOIN` conditions, or `ORDER BY`.
- **Trade-off:** indexes speed up reads but slow down writes slightly, because each `INSERT`/`UPDATE` must also maintain the index.

```sql
CREATE INDEX idx_tasks_project_id ON tasks(project_id);

-- See whether a query uses the index or scans the whole table:
EXPLAIN ANALYZE SELECT * FROM tasks WHERE project_id = 42;
--   "Index Scan using idx_tasks_project_id"  ✔  (good)
--   "Seq Scan on tasks"                       ✗  (no usable index)
```

### Triggers

Used to automate tasks at the DB level. A classic example keeps `updated_at` current so application code doesn't have to:

```sql
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS trigger AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_set_updated_at
  BEFORE UPDATE ON tasks
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

### Connection Pooling

Opening a new database connection per request is expensive (Postgres connections are heavyweight processes). A **connection pool** (e.g., PgBouncer, or your driver's built-in pool) keeps a set of reusable connections open and hands them out, dramatically cutting latency under load.

### The N+1 Query Problem

A subtle performance killer: fetching a list, then firing one extra query *per item*.

```
✗ N+1:  1 query for 100 projects, then 100 queries (one per project) for their tasks  = 101 queries
✔ Fix:  fetch projects, then ONE query for all their tasks using WHERE project_id = ANY($1)  = 2 queries
        (or a single JOIN)
```

---

## 9. API Query Design

- **Fetching lists — always paginate** with `LIMIT`/`OFFSET` (ties back to the list-API design in [Chapter 7](07-rest-api-design.md)):

  ```sql
  SELECT * FROM books ORDER BY created_at DESC LIMIT 20 OFFSET 40;  -- page 3, 20 per page
  ```

  > **Scaling note:** `OFFSET` gets slow on deep pages (the DB still scans and discards the skipped rows). For large datasets, prefer **keyset (cursor) pagination**: `WHERE created_at < $last_seen ORDER BY created_at DESC LIMIT 20`.

- **Filtering** — use `ILIKE` for case-insensitive pattern matching:

  ```sql
  SELECT * FROM users WHERE name ILIKE '%zack%';
  ```

- **Joins** — use `LEFT JOIN` to keep rows from the main table even when the related table has nothing:

  ```sql
  -- Returns every user, with profile columns NULL for users who have no profile
  SELECT u.id, u.email, p.bio
  FROM users u
  LEFT JOIN user_profiles p ON p.user_id = u.id;
  ```

---

← Previous: [« Chapter 7 — REST API Design](07-rest-api-design.md) · Back to the [index](../README.md) · Next: [Chapter 9 — Caching »](09-caching.md)
