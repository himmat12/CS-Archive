# psql Cheatsheet

Quick reference for PostgreSQL's interactive terminal `psql`. Covers connection, meta‑commands, common SQL, roles, performance, and scripting.

---

## Connecting to PostgreSQL

```bash
# Basic connection
psql -U username -d dbname -h host -p port

# Common patterns
psql -U postgres                      # local, default DB, default user
psql -U himmat -d mydb                # local, specific DB
psql -U himmat -d mydb -h localhost   # explicit localhost (TCP)
psql -U himmat -d mydb -h db.example.com -p 5432

# Connection string
psql "postgresql://himmatt:password@host:5432/mydb"

# From inside psql
\c dbname                 # connect to another DB (same user/host)
\c dbname user host port  # full reconnect
\conninfo                 # show current connection info
```

---

## Essential Meta‑Commands (Backslash Commands)

Anything starting with `\` is handled by `psql` itself.

### Databases & Schemas

```sql
\l              -- list all databases
\l+             -- list databases with size & description
\c dbname       -- connect to a database
\dn             -- list schemas
\dn+            -- schemas with description & owner
```

### Tables, Views, Indexes, Sequences

```sql
\dt             -- list tables in current schema
\dt+            -- tables with size & description
\dt schema.*    -- list tables in a specific schema
\dt *.*         -- list tables in all schemas

\dv             -- list views
\dv+            -- views with description

\di             -- list indexes
\di+            -- indexes with size & description

\ds             -- list sequences
\ds+            -- sequences with details

\d  tablename   -- describe a table (columns, types, constraints, indexes)
\d+ tablename   -- verbose table description (storage, stats, comments)
\d  schema.tablename
```

### Roles & Privileges

```sql
\du             -- list all roles (users) and attributes
\du+            -- roles with more detail (members, replication, etc.)
\dp             -- list table access privileges (GRANTs) for current schema
\dp tablename   -- privileges for a specific table
```

### Functions, Extensions, Types

```sql
\df             -- list functions
\df+            -- functions with full details
\sf funcname    -- show function source code

\dx             -- list installed extensions
\dx+ extname    -- details of a specific extension

\dT             -- list data types
\dT+            -- types with details
```

### Output & Formatting

```sql
\x              -- toggle expanded output (vertical layout)
\x auto         -- expanded only when row is too wide
\timing         -- toggle query execution time display
\pset null '∅'  -- show NULLs as a visible symbol
\pset pager off -- disable pager (less) for output
\o filename     -- redirect output to a file
\o              -- stop redirecting output (back to screen)
```

### Editing & Running SQL

```sql
\e              -- open current query buffer in $EDITOR
\i filename.sql -- run SQL from a file (absolute or relative to cwd)
\ir filename.sql-- run SQL file relative to current script directory
\g              -- execute current query buffer
\g filename     -- execute and save result to file
\watch 5        -- re-run last query every 5 seconds
\! command      -- run a shell command (e.g. \! dir)
```

### Help & Info

```sql
\?              -- help on all backslash commands
\h              -- help on SQL commands (e.g. \h SELECT)
\h CREATE TABLE -- detailed help for a specific command
\echo text      -- print text to output
\version        -- show psql and server version
\copyright      -- PostgreSQL copyright info
```

### Session Control

```sql
\q              -- quit psql
\password       -- change password for current role
\password rolename -- change password for a specific role
```

---

## Common SQL: DDL (Schema Changes)

### Databases

```sql
CREATE DATABASE mydb;
DROP DATABASE mydb;

