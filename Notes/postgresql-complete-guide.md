# PostgreSQL Complete Reference Guide

> Database design, DDL, and JSON ingestion. Built for fast lookup.
> Targets **PostgreSQL 18** (current stable). Everything here also works on 15–17.

---

## Table of Contents

1. [Mental Model: Scopes & Hierarchy](#1-mental-model-scopes--hierarchy)
2. [psql Survival Kit](#2-psql-survival-kit)
3. [Databases](#3-databases)
4. [Schemas](#4-schemas)
5. [Roles, Users & Permissions](#5-roles-users--permissions)
6. [Data Types](#6-data-types)
7. [Tables](#7-tables)
8. [Constraints](#8-constraints)
9. [ALTER TABLE — Modifying Anything](#9-alter-table--modifying-anything)
10. [Auto-Increment & Identity](#10-auto-increment--identity)
11. [Relationships (1:1, 1:N, M:N)](#11-relationships-11-1n-mn)
12. [Indexes](#12-indexes)
13. [Views & Materialized Views](#13-views--materialized-views)
14. [Functions & Procedures](#14-functions--procedures)
15. [Triggers](#15-triggers)
16. [Sequences](#16-sequences)
17. [Generated Columns](#17-generated-columns)
18. [Transactions](#18-transactions)
19. [Working With JSON Inside Postgres](#19-working-with-json-inside-postgres)
20. [Importing Data via JSON](#20-importing-data-via-json) ← **the JSON ingestion section**
21. [Importing Data via CSV](#21-importing-data-via-csv)
22. [Inspecting the Database (Introspection)](#22-inspecting-the-database-introspection)
23. [Dropping & Cleanup Reference](#23-dropping--cleanup-reference)
24. [Conventions & Best Practices](#24-conventions--best-practices)

---

## 1. Mental Model: Scopes & Hierarchy

PostgreSQL nests objects in this order. Knowing the scope tells you what command to use and what a name refers to.

```
Cluster (one running PostgreSQL server / one data directory)
└── Database          ← you connect to exactly ONE at a time; cannot query across them
    └── Schema        ← a namespace inside a database (default: "public")
        └── Tables, Views, Functions, Sequences, Types, Indexes, Triggers...
```

Key consequences:

- **You cannot join across databases.** If two things need to talk, they belong in the same database, in different schemas.
- **Schemas are your organizational tool.** Use them to group a tool's tables (e.g. `app`, `staging`, `audit`).
- **Roles (users) live at the cluster level**, above databases — one login works across all databases (subject to permissions).
- A fully-qualified name is `database.schema.object`, but within a connection you write `schema.object` (e.g. `app.customer`), or just `object` if it's on your `search_path`.

---

## 2. psql Survival Kit

`psql` is the built-in command-line client. Meta-commands start with `\`.

### Connecting
```bash
psql -U <user> -h localhost -d <database>      # standard local connection
psql -U postgres -h localhost                  # connect as superuser
psql "postgresql://user:pass@host:5432/dbname" # connection-string form
```

### Essential meta-commands
```text
\l                list databases
\c dbname         connect to (switch to) a database
\dn               list schemas
\dt               list tables (current schema)
\dt app.*         list tables in schema "app"
\d tablename      describe a table (columns, types, constraints, indexes)
\d+ tablename     describe with more detail (storage, comments)
\di               list indexes
\dv               list views
\df               list functions
\du               list roles/users
\sf functionname  show a function's source
\x                toggle expanded display (great for wide rows)
\timing           toggle query timing on/off
\conninfo         show current connection
\i file.sql       run a .sql script file
\e                open editor to compose a query
\q                quit
```

### Running a script from the OS shell
```bash
psql -U dalto -h localhost -d appdb -f schema.sql
```

---

## 3. Databases

### Create
```sql
CREATE DATABASE appdb;
CREATE DATABASE appdb OWNER dalto;                  -- assign an owner
CREATE DATABASE appdb
    OWNER       dalto
    ENCODING    'UTF8'
    LC_COLLATE  'en_US.UTF-8'
    LC_CTYPE    'en_US.UTF-8'
    TEMPLATE    template0;                           -- needed when overriding locale
```

### Modify
```sql
ALTER DATABASE appdb RENAME TO app_production;
ALTER DATABASE appdb OWNER TO dalto;
ALTER DATABASE appdb SET timezone TO 'UTC';         -- set a default param for the db
```

### Remove
```sql
DROP DATABASE appdb;
DROP DATABASE IF EXISTS appdb;
DROP DATABASE appdb WITH (FORCE);                   -- kick off connected users first
```
> You can't drop the database you're currently connected to. Connect to another (e.g. `\c postgres`) first.

---

## 4. Schemas

A schema is a namespace for objects inside a database.

### Create
```sql
CREATE SCHEMA app;
CREATE SCHEMA app AUTHORIZATION dalto;              -- owned by a role
CREATE SCHEMA IF NOT EXISTS staging;
```

### Modify
```sql
ALTER SCHEMA app RENAME TO application;
ALTER SCHEMA app OWNER TO dalto;
```

### Remove
```sql
DROP SCHEMA app;                                    -- fails if it contains objects
DROP SCHEMA app CASCADE;                            -- drops the schema AND everything in it
DROP SCHEMA IF EXISTS app CASCADE;
```

### search_path (which schemas are searched for unqualified names)
```sql
SHOW search_path;                                   -- see current
SET search_path TO app, public;                     -- this session only
ALTER ROLE dalto SET search_path TO app, public;    -- persistent for this user
ALTER DATABASE appdb SET search_path TO app, public;-- persistent for the database
```
With `search_path = app, public`, writing `customer` resolves to `app.customer` automatically.

---

## 5. Roles, Users & Permissions

In Postgres a **USER is just a ROLE with LOGIN**. `CREATE USER` = `CREATE ROLE ... LOGIN`.

### Create roles
```sql
CREATE ROLE dalto LOGIN PASSWORD 'secret';          -- a login account
CREATE ROLE readonly;                               -- a group role (no login) for grants
CREATE ROLE app_admin LOGIN PASSWORD 'secret'
    CREATEDB CREATEROLE;                             -- elevated privileges
```

### Modify roles
```sql
ALTER ROLE dalto PASSWORD 'newsecret';
ALTER ROLE dalto RENAME TO dalton;
ALTER ROLE dalto CREATEDB;                           -- grant an attribute
ALTER ROLE dalto NOCREATEDB;                         -- revoke an attribute
ALTER ROLE dalto VALID UNTIL '2027-01-01';           -- password expiry
GRANT readonly TO dalto;                             -- add dalto to the readonly group
REVOKE readonly FROM dalto;
```

### Remove roles
```sql
DROP ROLE dalto;
DROP ROLE IF EXISTS dalto;
```
> You must reassign or drop a role's owned objects first:
> `REASSIGN OWNED BY dalto TO postgres;` then `DROP OWNED BY dalto;`

### Permissions (GRANT / REVOKE)
Privileges apply at database, schema, and object levels — all three may be needed.

```sql
-- Database level
GRANT CONNECT ON DATABASE appdb TO readonly;

-- Schema level (USAGE = "allowed to look inside")
GRANT USAGE ON SCHEMA app TO readonly;

-- Table level
GRANT SELECT ON app.customer TO readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON app.customer TO app_admin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA app TO app_admin;

-- Revoke
REVOKE INSERT ON app.customer FROM app_admin;
```

### Default privileges (auto-grant on FUTURE objects)
Without this, roles can't see tables you create *later*.
```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA app
    GRANT SELECT ON TABLES TO readonly;
```

---

## 6. Data Types

Pick the narrowest correct type. Common types and when to use them:

### Numbers
| Type | Use for |
|---|---|
| `smallint` | small ints (−32k to 32k) |
| `integer` / `int` | default whole numbers |
| `bigint` | large ints, IDs, counts |
| `numeric(p,s)` | **exact** decimals — money, anything where rounding matters |
| `real` / `double precision` | floating point — science/measurements, **never money** |

```sql
price_cents integer            -- store money as integer cents (preferred)
amount      numeric(12,2)      -- or exact decimal with 2 places
```
> **Never store money as float.** Use integer cents or `numeric`.

### Text
| Type | Use for |
|---|---|
| `text` | **default choice** for all strings — no length limit, no performance penalty |
| `varchar(n)` | only when you truly need a hard length cap |
| `char(n)` | almost never (pads with spaces) |

### Boolean
```sql
is_active boolean DEFAULT true      -- accepts true/false, 't'/'f', 1/0
```

### Date & Time
| Type | Use for |
|---|---|
| `date` | calendar date, no time |
| `time` | time of day, no date |
| `timestamp` | date+time, **no** timezone (avoid) |
| `timestamptz` | date+time **with** timezone — **preferred for all timestamps** |
| `interval` | a duration ("3 days", "2 hours") |

```sql
created_at timestamptz NOT NULL DEFAULT now()
```
Useful functions: `now()`, `CURRENT_DATE`, `CURRENT_TIMESTAMP`, `age(...)`.

### UUID
```sql
id uuid PRIMARY KEY DEFAULT gen_random_uuid()   -- gen_random_uuid() is built-in (PG13+)
```

### JSON
| Type | Use for |
|---|---|
| `json` | raw text storage, preserves formatting/key order (rarely what you want) |
| `jsonb` | **preferred** — binary, indexable, supports operators, deduplicates keys |

### Arrays
```sql
tags text[]                        -- an array of text
INSERT ... VALUES (ARRAY['a','b']);
SELECT * FROM t WHERE 'a' = ANY(tags);
```

### Enum (a fixed set of values)
```sql
CREATE TYPE order_status AS ENUM ('pending', 'shipped', 'cancelled');
-- column:  status order_status NOT NULL DEFAULT 'pending'
```
> **Tradeoff:** enums are fast and self-documenting but **hard to modify** (reordering/removing values is painful). For value sets that may change, prefer a **lookup table** with a foreign key (see §11).

### Other useful types
`bytea` (binary blobs), `inet` / `cidr` (IP addresses), `macaddr`, `tsvector` (full-text search).

---

## 7. Tables

### Create — full anatomy
```sql
CREATE TABLE app.customer (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        text        NOT NULL,
    email       text        NOT NULL UNIQUE,
    age         integer     CHECK (age >= 0),
    is_active   boolean     NOT NULL DEFAULT true,
    metadata    jsonb,
    created_at  timestamptz NOT NULL DEFAULT now()
);
```
Each column = `name  type  [constraints...]`. Constraints can be inline (above) or named at the table level (below, §8).

### Create variants
```sql
CREATE TABLE IF NOT EXISTS app.customer ( ... );

-- Copy structure (and optionally data) from another table
CREATE TABLE app.customer_backup AS TABLE app.customer;            -- structure + data
CREATE TABLE app.customer_empty  (LIKE app.customer INCLUDING ALL);-- structure only, incl. constraints/indexes

-- Temporary table (vanishes at end of session) — great for staging imports
CREATE TEMP TABLE staging ( ... );
```

### Rename / change owner / move schema
```sql
ALTER TABLE app.customer RENAME TO client;
ALTER TABLE app.customer OWNER TO dalto;
ALTER TABLE app.customer SET SCHEMA archive;
```

### Remove
```sql
DROP TABLE app.customer;
DROP TABLE IF EXISTS app.customer;
DROP TABLE app.customer CASCADE;          -- also drops things that depend on it (FKs, views)
TRUNCATE app.customer;                    -- delete ALL rows fast (keeps the table)
TRUNCATE app.customer RESTART IDENTITY;   -- also reset the identity counter to 1
```

---

## 8. Constraints

Constraints enforce rules at the database level — the safest place to put them. Six kinds:

| Constraint | Enforces |
|---|---|
| `PRIMARY KEY` | unique + not null; the row's identity |
| `FOREIGN KEY` | value must exist in another table |
| `UNIQUE` | no duplicate values |
| `CHECK` | a boolean condition must hold |
| `NOT NULL` | value required |
| `DEFAULT` | value used when none supplied (technically a default, not a constraint, but lives here) |

### Inline (inside CREATE TABLE)
```sql
CREATE TABLE app.order (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id bigint NOT NULL REFERENCES app.customer (id),
    total_cents integer NOT NULL CHECK (total_cents >= 0),
    status      text NOT NULL DEFAULT 'pending'
);
```

### Named, at table level (recommended for anything you may drop later)
```sql
CREATE TABLE app.order (
    id          bigint GENERATED ALWAYS AS IDENTITY,
    customer_id bigint NOT NULL,
    total_cents integer NOT NULL,

    CONSTRAINT pk_order              PRIMARY KEY (id),
    CONSTRAINT fk_order_customer     FOREIGN KEY (customer_id) REFERENCES app.customer (id),
    CONSTRAINT chk_order_total       CHECK (total_cents >= 0),
    CONSTRAINT uq_order_customer_day UNIQUE (customer_id, total_cents)
);
```
> **Name your constraints.** `fk_order_customer` is far easier to drop/alter later than the auto-generated `order_customer_id_fkey`.

### Foreign key referential actions
Control what happens to child rows when a parent is deleted/updated:
```sql
customer_id bigint REFERENCES app.customer (id)
    ON DELETE CASCADE      -- delete children too
    ON DELETE RESTRICT     -- block the delete if children exist (default)
    ON DELETE SET NULL      -- null the reference
    ON UPDATE CASCADE       -- propagate key changes
```

### Composite (multi-column) keys
```sql
PRIMARY KEY (order_id, product_id)      -- the combination must be unique
UNIQUE (tenant_id, email)               -- email unique *within* a tenant
```

### Add / drop constraints on an existing table — see §9.

---

## 9. ALTER TABLE — Modifying Anything

This is the workhorse for evolving a schema. Every structural change to a table.

### Columns
```sql
-- ADD
ALTER TABLE app.customer ADD COLUMN phone text;
ALTER TABLE app.customer ADD COLUMN IF NOT EXISTS phone text;
ALTER TABLE app.customer ADD COLUMN status text NOT NULL DEFAULT 'active';

-- DROP
ALTER TABLE app.customer DROP COLUMN phone;
ALTER TABLE app.customer DROP COLUMN IF EXISTS phone;
ALTER TABLE app.customer DROP COLUMN phone CASCADE;   -- also drop dependents

-- RENAME
ALTER TABLE app.customer RENAME COLUMN phone TO phone_number;

-- CHANGE TYPE (USING tells Postgres how to convert existing data)
ALTER TABLE app.customer ALTER COLUMN age TYPE bigint;
ALTER TABLE app.customer ALTER COLUMN age TYPE integer USING age::integer;
ALTER TABLE app.customer ALTER COLUMN price TYPE numeric(12,2) USING price::numeric;

-- DEFAULTS
ALTER TABLE app.customer ALTER COLUMN status SET DEFAULT 'active';
ALTER TABLE app.customer ALTER COLUMN status DROP DEFAULT;

-- NULLABILITY
ALTER TABLE app.customer ALTER COLUMN email SET NOT NULL;
ALTER TABLE app.customer ALTER COLUMN email DROP NOT NULL;
```

### Constraints
```sql
-- ADD
ALTER TABLE app.order ADD CONSTRAINT fk_order_customer
    FOREIGN KEY (customer_id) REFERENCES app.customer (id);
ALTER TABLE app.customer ADD CONSTRAINT uq_customer_email UNIQUE (email);
ALTER TABLE app.customer ADD CONSTRAINT chk_age CHECK (age >= 0);
ALTER TABLE app.order ADD PRIMARY KEY (id);

-- DROP (this is why you name them)
ALTER TABLE app.order ADD CONSTRAINT fk_order_customer ... ;
ALTER TABLE app.order DROP CONSTRAINT fk_order_customer;
ALTER TABLE app.customer DROP CONSTRAINT IF EXISTS uq_customer_email;
```

### Whole table
```sql
ALTER TABLE app.customer RENAME TO client;
ALTER TABLE app.customer SET SCHEMA archive;
ALTER TABLE app.customer OWNER TO dalto;
```

---

## 10. Auto-Increment & Identity

### Use IDENTITY (modern standard)
```sql
id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY     -- DB always supplies the value
id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY -- you MAY override it
```
- `ALWAYS` → you cannot insert your own id (safest).
- `BY DEFAULT` → you can insert an explicit id when needed (e.g. data migration).

### Avoid `serial` (legacy)
`serial` / `bigserial` still work but are the old way. New code should use `IDENTITY`.

### Reset the counter
```sql
ALTER TABLE app.customer ALTER COLUMN id RESTART WITH 1000;
TRUNCATE app.customer RESTART IDENTITY;
```

---

## 11. Relationships (1:1, 1:N, M:N)

The three relationship shapes and how to model each.

### One-to-Many (the most common)
A customer has many orders. Put the foreign key on the "many" side.
```sql
CREATE TABLE app.order (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id bigint NOT NULL REFERENCES app.customer (id),  -- FK lives here
    placed_at   timestamptz NOT NULL DEFAULT now()
);
```

### Many-to-Many (via a junction table)
Orders contain many products; products appear in many orders. Create a third table.
```sql
CREATE TABLE app.order_line (
    order_id   bigint NOT NULL REFERENCES app.order (id) ON DELETE CASCADE,
    product_id bigint NOT NULL REFERENCES app.product (id),
    quantity   integer NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (order_id, product_id)        -- composite key = each pair once
);
```

### One-to-One
Rare. Put a UNIQUE foreign key on the dependent table.
```sql
CREATE TABLE app.customer_profile (
    customer_id bigint PRIMARY KEY REFERENCES app.customer (id) ON DELETE CASCADE,
    bio         text
);
-- PK on the FK column guarantees at most one profile per customer
```

### Lookup table (preferred alternative to enum)
```sql
CREATE TABLE app.order_status (
    code  text PRIMARY KEY,        -- 'pending', 'shipped', ...
    label text NOT NULL
);
INSERT INTO app.order_status (code, label)
VALUES ('pending','Pending'), ('shipped','Shipped'), ('cancelled','Cancelled');

-- reference it:
ALTER TABLE app.order
    ADD COLUMN status text NOT NULL DEFAULT 'pending'
    REFERENCES app.order_status (code);
```
Adding a new status is just an `INSERT` — no schema change. This is why lookup tables beat enums for evolving value sets.

> **Always index your foreign keys.** Postgres auto-indexes PRIMARY KEY and UNIQUE columns but **NOT** foreign keys. Unindexed FKs make joins and parent-deletes slow. See §12.

---

## 12. Indexes

Indexes speed up reads (`WHERE`, `JOIN`, `ORDER BY`) at the cost of slower writes and disk space. Index what you filter and join on.

### Create
```sql
CREATE INDEX idx_order_customer ON app.order (customer_id);     -- standard (b-tree)
CREATE UNIQUE INDEX uq_customer_email ON app.customer (email); -- enforce uniqueness
CREATE INDEX idx_order_cust_date ON app.order (customer_id, placed_at); -- multicolumn
CREATE INDEX idx_active_customers ON app.customer (id) WHERE is_active; -- partial
CREATE INDEX idx_lower_email ON app.customer (lower(email));    -- expression index
```

### Build without locking the table (for large/production tables)
```sql
CREATE INDEX CONCURRENTLY idx_order_customer ON app.order (customer_id);
```
> `CONCURRENTLY` can't run inside a transaction block, but it doesn't lock writes — use it on big live tables.

### Index types (the `USING` clause)
| Type | Use for |
|---|---|
| `btree` | **default** — equality and range (`=`, `<`, `>`, `BETWEEN`, sorting) |
| `gin` | `jsonb`, arrays, full-text search (containment `@>`, key existence) |
| `gist` | geometric data, ranges, nearest-neighbor |
| `brin` | very large tables with naturally ordered data (e.g. timestamps) — tiny + fast |
| `hash` | equality only (rarely worth it over btree) |

```sql
CREATE INDEX idx_meta ON app.customer USING gin (metadata);     -- index a jsonb column
```

### Remove
```sql
DROP INDEX idx_order_customer;
DROP INDEX IF EXISTS idx_order_customer;
DROP INDEX CONCURRENTLY idx_order_customer;
```

---

## 13. Views & Materialized Views

### View (a saved query — always live, stores no data)
```sql
CREATE VIEW app.active_customers AS
    SELECT id, name, email FROM app.customer WHERE is_active;

CREATE OR REPLACE VIEW app.active_customers AS    -- redefine in place
    SELECT id, name, email, created_at FROM app.customer WHERE is_active;

DROP VIEW app.active_customers;
DROP VIEW IF EXISTS app.active_customers;
```
Query it like a table: `SELECT * FROM app.active_customers;`

### Materialized View (stores the result — fast reads, must be refreshed)
```sql
CREATE MATERIALIZED VIEW app.sales_summary AS
    SELECT customer_id, count(*) AS order_count, sum(total_cents) AS total
    FROM app.order GROUP BY customer_id;

REFRESH MATERIALIZED VIEW app.sales_summary;              -- recompute (locks reads)
REFRESH MATERIALIZED VIEW CONCURRENTLY app.sales_summary;-- recompute w/o blocking reads
                                                          -- (needs a UNIQUE index on the MV)
DROP MATERIALIZED VIEW app.sales_summary;
```
Use a materialized view for expensive aggregations you read often and can tolerate being slightly stale.

---

## 14. Functions & Procedures

### Function (returns a value; usable inside queries)
```sql
CREATE OR REPLACE FUNCTION app.full_label(p_code text)
RETURNS text
LANGUAGE sql
AS $$
    SELECT label FROM app.order_status WHERE code = p_code;
$$;

SELECT app.full_label('shipped');     -- 'Shipped'
```

### Function in PL/pgSQL (procedural logic, variables, control flow)
```sql
CREATE OR REPLACE FUNCTION app.order_total(p_order_id bigint)
RETURNS integer
LANGUAGE plpgsql
AS $$
DECLARE
    v_total integer;
BEGIN
    SELECT sum(ol.quantity * p.price_cents)
      INTO v_total
      FROM app.order_line ol
      JOIN app.product p ON p.id = ol.product_id
     WHERE ol.order_id = p_order_id;

    RETURN COALESCE(v_total, 0);
END;
$$;
```

### Procedure (no return value; can manage its own transactions)
```sql
CREATE OR REPLACE PROCEDURE app.archive_old_orders(p_before timestamptz)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO archive.order SELECT * FROM app.order WHERE placed_at < p_before;
    DELETE FROM app.order WHERE placed_at < p_before;
END;
$$;

CALL app.archive_old_orders('2025-01-01');   -- procedures are invoked with CALL
```

### Remove
```sql
DROP FUNCTION app.full_label(text);          -- include arg types to disambiguate
DROP PROCEDURE app.archive_old_orders(timestamptz);
```

---

## 15. Triggers

A trigger runs a function automatically on INSERT/UPDATE/DELETE. Classic use: auto-update an `updated_at` column.

```sql
-- 1) Add the column
ALTER TABLE app.customer ADD COLUMN updated_at timestamptz NOT NULL DEFAULT now();

-- 2) Create the trigger function (must return trigger)
CREATE OR REPLACE FUNCTION app.set_updated_at()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at := now();
    RETURN NEW;
END;
$$;

-- 3) Attach it to the table
CREATE TRIGGER trg_customer_updated
    BEFORE UPDATE ON app.customer
    FOR EACH ROW
    EXECUTE FUNCTION app.set_updated_at();
```
Now any `UPDATE` to a customer row refreshes `updated_at` automatically.

```sql
DROP TRIGGER trg_customer_updated ON app.customer;
```
Timing options: `BEFORE` / `AFTER` / `INSTEAD OF`; events: `INSERT` / `UPDATE` / `DELETE`; granularity: `FOR EACH ROW` / `FOR EACH STATEMENT`.

---

## 16. Sequences

A sequence generates numbers. `IDENTITY` (§10) uses one under the hood — prefer IDENTITY. Use a raw sequence only when you need a counter shared across tables.

```sql
CREATE SEQUENCE app.invoice_no START 1000 INCREMENT 1;

SELECT nextval('app.invoice_no');   -- 1000, then 1001, ...
SELECT currval('app.invoice_no');   -- last value YOU got in this session
SELECT setval('app.invoice_no', 5000);

ALTER SEQUENCE app.invoice_no RESTART WITH 1;
DROP SEQUENCE app.invoice_no;
```

---

## 17. Generated Columns

A column computed from other columns, stored automatically.
```sql
CREATE TABLE app.line (
    quantity    integer NOT NULL,
    price_cents integer NOT NULL,
    total_cents integer GENERATED ALWAYS AS (quantity * price_cents) STORED
);
```
You never write to `total_cents`; Postgres computes and stores it. `STORED` is required (the value is physically saved).

---

## 18. Transactions

A transaction groups statements so they all succeed or all fail together.

```sql
BEGIN;
    INSERT INTO app.order (customer_id) VALUES (1);
    INSERT INTO app.order_line (order_id, product_id, quantity) VALUES (1, 10, 2);
COMMIT;        -- make it permanent
-- or:
ROLLBACK;      -- undo everything since BEGIN
```

### Savepoints (partial rollback)
```sql
BEGIN;
    INSERT INTO app.customer (name, email) VALUES ('A', 'a@x.com');
    SAVEPOINT sp1;
    INSERT INTO app.customer (name, email) VALUES ('B', 'bad');
    ROLLBACK TO sp1;        -- undo just the second insert
    INSERT INTO app.customer (name, email) VALUES ('B', 'b@x.com');
COMMIT;
```

> **DDL is transactional in PostgreSQL.** You can wrap `CREATE TABLE`, `ALTER TABLE`, `DROP`, etc. in `BEGIN ... COMMIT` and `ROLLBACK` them if something goes wrong. This is a major advantage — test schema changes safely. (Exceptions: `CREATE INDEX CONCURRENTLY`, `CREATE DATABASE`, `VACUUM` can't run inside a transaction block.)

---

## 19. Working With JSON Inside Postgres

Use `jsonb` (not `json`). Operators you'll actually use:

```sql
-- Sample column: metadata jsonb holding {"tier":"gold","tags":["a","b"],"score":42}

SELECT metadata -> 'tier'        FROM app.customer;  -- -> returns jsonb:  "gold"
SELECT metadata ->> 'tier'       FROM app.customer;  -- ->> returns text:  gold
SELECT metadata -> 'tags' -> 0   FROM app.customer;  -- array element (jsonb)
SELECT metadata #> '{tags,0}'    FROM app.customer;  -- path to nested value (jsonb)
SELECT metadata #>> '{tags,0}'   FROM app.customer;  -- path, as text

-- Filtering
SELECT * FROM app.customer WHERE metadata ->> 'tier' = 'gold';
SELECT * FROM app.customer WHERE metadata @> '{"tier":"gold"}';   -- contains
SELECT * FROM app.customer WHERE metadata ? 'tier';               -- key exists

-- Modifying jsonb
SELECT jsonb_set(metadata, '{tier}', '"platinum"') FROM app.customer;
SELECT metadata || '{"vip":true}'::jsonb FROM app.customer;       -- merge/add key
SELECT metadata - 'score' FROM app.customer;                      -- remove key
```

### Index jsonb for fast containment queries
```sql
CREATE INDEX idx_customer_meta ON app.customer USING gin (metadata);
-- speeds up @> and ? queries
```

### Expanding JSON into rows/columns (the import engine)
```sql
-- jsonb_array_elements: one row per array element
SELECT jsonb_array_elements('[{"a":1},{"a":2}]'::jsonb);

-- jsonb_to_recordset: turn a JSON array into a typed result set (KEY for imports)
SELECT *
FROM jsonb_to_recordset('[{"sku":"X1","qty":3},{"sku":"X2","qty":5}]'::jsonb)
     AS x(sku text, qty integer);
-- returns a 2-column, 2-row table you can INSERT ... SELECT from
```

---

## 20. Importing Data via JSON

This is the recommended pipeline for a tool you control that emits JSON. The strategy:

> **Land the JSON in a staging table → expand it with `jsonb_to_recordset` → INSERT into your real tables, resolving foreign keys by business keys.**

### 20.1 The Preferred JSON Format

**Emit NDJSON (newline-delimited JSON): one complete JSON object per line, one file per table.**

Why NDJSON instead of one big `[...]` array:
- Each line loads as one row — trivial, streaming, scales to millions of rows.
- A single multi-line `[...]` array is awkward to load through `COPY` (the whole document must be one value). NDJSON sidesteps every parsing problem.

**Rules for the objects your tool generates:**

1. **Flat objects.** Keys map directly to columns. Keep nesting out of per-row data where you can.
2. **Key names = column names.** Makes the import SQL a 1:1 mapping.
3. **Correct JSON types:** numbers as JSON numbers (`42`, not `"42"`), booleans as `true`/`false`, missing values as `null`.
4. **Timestamps as ISO-8601 strings** with a `Z` or offset: `"2026-06-26T14:30:00Z"`. Postgres casts these straight to `timestamptz`.
5. **Money as integer cents** (`1999`), matching your `integer` columns.
6. **Reference parents by a business/natural key** (email, SKU, code) — **not** by database IDs. Your tool doesn't know the auto-generated IDs; the import resolves them with a join.

#### Example: `product.ndjson` (one file for the product table)
```json
{"sku":"WIDGET-01","name":"Standard Widget","price_cents":1999,"is_active":true}
{"sku":"WIDGET-02","name":"Deluxe Widget","price_cents":4999,"is_active":true}
{"sku":"GADGET-07","name":"Gadget","price_cents":2999,"is_active":false}
```

#### Example: `customer.ndjson`
```json
{"email":"alice@example.com","name":"Alice","tier":"gold"}
{"email":"bob@example.com","name":"Bob","tier":"silver"}
```

#### Example: `order.ndjson` — child references parent by **email** and **sku**, not IDs
```json
{"customer_email":"alice@example.com","status":"pending","placed_at":"2026-06-20T10:00:00Z","lines":[{"sku":"WIDGET-01","qty":2},{"sku":"GADGET-07","qty":1}]}
{"customer_email":"bob@example.com","status":"shipped","placed_at":"2026-06-21T09:30:00Z","lines":[{"sku":"WIDGET-02","qty":5}]}
```
Here `lines` stays nested because order lines belong wholly to their order — we expand it during import.

### 20.2 Loading NDJSON into a staging table

Create a one-column staging table and `\copy` the file in. The custom `DELIMITER`/`QUOTE` (control characters that never appear in JSON) make `COPY` drop each whole line into the single `jsonb` column intact, **including backslash escapes**:

```sql
CREATE TEMP TABLE _load (doc jsonb);

\copy _load (doc) FROM 'product.ndjson' WITH (FORMAT csv, DELIMITER E'\x02', QUOTE E'\x01')
```
> Why CSV with control chars, not the default? Default `COPY` text format interprets backslashes (`\n`, `\"`), which corrupts JSON string escapes. CSV format doesn't touch backslashes, and the unused control-char delimiter/quote guarantee each line lands whole. This recipe is robust for any valid JSON.

(If your data is guaranteed to contain no backslashes and no tabs, plain `\copy _load (doc) FROM 'file.ndjson'` also works — but the recipe above always works.)

### 20.3 Insert into a flat table

```sql
-- product.ndjson is now in _load. Expand and insert.
INSERT INTO app.product (sku, name, price_cents, is_active)
SELECT
    doc ->> 'sku',
    doc ->> 'name',
    (doc ->> 'price_cents')::integer,
    (doc ->> 'is_active')::boolean
FROM _load
ON CONFLICT (sku) DO NOTHING;          -- idempotent: re-running won't duplicate
```
`ON CONFLICT` requires a UNIQUE constraint on the conflict column (here `sku`). Variants:
```sql
ON CONFLICT (sku) DO NOTHING;                                   -- skip duplicates
ON CONFLICT (sku) DO UPDATE SET name = EXCLUDED.name,           -- "upsert": update instead
                                price_cents = EXCLUDED.price_cents;
```

### 20.4 Insert resolving a foreign key by business key

Load `customer.ndjson` first (same pattern as products), then load orders, joining on email to get the real `customer_id`:

```sql
-- _load now holds order.ndjson rows
INSERT INTO app.order (customer_id, status, placed_at)
SELECT
    c.id,                                   -- resolved FK
    doc ->> 'status',
    (doc ->> 'placed_at')::timestamptz
FROM _load
JOIN app.customer c ON c.email = doc ->> 'customer_email';
```

### 20.5 Expand a nested child array (`lines`) into its own table

This handles the nested `lines` inside each order. We expand the array, then join twice — to the order and to the product — to resolve both foreign keys:

```sql
INSERT INTO app.order_line (order_id, product_id, quantity)
SELECT
    o.id,                                   -- resolved order FK
    p.id,                                   -- resolved product FK
    (line ->> 'qty')::integer
FROM _load
-- pull each element of the "lines" array out as its own row:
CROSS JOIN LATERAL jsonb_array_elements(doc -> 'lines') AS line
-- match the parent order (here, by customer + timestamp; use a real key in practice):
JOIN app.customer c ON c.email = doc ->> 'customer_email'
JOIN app.order    o ON o.customer_id = c.id
                   AND o.placed_at = (doc ->> 'placed_at')::timestamptz
-- match the product by sku:
JOIN app.product  p ON p.sku = line ->> 'sku';
```
> `jsonb_array_elements(doc -> 'lines')` turns the nested array into rows; `LATERAL` lets that expansion reference the current `doc`. This is the core technique for flattening nested JSON into normalized tables.

### 20.6 Alternative: typed expansion with `jsonb_to_recordset`

If you prefer declaring the shape up front (cleaner for flat data):
```sql
INSERT INTO app.product (sku, name, price_cents, is_active)
SELECT r.sku, r.name, r.price_cents, r.is_active
FROM _load,
     LATERAL jsonb_to_recordset(
         CASE WHEN jsonb_typeof(doc) = 'array' THEN doc ELSE jsonb_build_array(doc) END
     ) AS r(sku text, name text, price_cents integer, is_active boolean);
```

### 20.7 Whole pipeline, start to finish

```sql
BEGIN;

-- 1. Products
CREATE TEMP TABLE _load (doc jsonb) ON COMMIT DROP;
\copy _load (doc) FROM 'product.ndjson' WITH (FORMAT csv, DELIMITER E'\x02', QUOTE E'\x01')
INSERT INTO app.product (sku, name, price_cents, is_active)
SELECT doc->>'sku', doc->>'name', (doc->>'price_cents')::int, (doc->>'is_active')::bool
FROM _load ON CONFLICT (sku) DO NOTHING;
DELETE FROM _load;            -- reuse the staging table

-- 2. Customers
\copy _load (doc) FROM 'customer.ndjson' WITH (FORMAT csv, DELIMITER E'\x02', QUOTE E'\x01')
INSERT INTO app.customer (email, name)
SELECT doc->>'email', doc->>'name'
FROM _load ON CONFLICT (email) DO NOTHING;
DELETE FROM _load;

-- 3. Orders
\copy _load (doc) FROM 'order.ndjson' WITH (FORMAT csv, DELIMITER E'\x02', QUOTE E'\x01')
INSERT INTO app.order (customer_id, status, placed_at)
SELECT c.id, doc->>'status', (doc->>'placed_at')::timestamptz
FROM _load JOIN app.customer c ON c.email = doc->>'customer_email';

-- 4. Order lines (nested)
INSERT INTO app.order_line (order_id, product_id, quantity)
SELECT o.id, p.id, (line->>'qty')::int
FROM _load
CROSS JOIN LATERAL jsonb_array_elements(doc->'lines') AS line
JOIN app.customer c ON c.email = doc->>'customer_email'
JOIN app.order    o ON o.customer_id = c.id AND o.placed_at = (doc->>'placed_at')::timestamptz
JOIN app.product  p ON p.sku = line->>'sku';

COMMIT;
```
Because it's all in one transaction, any failure rolls the entire import back — you never end up half-loaded.

### 20.8 Spec sheet for your JSON-generating tool

Hand this to the tool you're building later:

- **Format:** NDJSON — one JSON object per line, UTF-8, `\n` line endings. One file per target table.
- **Object keys:** exactly match destination column names.
- **Types:** JSON numbers for numerics, `true`/`false` for booleans, `null` for missing, ISO-8601 strings for timestamps.
- **Money:** integer cents.
- **Foreign references:** by stable business key (email, SKU, code), never by surrogate ID.
- **Nested children** (e.g. order lines): embed as a JSON array on the parent object under a predictable key.
- **Provide a stable business key on every parent** so children can be re-linked on import without surrogate IDs. (In the example above, matching orders on `placed_at` is illustrative; give each order its own natural key — e.g. an `order_ref` — for reliable linking.)

---

## 21. Importing Data via CSV

For purely tabular data, CSV via `COPY` is the fastest path (an order of magnitude faster than row-by-row `INSERT`).

```sql
-- \copy runs client-side (file on YOUR machine) — use this from psql
\copy app.product (sku, name, price_cents) FROM 'products.csv' WITH (FORMAT csv, HEADER true)

-- COPY runs server-side (file must be on the DB server, needs privileges)
COPY app.product (sku, name, price_cents) FROM '/srv/data/products.csv' WITH (FORMAT csv, HEADER true);
```
Useful options: `DELIMITER ','`, `HEADER true`, `NULL ''` (treat empty as NULL), `QUOTE '"'`, `ENCODING 'UTF8'`.

Export the same way:
```sql
\copy app.product TO 'out.csv' WITH (FORMAT csv, HEADER true)
\copy (SELECT * FROM app.order WHERE status='shipped') TO 'shipped.csv' WITH (FORMAT csv, HEADER true)
```

---

## 22. Inspecting the Database (Introspection)

### Via psql meta-commands
```text
\dn               schemas
\dt app.*         tables in schema app
\d app.customer   full table description
\d+ app.customer  + storage, comments
\di               indexes
\dv               views
\df app.*         functions in schema app
\du               roles
```

### Via SQL (programmatic / from any client)
```sql
-- all columns of a table
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'app' AND table_name = 'customer'
ORDER BY ordinal_position;

-- all tables in a schema
SELECT table_name FROM information_schema.tables WHERE table_schema = 'app';

-- foreign keys
SELECT conname, conrelid::regclass AS table, confrelid::regclass AS references
FROM pg_constraint WHERE contype = 'f';

-- table size on disk
SELECT pg_size_pretty(pg_total_relation_size('app.customer'));
```

---

## 23. Dropping & Cleanup Reference

| Goal | Command |
|---|---|
| Delete all rows, keep table | `TRUNCATE app.customer;` |
| Delete all rows + reset IDs | `TRUNCATE app.customer RESTART IDENTITY;` |
| Truncate table referenced by FKs | `TRUNCATE app.customer CASCADE;` |
| Drop a table | `DROP TABLE app.customer;` |
| Drop a table + dependents | `DROP TABLE app.customer CASCADE;` |
| Drop a column | `ALTER TABLE app.customer DROP COLUMN x;` |
| Drop a constraint | `ALTER TABLE app.customer DROP CONSTRAINT name;` |
| Drop an index | `DROP INDEX idx_name;` |
| Drop a schema + all contents | `DROP SCHEMA app CASCADE;` |
| Drop a database | `DROP DATABASE appdb;` (must be disconnected from it) |
| Safe-guard any drop | add `IF EXISTS` |

**`CASCADE` vs `RESTRICT`:** `RESTRICT` (default) blocks the drop if anything depends on the object; `CASCADE` drops the dependents too. Use `CASCADE` deliberately.

### Development reset pattern (rebuild schema from scratch)
Put this at the top of your `schema.sql` so every run gives a clean slate:
```sql
DROP SCHEMA IF EXISTS app CASCADE;
CREATE SCHEMA app;
-- ... all your CREATE TABLE statements follow ...
```

---

## 24. Conventions & Best Practices

A condensed rule set — the defaults that keep a schema clean and professional.

**Schema as code**
- Keep your whole schema in a version-controlled `.sql` file. It's the source of truth — never build a real schema by clicking through a GUI.
- Use a GUI (DBeaver) for *exploring, diagramming, and debugging* — not for *authoring*.

**Naming**
- Tables and columns: `snake_case`, lowercase. (Unquoted identifiers are folded to lowercase, so mixed case causes pain.)
- Table names: singular (`customer`, not `customers`) — pick one convention and hold it.
- **Name your constraints and indexes** (`fk_order_customer`, `idx_order_customer`) so you can drop/alter them later.

**Types**
- `text` for strings (not `varchar(n)` unless a real cap is needed).
- `timestamptz` for all timestamps (not `timestamp`).
- Integer cents or `numeric` for money — never float.
- `bigint GENERATED ALWAYS AS IDENTITY` for surrogate keys (not `serial`).
- `jsonb` (not `json`) for JSON.

**Integrity**
- Push rules into the database: `NOT NULL`, `CHECK`, `UNIQUE`, foreign keys. Don't rely on application code to enforce data shape.
- **Index every foreign key** you join or filter on — Postgres won't do it for you.
- Use transactions for multi-step writes so they succeed or fail as a unit.

**Imports**
- Bulk tabular data → `COPY` from CSV.
- Structured/nested data from your own tool → NDJSON, staged in a `jsonb` table, expanded with `jsonb_to_recordset` / `jsonb_array_elements`.
- Reference parents by business keys in exported JSON; resolve to surrogate IDs on import.
- Make imports idempotent with `ON CONFLICT ... DO NOTHING/UPDATE`.

**Organization**
- Group a tool's objects under a dedicated schema (`app`), keep import scratch tables in `staging`, keep `public` clean.

---

*End of reference.*
