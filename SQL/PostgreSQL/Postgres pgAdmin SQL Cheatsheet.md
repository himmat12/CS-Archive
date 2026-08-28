# PostgreSQL SQL Cheatsheet for pgAdmin

A comprehensive SQL reference for working with PostgreSQL in pgAdmin's Query Tool.

---

## Table of Contents

1. [pgAdmin Query Tool Basics](#pgadmin-query-tool-basics)
2. [Connection & Session](#connection--session)
3. [Database & Schema Management](#database--schema-management)
4. [Table Management](#table-management)
5. [Data Manipulation (DML)](#data-manipulation-dml)
6. [Querying Data](#querying-data)
7. [Joins](#joins)
8. [Subqueries & CTEs](#subqueries--ctes)
9. [Window Functions](#window-functions)
10. [Aggregation & Grouping](#aggregation--grouping)
11. [Data Types](#data-types)
12. [Constraints](#constraints)
13. [Indexes](#indexes)
14. [Views & Materialized Views](#views--materialized-views)
15. [Functions & Stored Procedures](#functions--stored-procedures)
16. [Triggers](#triggers)
17. [Transactions](#transactions)
18. [Permissions & Roles](#permissions--roles)
19. [System Catalog Queries](#system-catalog-queries)
20. [Performance & Query Plans](#performance--query-plans)
21. [Import & Export](#import--export)
22. [Backup & Restore Commands](#backup--restore-commands)
23. [Maintenance](#maintenance)
24. [Common Extensions](#common-extensions)
25. [SQL Server to PostgreSQL Mapping](#sql-server-to-postgresql-mapping)
26. [pgAdmin Tips & Shortcuts](#pgadmin-tips--shortcuts)

---

## pgAdmin Query Tool Basics

### Opening Query Tool

- Right-click database → **Query Tool**
- Or: Tools → **Query Tool**
- Keyboard: `Ctrl+E` (when database is selected)

### Executing Queries

- **Execute all**: `F5` or click ▶️
- **Execute selected text**: Select text → `F5`
- **Explain (visual)**: `F7` or click 📊
- **Explain Analyze**: `F8` or click 📈
- **Cancel query**: `Ctrl+C` or 🛑 button

### Results Grid

- Results appear in **Data Output** tab
- Edit cells directly (if table is updatable)
- Click **Save** to persist edits, or **Discard**
- Export: Right-click grid → **Save to File** (CSV, JSON, XML, etc.)

### Query History

- **History** tab shows previous queries in session
- Double-click to reload into editor

### Explain Plan (Visual)

- `F7` opens graphical query plan
- Shows node costs, rows, actual times (with ANALYZE)
- Hover nodes for details

---

## Connection & Session

### Connect to Server

pgAdmin handles connections via the browser tree. Once connected, open Query Tool on any database.

### Current Database & User

```sql
-- Current database
SELECT current_database();

-- Current user/role
SELECT current_user;
SELECT session_user;

-- Server version
SELECT version();
```

### Change Database (in same session)

In pgAdmin, open a new Query Tool on the target database, or reconnect via the tree. SQL-level database switching isn't supported; each Query Tool is tied to one database.

### Set Search Path

```sql
-- Set schema search order for session
SET search_path TO myschema, public;

-- View current search_path
SHOW search_path;
```

---

## Database & Schema Management

### Create Database

```sql
CREATE DATABASE mydb;

-- With owner and template
CREATE DATABASE mydb
    OWNER myuser
    TEMPLATE template0
    ENCODING 'UTF8'
    LC_COLLATE 'en_US.UTF-8'
    LC_CTYPE 'en_US.UTF-8';
```

### List Databases

```sql
SELECT datname, pg_catalog.pg_get_userbyid(datdba) AS owner, encoding
FROM pg_database
ORDER BY datname;
```

### Drop Database

```sql
DROP DATABASE mydb;

-- If exists
DROP DATABASE IF EXISTS mydb;
```

### Create Schema

```sql
CREATE SCHEMA myschema;

-- With owner
CREATE SCHEMA myschema AUTHORIZATION myuser;
```

### List Schemas

```sql
SELECT schema_name
FROM information_schema.schemata
ORDER BY schema_name;

-- Or
SELECT nspname FROM pg_namespace WHERE nspname NOT LIKE 'pg_%' ORDER BY nspname;
```

### Set Default Schema

```sql
SET search_path TO myschema, public;
```

### Drop Schema

```sql
DROP SCHEMA myschema CASCADE;  -- drops all objects in schema
```

---

## Table Management

### Create Table

```sql
CREATE TABLE users (
    id          SERIAL PRIMARY KEY,
    username    VARCHAR(50) NOT NULL UNIQUE,
    email       VARCHAR(255) NOT NULL,
    created_at  TIMESTAMP DEFAULT NOW(),
    is_active   BOOLEAN DEFAULT TRUE
);
```

### Create Table with Constraints

```sql
CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    user_id     INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    total       NUMERIC(10,2) CHECK (total >= 0),
    status      VARCHAR(20) DEFAULT 'pending',
    created_at  TIMESTAMP DEFAULT NOW()
);
```

### List Tables

```sql
-- All tables in current database
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
  AND table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY table_schema, table_name;

-- Or using pg_tables
SELECT schemaname, tablename
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename;
```

### Describe Table (Columns, Types)

```sql
-- Information schema
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = 'users'
ORDER BY ordinal_position;

-- Or with schema
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name = 'users'
ORDER BY ordinal_position;
```

### Add Column

```sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
```

### Modify Column

```sql
-- Change type
ALTER TABLE users ALTER COLUMN email TYPE TEXT;

-- Set NOT NULL
ALTER TABLE users ALTER COLUMN email SET NOT NULL;

-- Drop NOT NULL
ALTER TABLE users ALTER COLUMN email DROP NOT NULL;

-- Set default
ALTER TABLE users ALTER COLUMN is_active SET DEFAULT TRUE;

-- Drop default
ALTER TABLE users ALTER COLUMN is_active DROP DEFAULT;
```

### Drop Column

```sql
ALTER TABLE users DROP COLUMN phone;
```

### Rename Table

```sql
ALTER TABLE users RENAME TO customers;
```

### Drop Table

```sql
DROP TABLE users;

-- If exists
DROP TABLE IF EXISTS users;

-- Cascade (drop dependent objects)
DROP TABLE users CASCADE;
```

### Truncate Table (Fast Delete All)

```sql
TRUNCATE TABLE users RESTART IDENTITY;  -- resets SERIAL
TRUNCATE TABLE users, orders;           -- multiple tables
```

---

## Data Manipulation (DML)

### INSERT

```sql
-- Single row
INSERT INTO users (username, email)
VALUES ('alice', 'alice@example.com');

-- Multiple rows
INSERT INTO users (username, email)
VALUES
    ('bob', 'bob@example.com'),
    ('carol', 'carol@example.com');

-- With RETURNING
INSERT INTO users (username, email)
VALUES ('dave', 'dave@example.com')
RETURNING id, username, created_at;
```

### SELECT

```sql
-- All columns
SELECT * FROM users;

-- Specific columns
SELECT id, username, email FROM users;

-- With alias
SELECT id AS user_id, username AS name FROM users;

-- Distinct
SELECT DISTINCT status FROM orders;

-- Limit & offset
SELECT * FROM users ORDER BY created_at DESC LIMIT 10 OFFSET 20;
```

### UPDATE

```sql
-- Update all
UPDATE users SET is_active = FALSE;

-- Update with condition
UPDATE users SET is_active = TRUE WHERE username = 'alice';

-- Update multiple columns
UPDATE users
SET email = 'new@example.com', is_active = FALSE
WHERE id = 1;

-- With RETURNING
UPDATE users SET is_active = FALSE
WHERE created_at < '2024-01-01'
RETURNING id, username;
```

### DELETE

```sql
-- Delete with condition
DELETE FROM users WHERE username = 'alice';

-- Delete all (slow, use TRUNCATE for large tables)
DELETE FROM users;

-- With RETURNING
DELETE FROM users WHERE id = 5
RETURNING id, username;
```

### INSERT ... ON CONFLICT (Upsert)

```sql
-- Unique constraint on username
INSERT INTO users (username, email)
VALUES ('alice', 'newalice@example.com')
ON CONFLICT (username) DO UPDATE
SET email = EXCLUDED.email, updated_at = NOW();

-- Or do nothing
INSERT INTO users (username, email)
VALUES ('alice', 'alice@example.com')
ON CONFLICT (username) DO NOTHING;
```

---

## Querying Data

### WHERE Clause

```sql
SELECT * FROM users
WHERE is_active = TRUE
  AND created_at > '2024-01-01';

-- IN
SELECT * FROM users WHERE username IN ('alice', 'bob', 'carol');

-- LIKE (case-sensitive)
SELECT * FROM users WHERE username LIKE 'a%';

-- ILIKE (case-insensitive)
SELECT * FROM users WHERE username ILIKE 'A%';

-- BETWEEN
SELECT * FROM orders WHERE total BETWEEN 100 AND 500;

-- IS NULL / IS NOT NULL
SELECT * FROM users WHERE phone IS NULL;
SELECT * FROM users WHERE phone IS NOT NULL;
```

### ORDER BY

```sql
SELECT * FROM users
ORDER BY created_at DESC, username ASC;
```

### CASE Expression

```sql
SELECT
    username,
    CASE
        WHEN total > 1000 THEN 'high'
        WHEN total > 100 THEN 'medium'
        ELSE 'low'
    END AS customer_tier
FROM users
JOIN orders ON users.id = orders.user_id;
```

---

## Joins

### INNER JOIN

```sql
SELECT u.username, o.id AS order_id, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
```

### LEFT JOIN

```sql
SELECT u.username, o.id AS order_id
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
```

### RIGHT JOIN

```sql
SELECT u.username, o.id AS order_id
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;
```

### FULL OUTER JOIN

```sql
SELECT u.username, o.id AS order_id
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;
```

### CROSS JOIN

```sql
SELECT u.username, p.product_name
FROM users u
CROSS JOIN products p;
```

### Self Join

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

---

## Subqueries & CTEs

### Subquery in WHERE

```sql
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 1000);
```

### Subquery in SELECT

```sql
SELECT
    u.username,
    (SELECT COUNT(*) FROM orders WHERE user_id = u.id) AS order_count
FROM users u;
```

### Correlated Subquery

```sql
SELECT u.username
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.total > 1000
);
```

### CTE (Common Table Expression)

```sql
WITH high_value_orders AS (
    SELECT user_id, SUM(total) AS total_spent
    FROM orders
    GROUP BY user_id
    HAVING SUM(total) > 1000
)
SELECT u.username, h.total_spent
FROM users u
JOIN high_value_orders h ON u.id = h.user_id;
```

### Recursive CTE

```sql
WITH RECURSIVE emp_tree AS (
    -- Anchor
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive
    SELECT e.id, e.name, e.manager_id, et.level + 1
    FROM employees e
    JOIN emp_tree et ON e.manager_id = et.id
)
SELECT * FROM emp_tree ORDER BY level, name;
```

---

## Window Functions

### ROW_NUMBER, RANK, DENSE_RANK

```sql
SELECT
    username,
    total,
    ROW_NUMBER() OVER (ORDER BY total DESC) AS row_num,
    RANK() OVER (ORDER BY total DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY total DESC) AS dense_rank
FROM users
JOIN orders ON users.id = orders.user_id;
```

### PARTITION BY

```sql
SELECT
    u.username,
    o.total,
    AVG(o.total) OVER (PARTITION BY u.id) AS avg_user_total,
    SUM(o.total) OVER (PARTITION BY u.id) AS total_user_spent
FROM users u
JOIN orders o ON u.id = o.user_id;
```

### LAG / LEAD

```sql
SELECT
    username,
    created_at,
    LAG(created_at) OVER (ORDER BY created_at) AS prev_login,
    LEAD(created_at) OVER (ORDER BY created_at) AS next_login
FROM users;
```

### Running Total

```sql
SELECT
    order_date,
    total,
    SUM(total) OVER (ORDER BY order_date) AS running_total
FROM orders;
```

---

## Aggregation & Grouping

### Basic Aggregates

```sql
SELECT
    COUNT(*) AS total_users,
    COUNT(DISTINCT username) AS unique_users,
    SUM(total) AS total_revenue,
    AVG(total) AS avg_order_value,
    MIN(total) AS min_order,
    MAX(total) AS max_order
FROM orders;
```

### GROUP BY

```sql
SELECT
    status,
    COUNT(*) AS order_count,
    SUM(total) AS total_revenue
FROM orders
GROUP BY status;
```

### HAVING

```sql
SELECT
    user_id,
    COUNT(*) AS order_count
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 5;
```

### GROUPING SETS, CUBE, ROLLUP

```sql
-- Multiple groupings in one query
SELECT
    status,
    user_id,
    COUNT(*)
FROM orders
GROUP BY GROUPING SETS (
    (status),
    (user_id),
    (status, user_id),
    ()
);

-- ROLLUP (hierarchical subtotals)
SELECT status, user_id, COUNT(*)
FROM orders
GROUP BY ROLLUP (status, user_id);

-- CUBE (all combinations)
SELECT status, user_id, COUNT(*)
FROM orders
GROUP BY CUBE (status, user_id);
```

---

## Data Types

### Numeric

```sql
INTEGER, BIGINT, SMALLINT
NUMERIC(precision, scale), DECIMAL(precision, scale)
REAL, DOUBLE PRECISION
```

### Text

```sql
TEXT                -- unlimited length
VARCHAR(n)          -- max n characters
CHAR(n)             -- fixed length, padded
```

### Boolean

```sql
BOOLEAN             -- TRUE, FALSE, NULL
```

### Date/Time

```sql
DATE                -- 2024-08-28
TIME                -- 14:30:00
TIME WITH TIME ZONE
TIMESTAMP           -- 2024-08-28 14:30:00
TIMESTAMP WITH TIME ZONE  -- 2024-08-28 14:30:00+01
INTERVAL            -- '1 day', '2 hours', '3 months'
```

### JSON

```sql
JSON                -- stored as text
JSONB               -- binary, indexed, preferred
```

Example:

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    metadata JSONB
);

-- Query JSONB
SELECT * FROM products
WHERE metadata @> '{"category": "electronics"}';

-- Extract value
SELECT metadata->>'category' AS category FROM products;
```

### Arrays

```sql
INTEGER[], TEXT[], TIMESTAMP[]

-- Create
CREATE TABLE tags (
    id SERIAL,
    name TEXT,
    labels TEXT[]
);

-- Insert
INSERT INTO tags (name, labels)
VALUES ('post', ARRAY['tech', 'postgres']);

-- Query
SELECT * FROM tags WHERE 'tech' = ANY(labels);
```

### UUID

```sql
UUID

-- Generate
SELECT gen_random_uuid();  -- requires pgcrypto or use uuid_generate_v4()
```

---

## Constraints

### Primary Key

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY
);

-- Or named
CREATE TABLE users (
    id INTEGER,
    CONSTRAINT pk_users PRIMARY KEY (id)
);
```

### Foreign Key

```sql
CREATE TABLE orders (
    user_id INTEGER REFERENCES users(id)
);

-- Named with options
CREATE TABLE orders (
    user_id INTEGER,
    CONSTRAINT fk_orders_user
        FOREIGN KEY (user_id)
        REFERENCES users(id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
```

### Unique

```sql
CREATE TABLE users (
    email TEXT UNIQUE
);

-- Or named
CREATE TABLE users (
    email TEXT,
    CONSTRAINT uq_users_email UNIQUE (email)
);
```

### Check

```sql
CREATE TABLE products (
    price NUMERIC CHECK (price >= 0),
    status TEXT CHECK (status IN ('active', 'inactive'))
);
```

### Not Null

```sql
CREATE TABLE users (
    username TEXT NOT NULL
);
```

### Add Constraint to Existing Table

```sql
ALTER TABLE users ADD CONSTRAINT uq_email UNIQUE (email);
ALTER TABLE orders ADD CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id);
ALTER TABLE products ADD CONSTRAINT chk_price CHECK (price > 0);
```

### Drop Constraint

```sql
ALTER TABLE users DROP CONSTRAINT uq_email;
```

---

## Indexes

### Create Index

```sql
-- Basic
CREATE INDEX idx_users_username ON users(username);

-- Unique
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Composite
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Expression
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- Partial
CREATE INDEX idx_orders_active ON orders(created_at)
WHERE status = 'active';
```

### List Indexes

```sql
SELECT schemaname, tablename, indexname, indexdef
FROM pg_indexes
WHERE tablename = 'users'
ORDER BY indexname;
```

### Drop Index

```sql
DROP INDEX idx_users_username;
DROP INDEX IF EXISTS idx_users_username;
```

### Reindex

```sql
REINDEX TABLE users;
REINDEX INDEX idx_users_username;
REINDEX DATABASE mydb;
```

---

## Views & Materialized Views

### Create View

```sql
CREATE VIEW active_users AS
SELECT id, username, email
FROM users
WHERE is_active = TRUE;
```

### Create View with Columns

```sql
CREATE VIEW user_orders AS
SELECT
    u.id AS user_id,
    u.username,
    o.id AS order_id,
    o.total,
    o.created_at
FROM users u
JOIN orders o ON u.id = o.user_id;
```

### Drop View

```sql
DROP VIEW active_users;
DROP VIEW IF EXISTS active_users;
```

### Materialized View

```sql
CREATE MATERIALIZED VIEW mv_user_stats AS
SELECT
    u.id AS user_id,
    u.username,
    COUNT(o.id) AS order_count,
    SUM(o.total) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.username;
```

### Refresh Materialized View

```sql
REFRESH MATERIALIZED VIEW mv_user_stats;

-- Concurrently (allows reads during refresh, requires unique index)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_user_stats;
```

### Drop Materialized View

```sql
DROP MATERIALIZED VIEW mv_user_stats;
```

---

## Functions & Stored Procedures

### Create Function (SQL)

```sql
CREATE FUNCTION get_user_by_id(p_user_id INTEGER)
RETURNS TABLE(id INTEGER, username TEXT, email TEXT)
LANGUAGE SQL
AS $$
    SELECT id, username, email
    FROM users
    WHERE id = p_user_id;
$$;
```

### Create Function (PL/pgSQL)

```sql
CREATE OR REPLACE FUNCTION increment_value(p_val INTEGER)
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN p_val + 1;
END;
$$;
```

### Function with Multiple Statements

```sql
CREATE FUNCTION create_user(p_username TEXT, p_email TEXT)
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_user_id INTEGER;
BEGIN
    INSERT INTO users (username, email)
    VALUES (p_username, p_email)
    RETURNING id INTO v_user_id;

    RETURN v_user_id;
END;
$$;
```

### Call Function

```sql
SELECT * FROM get_user_by_id(1);
SELECT increment_value(5);
SELECT create_user('alice', 'alice@example.com');
```

### Drop Function

```sql
DROP FUNCTION IF EXISTS get_user_by_id(INTEGER);
DROP FUNCTION IF EXISTS increment_value(INTEGER);
```

### Stored Procedure (PostgreSQL 11+)

```sql
CREATE PROCEDURE transfer_funds(
    p_from_id INTEGER,
    p_to_id INTEGER,
    p_amount NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE accounts SET balance = balance - p_amount WHERE id = p_from_id;
    UPDATE accounts SET balance = balance + p_amount WHERE id = p_to_id;
    COMMIT;
END;
$$;

-- Call
CALL transfer_funds(1, 2, 100);
```

---

## Triggers

### Create Trigger Function

```sql
CREATE FUNCTION log_user_changes()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO user_audit (user_id, action, changed_at)
        VALUES (NEW.id, 'INSERT', NOW());
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO user_audit (user_id, action, changed_at)
        VALUES (NEW.id, 'UPDATE', NOW());
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO user_audit (user_id, action, changed_at)
        VALUES (OLD.id, 'DELETE', NOW());
    END IF;
    RETURN NULL;
END;
$$;
```

### Create Trigger

```sql
CREATE TRIGGER trg_user_changes
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION log_user_changes();
```

### Drop Trigger

```sql
DROP TRIGGER IF EXISTS trg_user_changes ON users;
```

### List Triggers

```sql
SELECT
    trigger_name,
    event_manipulation,
    event_object_table,
    action_statement
FROM information_schema.triggers
WHERE event_object_schema = 'public'
ORDER BY event_object_table, trigger_name;
```

---

## Transactions

### Basic Transaction

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;

-- Or rollback
ROLLBACK;
```

### Savepoints

```sql
BEGIN;

INSERT INTO users (username, email) VALUES ('alice', 'alice@example.com');

SAVEPOINT sp1;

INSERT INTO users (username, email) VALUES ('bob', 'bob@example.com');

-- Undo to savepoint
ROLLBACK TO sp1;

-- Continue
INSERT INTO users (username, email) VALUES ('carol', 'carol@example.com');

COMMIT;
```

### Transaction Isolation Levels

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

BEGIN ISOLATION LEVEL REPEATABLE READ;
-- ...
COMMIT;
```

---

## Permissions & Roles

### Create Role

```sql
-- Login role
CREATE ROLE app_user WITH LOGIN PASSWORD 'secret';

-- With attributes
CREATE ROLE app_user WITH
    LOGIN
    PASSWORD 'secret'
    CREATEDB
    CONNECTION LIMIT 10;

-- Superuser (use carefully)
CREATE ROLE admin WITH SUPERUSER LOGIN PASSWORD 'admin_secret';
```

### Create Group Role

```sql
CREATE ROLE devs;
CREATE ROLE readonly;
```

### Grant Membership

```sql
GRANT devs TO app_user;
GRANT readonly TO reporting_user;
```

### Grant Privileges on Database

```sql
GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT ALL ON DATABASE mydb TO admin;
```

### Grant Privileges on Schema

```sql
GRANT USAGE ON SCHEMA public TO app_user;
GRANT ALL ON SCHEMA public TO admin;
```

### Grant Privileges on Tables

```sql
-- All tables in schema
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO devs;

-- Specific table
GRANT SELECT, INSERT ON users TO app_user;

-- Future tables (default privileges)
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO readonly;
```

### Grant on Sequences

```sql
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO devs;
```

### Grant on Functions

```sql
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA public TO app_user;
```

### Revoke Privileges

```sql
REVOKE INSERT, UPDATE, DELETE ON users FROM app_user;
REVOKE ALL ON TABLE users FROM app_user;
REVOKE ALL ON SCHEMA public FROM app_user;
```

### List Roles

```sql
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolcanlogin, rolreplication
FROM pg_roles
ORDER BY rolname;

-- Or in psql: \du
```

### List Grants on Table

```sql
SELECT grantee, privilege_type
FROM information_schema.role_table_grants
WHERE table_schema = 'public'
  AND table_name = 'users';
```

### Change Owner

```sql
ALTER TABLE users OWNER TO new_owner;
ALTER DATABASE mydb OWNER TO new_owner;
```

---

## System Catalog Queries

### Databases

```sql
SELECT datname, pg_catalog.pg_get_userbyid(datdba) AS owner, encoding
FROM pg_database
ORDER BY datname;
```

### Roles

```sql
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolcanlogin
FROM pg_roles
ORDER BY rolname;
```

### Tables

```sql
SELECT schemaname, tablename
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename;
```

### Columns

```sql
SELECT table_schema, table_name, column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'public'
ORDER BY table_name, ordinal_position;
```

### Indexes

```sql
SELECT schemaname, tablename, indexname, indexdef
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY tablename, indexname;
```

### Constraints

```sql
SELECT
    conname AS constraint_name,
    contype AS constraint_type,  -- p=PK, f=FK, u=unique, c=check
    conrelid::regclass AS table_name,
    pg_get_constraintdef(oid) AS definition
FROM pg_constraint
WHERE connamespace = 'public'::regnamespace
ORDER BY conrelid::regclass, conname;
```

### Active Queries

```sql
SELECT
    pid,
    usename,
    datname,
    client_addr,
    state,
    query,
    query_start,
    NOW() - query_start AS duration
FROM pg_stat_activity
WHERE state <> 'idle'
  AND pid <> pg_backend_pid()
ORDER BY query_start;
```

### Locks

```sql
SELECT
    l.locktype,
    l.relation::regclass,
    l.mode,
    l.granted,
    a.pid,
    a.usename,
    a.query
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.relation IS NOT NULL;
```

### Table Sizes

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS index_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

---

## Performance & Query Plans

### EXPLAIN

```sql
EXPLAIN SELECT * FROM users WHERE username = 'alice';
```

### EXPLAIN ANALYZE (Executes Query)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE username = 'alice';
```

In pgAdmin: `F7` (Explain) or `F8` (Explain Analyze) for visual plan.

### Common Performance Tips

- Use `EXPLAIN ANALYZE` to see actual timings
- Look for:
  - Sequential scans on large tables (`Seq Scan`)
  - High cost nodes
  - Nested loops with large row counts
- Add indexes on:
  - WHERE columns
  - JOIN columns
  - ORDER BY columns
- Avoid:
  - Functions on indexed columns in WHERE (`WHERE LOWER(email) = ...`)
  - Leading wildcard LIKE (`LIKE '%abc'`)
  - SELECT * in production code

### Find Slow Queries (with pg_stat_statements)

```sql
-- Enable extension first (requires superuser)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top queries by total time
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

---

## Import & Export

### COPY (Server-side, requires superuser or appropriate permissions)

```sql
-- Export to CSV
COPY users TO '/path/to/users.csv' WITH (FORMAT csv, HEADER true);

-- Import from CSV
COPY users (id, username, email)
FROM '/path/to/users.csv'
WITH (FORMAT csv, HEADER true);
```

### \copy (Client-side, works in psql; in pgAdmin use Import/Export wizard)

In pgAdmin:

- Right-click table → **Import/Export Data**
- Choose file, map columns, set options (CSV, delimiter, header, etc.)

### Export Query Results to CSV (pgAdmin)

1. Run query in Query Tool
2. In **Data Output** tab, click **Download** (floppy disk icon)
3. Choose format: CSV, JSON, XML, etc.

### JSON Export

```sql
-- Single row as JSON
SELECT row_to_json(t)
FROM (SELECT id, username, email FROM users WHERE id = 1) t;

-- Multiple rows as JSON array
SELECT array_to_json(array_agg(t))
FROM (SELECT id, username, email FROM users) t;

-- Pretty JSON
SELECT json_agg(t)
FROM (SELECT id, username, email FROM users) t;
```

---

## Backup & Restore Commands

These are command-line tools, but you can run them from a terminal or script.

### pg_dump (Backup Database)

```bash
# Full database
pg_dump -U postgres -h localhost -p 5432 mydb > mydb_backup.sql

# Custom format (for pg_restore)
pg_dump -U postgres -h localhost -p 5432 -Fc mydb > mydb_backup.dump

# Schema only
pg_dump -U postgres -h localhost -p 5432 -s mydb > mydb_schema.sql

# Data only
pg_dump -U postgres -h localhost -p 5432 -a mydb > mydb_data.sql

# Specific schema
pg_dump -U postgres -h localhost -p 5432 -n myschema mydb > myschema_backup.sql

# Specific table
pg_dump -U postgres -h localhost -p 5432 -t users mydb > users_backup.sql
```

### pg_restore (Restore from Custom Dump)

```bash
# Restore to existing database
pg_restore -U postgres -h localhost -p 5432 -d mydb mydb_backup.dump

# Create new database and restore
pg_restore -U postgres -h localhost -p 5432 -C -d postgres mydb_backup.dump

# Schema only
pg_restore -U postgres -h localhost -p 5432 -d mydb -s mydb_backup.dump

# Data only
pg_restore -U postgres -h localhost -p 5432 -d mydb -a mydb_backup.dump
```

### pg_dumpall (All Databases + Roles)

```bash
pg_dumpall -U postgres -h localhost -p 5432 > all_databases.sql
```

In pgAdmin:

- Right-click server/database → **Backup...**
- Choose format (Plain, Custom, Tar, Directory)
- Set options (schema, data, blobs, etc.)
- Run

---

## Maintenance

### VACUUM

```sql
-- Basic
VACUUM users;

-- Full (reclaims disk space, exclusive lock)
VACUUM FULL users;

-- All tables
VACUUM;

-- Verbose
VACUUM VERBOSE users;
```

### ANALYZE (Update Statistics)

```sql
ANALYZE users;
ANALYZE;  -- all tables
```

### REINDEX

```sql
REINDEX TABLE users;
REINDEX INDEX idx_users_username;
REINDEX DATABASE mydb;
```

### Check Table Health

```sql
SELECT
    schemaname,
    relname AS table_name,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

---

## Common Extensions

### Enable Extension

```sql
-- Must be run in the target database
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS uuid-ossp;
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS hstore;
CREATE EXTENSION IF NOT EXISTS postgis;  -- for geospatial
```

### List Extensions

```sql
SELECT * FROM pg_extension ORDER BY extname;
```

### Drop Extension

```sql
DROP EXTENSION IF EXISTS pg_stat_statements;
```

---

## SQL Server to PostgreSQL Mapping

| SQL Server                          | PostgreSQL                                      |
|-------------------------------------|-------------------------------------------------|
| `USE database_name`                 | `\c database_name` (psql) or new Query Tool     |
| `SELECT TOP 10 * FROM t`            | `SELECT * FROM t LIMIT 10`                      |
| `ISNULL(col, val)`                  | `COALESCE(col, val)`                            |
| `GETDATE()`                         | `NOW()` or `CURRENT_TIMESTAMP`                  |
| `LEN(col)`                          | `LENGTH(col)`                                   |
| `SUBSTRING(col, 1, 5)`              | `SUBSTRING(col FROM 1 FOR 5)` or same syntax    |
| `IDENTITY(1,1)`                     | `SERIAL` or `GENERATED ALWAYS AS IDENTITY`      |
| `NVARCHAR`, `VARCHAR`               | `TEXT`, `VARCHAR(n)`                            |
| `DATETIME`, `DATETIME2`             | `TIMESTAMP`, `TIMESTAMPTZ`                      |
| `BIT`                               | `BOOLEAN`                                       |
| `UNIQUEIDENTIFIER`                  | `UUID`                                          |
| `sys.tables`                        | `pg_tables`, `information_schema.tables`        |
| `sys.columns`                       | `information_schema.columns`, `pg_attribute`    |
| `sys.databases`                     | `pg_database`                                   |
| `sys.server_principals`             | `pg_roles`                                      |
| `CREATE LOGIN`                      | `CREATE ROLE ... WITH LOGIN`                    |
| `CREATE USER`                       | `CREATE ROLE ... WITH LOGIN` (same concept)     |
| `GO` (batch separator)              | `;` (statement terminator)                      |
| `#temp` (local temp table)          | `CREATE TEMP TABLE temp_name`                   |
| `##temp` (global temp table)        | Regular table in a dedicated schema             |
| `MERGE`                             | `INSERT ... ON CONFLICT`                        |
| `OUTPUT` clause                     | `RETURNING` clause                              |
| `OPENROWSET`, `OPENDATASOURCE`      | Foreign data wrappers (`postgres_fdw`, etc.)    |
| `xp_cmdshell`                       | No direct equivalent (use external scripts)     |

---

## pgAdmin Tips & Shortcuts

### Query Editor

- **Auto-complete**: `Ctrl+Space` for suggestions (tables, columns, keywords)
- **Comment/uncomment**: `Ctrl+/`
- **Format SQL**: Right-click → **Format SQL** (if enabled) or use external formatter
- **Find/Replace**: `Ctrl+F`, `Ctrl+H`
- **Multiple cursors**: `Alt+Click` (in newer versions)

### Results Grid

- **Filter**: Click column header filter icon
- **Sort**: Click column header
- **Copy**: Select cells → `Ctrl+C`
- **Paste**: `Ctrl+V` into grid (for updatable views/tables)
- **Export**: Right-click → **Save to File**

### Explain Plan

- `F7`: Graphical explain (no execution)
- `F8`: Graphical explain analyze (executes query)
- Hover nodes for details (actual rows, time, buffers)

### Snippets

- pgAdmin has a **Snippets** panel for reusable SQL fragments
- Create common patterns (e.g., pagination, date ranges)

### Connection Management

- Save frequently used connections in **Servers** tree
- Use **Connection Properties** to set:
  - Role
  - Database
  - SSL mode
  - Connection timeout

### Dashboard

- Dashboard shows:
  - Server activity
  - Locks
  - Sessions
  - Recent queries (with pg_stat_statements)

### Debugging (PL/pgSQL)

- pgAdmin includes a debugger for functions/procedures
- Set breakpoints, step through code, inspect variables

---

## Quick Reference: Most-Used Commands

```sql
-- List tables
SELECT tablename FROM pg_tables WHERE schemaname = 'public';

-- Describe table
SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name = 'users';

-- Count rows
SELECT COUNT(*) FROM users;

-- Find duplicates
SELECT username, COUNT(*)
FROM users
GROUP BY username
HAVING COUNT(*) > 1;

-- Recent rows
SELECT * FROM users ORDER BY created_at DESC LIMIT 10;

-- Update with join
UPDATE orders o
SET status = 'completed'
FROM users u
WHERE o.user_id = u.id
  AND u.is_active = FALSE;

-- Delete with join
DELETE FROM orders o
USING users u
WHERE o.user_id = u.id
  AND u.is_active = FALSE;

-- Upsert
INSERT INTO users (username, email)
VALUES ('alice', 'alice@example.com')
ON CONFLICT (username) DO UPDATE
SET email = EXCLUDED.email;

-- Window function
SELECT
    username,
    total,
    RANK() OVER (ORDER BY total DESC) AS rank
FROM users
JOIN orders USING (id);

-- CTE
WITH recent_orders AS (
    SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '7 days'
)
SELECT COUNT(*) FROM recent_orders;

-- Index
CREATE INDEX idx_orders_created ON orders(created_at);

-- View
CREATE VIEW active_users AS
SELECT * FROM users WHERE is_active = TRUE;

-- Grant
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

-- Explain
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE username = 'alice';
```

---

## Notes

- Always test destructive commands (`DROP`, `DELETE`, `TRUNCATE`) in a non-production environment first.
- Use transactions (`BEGIN; ... COMMIT;`) for multi-step changes.
- In pgAdmin, use **Explain Analyze** (`F8`) to understand query performance before optimizing.
- For large data operations, consider:
  - Running outside peak hours
  - Using `COPY` instead of row-by-row inserts
  - Monitoring with `pg_stat_activity` and dashboards

---

End of cheatsheet.