-- List DBs via SQL
SELECT datname FROM pg_database;
```

### Tables

```sql
CREATE TABLE users (
    id         SERIAL PRIMARY KEY,
    username   TEXT NOT NULL UNIQUE,
    email      TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Add / drop columns
ALTER TABLE users ADD COLUMN active BOOLEAN NOT NULL DEFAULT TRUE;
ALTER TABLE users DROP COLUMN email;

-- Rename table
ALTER TABLE users RENAME TO app_users;

-- Drop table
DROP TABLE app_users;
```

### Indexes

```sql
-- Simple index
CREATE INDEX idx_users_username ON users (username);

-- Unique index
CREATE UNIQUE INDEX idx_users_email ON users (email);

-- Partial index
CREATE INDEX idx_users_active ON users (username) WHERE active = TRUE;

-- Drop index
DROP INDEX idx_users_username;
```

### Constraints

```sql
-- Add foreign key
ALTER TABLE orders
ADD CONSTRAINT fk_orders_user_id
FOREIGN KEY (user_id) REFERENCES users (id);

-- Drop constraint
ALTER TABLE orders DROP CONSTRAINT fk_orders_user_id;
```

### Schemas

```sql
CREATE SCHEMA analytics;
SET search_path TO analytics, public;

-- Move table to another schema
ALTER TABLE users SET SCHEMA analytics;
```

---

## Common SQL: DML (Data Operations)

### Insert

```sql
INSERT INTO users (username, email)
VALUES ('himmatt', 'himmatt@example.com');

-- Insert from select
INSERT INTO archive_users
SELECT * FROM users WHERE created_at < '2024-01-01';
```

### Select

```sql
SELECT * FROM users;
SELECT id, username FROM users WHERE active = TRUE;
SELECT COUNT(*) FROM users;
SELECT DISTINCT username FROM users;

-- Joins
SELECT o.id, u.username
FROM orders o
JOIN users u ON o.user_id = u.id;

-- Aggregation
SELECT user_id, COUNT(*) AS order_count
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 5;

-- Window functions
SELECT
    id,
    username,
    ROW_NUMBER() OVER (ORDER BY created_at) AS rn
FROM users;
```

### Update

```sql
UPDATE users
SET active = FALSE
WHERE username = 'himmatt';
```

### Delete

```sql
DELETE FROM users WHERE username = 'himmatt';

-- Truncate (fast, resets serials)
TRUNCATE TABLE users RESTART IDENTITY;
```

---

## Transactions

```sql
BEGIN;

-- multiple statements
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;   -- or ROLLBACK;

-- Savepoints
BEGIN;
SAVEPOINT sp1;
-- ...
ROLLBACK TO sp1;
-- ...
COMMIT;
```

---

## Roles & Permissions

### Create / Modify Roles

```sql
-- Basic role with login
CREATE ROLE himmat WITH LOGIN PASSWORD 'strong_password';

-- Superuser (use sparingly)
ALTER ROLE himmat WITH SUPERUSER;

-- Database creation
ALTER ROLE himmat WITH CREATEDB;

-- Role creation
ALTER ROLE himmat WITH CREATEROLE;

-- Connection limit
ALTER ROLE himmat WITH CONNECTION LIMIT 10;

-- Drop role
DROP ROLE himmat;
```

### Grants & Revokes

```sql
-- Grant all on a database
GRANT ALL PRIVILEGES ON DATABASE mydb TO himmat;

-- Grant schema usage
GRANT USAGE ON SCHEMA public TO himmat;
GRANT ALL ON ALL TABLES IN SCHEMA public TO himmat;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO himmat;

-- Specific table
GRANT SELECT, INSERT, UPDATE ON users TO himmat;
GRANT ALL ON users TO himmat;

-- Revoke
REVOKE ALL ON users FROM himmat;
```

### Inspect Roles

```sql
-- List roles
\du

-- Via SQL
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolcanlogin
FROM pg_roles;

-- Who has what on a table
\dp users
```

---

## Inspecting System Catalogs (Advanced)

```sql
-- All tables in current DB
SELECT schemaname, tablename
FROM pg_tables
ORDER BY schemaname, tablename;

-- All views
SELECT schemaname, viewname
FROM pg_views;

-- Indexes
SELECT indexname, tablename
FROM pg_indexes;

-- Active sessions
SELECT pid, usename, application_name, client_addr, state, query
FROM pg_stat_activity
WHERE state <> 'idle';

-- Long-running queries
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;

-- Cancel a query
SELECT pg_cancel_backend(pid);

-- Terminate a backend
SELECT pg_terminate_backend(pid);
```

---

## Performance & Query Tuning

### EXPLAIN

```sql
EXPLAIN SELECT * FROM users WHERE username = 'himmatt';

-- With actual execution stats
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM users WHERE username = 'himmatt';
```

### Timing & Logging

Inside `psql`:

```sql
\timing          -- toggle timing
\set VERBOSITY verbose  -- more detailed errors
```

Server-side (in `postgresql.conf`):

```text
log_min_duration_statement = 1000   -- log queries > 1s
log_statement = 'all'               -- log all statements (dev only)
```

---

## Backup & Restore (CLI)

Run these from the shell, not inside `psql`.

### pg_dump

```bash
# Single database
pg_dump -U himmat -d mydb -f mydb.sql

# Custom format (good for large DBs, parallel restore)
pg_dump -U himmat -d mydb -F c -f mydb.dump

# Specific schema only
pg_dump -U himmat -d mydb -n analytics -f analytics.sql

# Specific table only
pg_dump -U himmat -d mydb -t users -f users.sql
```

### pg_restore

```bash
# From plain SQL
psql -U himmat -d mydb -f mydb.sql

# From custom format
pg_restore -U himmat -d mydb -F c mydb.dump

# List contents of dump
pg_restore -l mydb.dump
```

### Full Cluster Backup

```bash
pg_dumpall -U postgres -f all_clusters.sql
```

---

## Copying Data (CSV Import/Export)

### Server-side COPY (needs superuser or special privileges)

```sql
-- Export to CSV
COPY users TO '/tmp/users.csv' WITH (FORMAT csv, HEADER true);

-- Import from CSV
COPY users (username, email)
FROM '/tmp/users.csv'
WITH (FORMAT csv, HEADER true);
```

### Client-side `\copy` (no superuser needed, uses your client permissions)

```sql
\copy (SELECT * FROM users) TO 'users.csv' WITH (FORMAT csv, HEADER true)
\copy users (username, email) FROM 'users.csv' WITH (FORMAT csv, HEADER true)
```

---

## Scripting & Automation

### Run a SQL file

```bash
psql -U himmat -d mydb -f script.sql
```

Inside `psql`:

```sql
\i script.sql
```

### Run and time a script

```bash
psql -U himmat -d mydb -f script.sql -c '\timing on'
```

### Use environment variables

```bash
export PGHOST=localhost
export PGPORT=5432
export PGUSER=himmatt
export PGDATABASE=mydb
export PGPASSWORD='your_password'  # be careful with this

psql  # no flags needed
```

### One-liner queries

```bash
psql -U himmat -d mydb -c "SELECT COUNT(*) FROM users;"
psql -U himmat -d mydb -At -c "SELECT username FROM users;"  # plain output
```

---

## Handy Tips & Tricks

- Use `\d+ tablename` to see indexes, constraints, and storage details in one go.
- Use `\dt *.*` to list tables across all schemas.
- Use `\x auto` for nicer output when rows are wide.
- Use `\watch 5` to monitor a query (e.g., queue length, active jobs).
- Use `\timing` during development to spot slow queries quickly.
- Use `\h` to get quick SQL syntax help without leaving `psql`.
- Use `\!` to call out to your shell (e.g., `\! dir` on Windows, `\! ls` on Linux/macOS).

---

## Minimal “Survival” Subset

If you only remember a few commands:

```sql
\l          -- list databases
\c dbname   -- connect to a database
\dt         -- list tables
\d table    -- describe a table
\du         -- list roles
\q          -- quit

SELECT * FROM table LIMIT 10;
EXPLAIN (ANALYZE) SELECT ...;
```

Save this file somewhere and extend it as you learn more patterns.