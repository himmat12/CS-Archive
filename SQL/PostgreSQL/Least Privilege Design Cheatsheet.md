# PostgreSQL Least‑Privilege Database Design Cheatsheet

General template for a single‑database application with separate schemas for tables and business logic, using three application roles: owner, developer, and application user.

Replace placeholders like `<db_name>`, `<owner_role>`, `<dev_role>`, `<app_role>`, `<data_schema>`, and `<logic_schema>` with your actual names.

---

## 1. Two Layers of Permissions

| Layer | Scope | Examples | Purpose |
|-------|-------|----------|---------|
| **Role attributes** | Cluster‑wide (server level) | `SUPERUSER`, `CREATEDB`, `CREATEROLE`, `LOGIN`, `REPLICATION`, `BYPASSRLS` | Define powerful capabilities a role has in general. |
| **Object privileges** | Specific objects (cluster, database, schema, table, function, etc.) | `CONNECT`, `CREATE`, `TEMPORARY`, `USAGE`, `SELECT`, `INSERT`, `UPDATE`, `EXECUTE` | Define what a role can do on each specific object. |

**Key principle:**  
- Keep role attributes minimal (almost always just `LOGIN` for app/dev roles).  
- Enforce restrictions via object privileges. [web:16][web:23][web:25]

---

## 2. Role Design (Template)

| Role | Intended Use | Recommended Attributes | Notes |
|------|--------------|------------------------|-------|
| `postgres` | Cluster superuser (admin only) | `SUPERUSER`, `CREATEROLE`, `CREATEDB`, … | Never use for application connections. Harden password, restrict network access. [web:44][web:46][web:48][web:50] |
| `<owner_role>` | Database + schema owner, migrations, DDL | `LOGIN`, `NOSUPERUSER`, `NOCREATEDB`, `NOCREATEROLE` | Owns `<db_name>`, `<data_schema>`, `<logic_schema>`. Grants privileges to other roles. |
| `<dev_role>` | Interactive development | `LOGIN`, `NOSUPERUSER`, `NOCREATEDB`, `NOCREATEROLE` | Full privileges on `<data_schema>` and `<logic_schema>` for development. |
| `<app_role>` | Application runtime | `LOGIN`, `NOSUPERUSER`, `NOCREATEDB`, `NOCREATEROLE` | Least privilege: only what the app needs (connect, execute procedures, maybe some DML). |

**Never** give `SUPERUSER`, `CREATEDB`, or `CREATEROLE` to application or dev roles unless absolutely necessary. [web:48][web:50][web:56]

---

## 3. Database‑Level Privileges (Object Layer)

### Default Behavior

When you run:

```sql
CREATE DATABASE <db_name>;
```

PostgreSQL grants default privileges:

- `PUBLIC` (everyone) can `CONNECT`.
- `PUBLIC` can create `TEMP` tables. [web:30][web:42]

Visible as:

```text
=Tc/<owner_role>   -- public has TEMP (T) and CONNECT (c)
```

### Least‑Privilege Pattern

1. **Revoke defaults from PUBLIC**

   ```sql
   REVOKE CONNECT   ON DATABASE <db_name> FROM PUBLIC;
   REVOKE TEMPORARY ON DATABASE <db_name> FROM PUBLIC;
   ```

2. **Grant explicitly to allowed roles**

   ```sql
   -- CONNECT
   GRANT CONNECT ON DATABASE <db_name> TO <owner_role>;
   GRANT CONNECT ON DATABASE <db_name> TO <dev_role>;
   GRANT CONNECT ON DATABASE <db_name> TO <app_role>;

   -- TEMPORARY (temp tables)
   GRANT TEMPORARY ON DATABASE <db_name> TO <owner_role>;
   GRANT TEMPORARY ON DATABASE <db_name> TO <dev_role>;
   GRANT TEMPORARY ON DATABASE <db_name> TO <app_role>;
   ```

3. **Verify**

   ```sql
   \l
   ```

   Expected pattern:

   ```text
   <db_name> | <owner_role> | ... | <owner_role>=CTc/<owner_role> +
             |              |     | <dev_role>=cT/<owner_role>    +
             |              |     | <app_role>=cT/<owner_role>
   ```

   No `=...` (public) line.

### Checklist: Database Level

- [ ] `PUBLIC` cannot `CONNECT` to the database.
- [ ] `PUBLIC` cannot create `TEMP` tables.
- [ ] Only intended roles (`<owner_role>`, `<dev_role>`, `<app_role>`) have `CONNECT`.
- [ ] Only roles that need temp tables have `TEMPORARY`.

---

## 4. Schema‑Level Privileges

### Typical Schema Design

- `<data_schema>` – all tables.
- `<logic_schema>` – all stored procedures / functions.
- `public` – left untouched or restricted.

### Typical Grants

```sql
-- Allow using schemas
GRANT USAGE ON SCHEMA <data_schema>  TO <owner_role>;
GRANT USAGE ON SCHEMA <logic_schema> TO <owner_role>;

GRANT USAGE ON SCHEMA <data_schema>  TO <dev_role>;
GRANT USAGE ON SCHEMA <logic_schema> TO <dev_role>;

GRANT USAGE ON SCHEMA <logic_schema> TO <app_role>;
-- (Optionally USAGE on <data_schema> if app must read tables directly)

-- Allow creating objects in schemas (owner + dev)
GRANT CREATE ON SCHEMA <data_schema>  TO <owner_role>;
GRANT CREATE ON SCHEMA <logic_schema> TO <owner_role>;

GRANT CREATE ON SCHEMA <data_schema>  TO <dev_role>;
GRANT CREATE ON SCHEMA <logic_schema> TO <dev_role>;

-- Do NOT grant CREATE on schemas to <app_role>
```

Verify:

```sql
\dn+
```

You should see entries like:

```text
<data_schema>  | <owner_role> | <owner_role>=UC/<owner_role> +
               |              | <dev_role>=UC/<owner_role>
<logic_schema> | <owner_role> | <owner_role>=UC/<owner_role> +
               |              | <dev_role>=UC/<owner_role>   +
               |              | <app_role>=U/<owner_role>
```

### Checklist: Schema Level

- [ ] `<app_role>` has `USAGE` only on required schemas (e.g. `<logic_schema>`).
- [ ] `<app_role>` does **not** have `CREATE` on any schema.
- [ ] `<dev_role>` and `<owner_role>` have `USAGE` + `CREATE` on `<data_schema>` and `<logic_schema>`.
- [ ] `public` schema is not unnecessarily open (consider revoking `CREATE` from `PUBLIC` on `public`).

---

## 5. Object‑Level Privileges (Tables, Functions, Procedures)

### Tables (in `<data_schema>`)

```sql
-- For <dev_role> (full control)
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA <data_schema> TO <dev_role>;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA <data_schema> TO <dev_role>;

-- For <app_role> (least privilege)
-- Option A: read/write on specific tables
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE <data_schema>.some_table TO <app_role>;

-- Option B: no direct table access (only via procedures)
-- → grant nothing on tables to <app_role>
```

### Functions / Procedures (in `<logic_schema>`)

```sql
-- All functions/procedures in <logic_schema> (common for app role)
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA <logic_schema> TO <app_role>;
GRANT ALL PRIVILEGES ON ALL PROCEDURES IN SCHEMA <logic_schema> TO <app_role>;

-- Or more restrictive: specific objects only
GRANT EXECUTE ON FUNCTION <logic_schema>.some_func(text) TO <app_role>;
GRANT EXECUTE ON PROCEDURE <logic_schema>.some_proc(text, int) TO <app_role>;
```

Remember: `EXECUTE` is **not** a schema privilege; it applies to functions/procedures. [web:1][web:6][web:12]

### Default Privileges for Future Objects

Run as the role that will create objects (often `<owner_role>` or `<dev_role>`):

```sql
-- Future tables/sequences in <data_schema>
ALTER DEFAULT PRIVILEGES IN SCHEMA <data_schema>
    GRANT ALL PRIVILEGES ON TABLES TO <dev_role>;
ALTER DEFAULT PRIVILEGES IN SCHEMA <data_schema>
    GRANT ALL PRIVILEGES ON SEQUENCES TO <dev_role>;

-- Future functions/procedures in <logic_schema>
ALTER DEFAULT PRIVILEGES IN SCHEMA <logic_schema>
    GRANT ALL PRIVILEGES ON FUNCTIONS TO <dev_role>;
ALTER DEFAULT PRIVILEGES IN SCHEMA <logic_schema>
    GRANT ALL PRIVILEGES ON PROCEDURES TO <dev_role>;

-- For <app_role> (app) on <logic_schema>
ALTER DEFAULT PRIVILEGES IN SCHEMA <logic_schema>
    GRANT ALL PRIVILEGES ON FUNCTIONS TO <app_role>;
ALTER DEFAULT PRIVILEGES IN SCHEMA <logic_schema>
    GRANT ALL PRIVILEGES ON PROCEDURES TO <app_role>;
```

If another role creates objects, specify it:

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE <owner_role> IN SCHEMA <logic_schema>
    GRANT EXECUTE ON FUNCTIONS TO <app_role>;
```

### Checklist: Object Level

- [ ] `<app_role>` has only the minimum required privileges on tables (often none).
- [ ] `<app_role>` has `EXECUTE` only on required functions/procedures.
- [ ] `<dev_role>` has full privileges on `<data_schema>` and `<logic_schema>` objects for development.
- [ ] Default privileges are set so new objects automatically get the right grants.

---

## 6. Superuser and Admin Strategy

### `postgres` Superuser

- Keep as the **only** superuser in the cluster.
- Use only for:
  - Initial setup (extensions requiring superuser, cluster config).
  - Emergency admin tasks.
- Never:
  - Use in application connection strings.
  - Run routine scripts or deployments as `postgres`. [web:44][web:46][web:48][web:50][web:56]

Harden `postgres`:

- Strong, randomly generated password stored in a secrets manager.
- Restrict `pg_hba.conf` to trusted IPs / localhost only.
- Require SSL and strong authentication (e.g. `scram-sha-256`). [web:46][web:47][web:52]

### `<owner_role>` Role

- Not a superuser.
- Owner of:
  - Database `<db_name>`.
  - Schemas `<data_schema>` and `<logic_schema>`.
- Responsible for:
  - DDL/migrations.
  - Granting/revoking privileges for `<dev_role>` and `<app_role>`.

This gives you admin control without cluster‑wide superuser powers. [web:48][web:50][web:57]

---

## 7. Architecture Decisions & Considerations

### Separating Tables and Procedures into Different Schemas

Template design:

- `<data_schema>` – tables.
- `<logic_schema>` – stored procedures / functions.

**Pros:**

- Clear separation between data model and API / business logic.
- Easier to enforce “app only calls procedures, never touches tables directly”.
- Different permission boundaries for different roles. [web:32][web:39][web:40]

**Cons / trade‑offs:**

- Slightly more complex privilege management (cross‑schema).
- Not strictly required by PostgreSQL; many projects use one schema per domain (e.g. `auth`, `billing`) with tables and functions together. [web:30][web:41][web:43]

**When it’s especially useful:**

- You want a procedural API layer where all writes go through procedures.
- You anticipate internal table changes but stable procedure signatures.
- You want to restrict some roles to “logic only” with no direct table access. [web:32][web:39]

### Temp Tables and Complex Queries

- Temp tables are as useful in PostgreSQL as in SQL Server for complex queries and intermediate results. [web:30][web:35]
- Grant `TEMPORARY` on the database to roles that need it (`<owner_role>`, `<dev_role>`, `<app_role>`).
- Revoke `TEMPORARY` from `PUBLIC` to follow least privilege. [web:48][web:50]

### Multi‑Environment Considerations

Apply the same pattern across environments (dev, test, prod):

- Same roles (`<owner_role>`, `<dev_role>`, `<app_role>`).
- Same schema structure (`<data_schema>`, `<logic_schema>`).
- Same privilege model, only differing in:
  - Which roles exist.
  - How tightly you restrict `<app_role>` in prod vs dev.

This makes migrations and reasoning about permissions consistent.

---

## 8. Quick Reference Commands

```sql
-- Inspect roles and attributes
\du
\du+

-- Inspect databases and privileges
\l
\l+

-- Inspect schemas and privileges
\dn+

-- Inspect table privileges
\dp schema.table

-- Inspect function/procedure privileges
\df+ schema.*

-- Revoke public defaults on a new database
REVOKE CONNECT   ON DATABASE <db_name> FROM PUBLIC;
REVOKE TEMPORARY ON DATABASE <db_name> FROM PUBLIC;

-- Grant database privileges
GRANT CONNECT   ON DATABASE <db_name> TO <owner_role>;
GRANT TEMPORARY ON DATABASE <db_name> TO <owner_role>;
GRANT CONNECT   ON DATABASE <db_name> TO <dev_role>;
GRANT CONNECT   ON DATABASE <db_name> TO <app_role>;

-- Grant schema privileges
GRANT USAGE  ON SCHEMA <data_schema>  TO <app_role>;
GRANT CREATE ON SCHEMA <data_schema>  TO <dev_role>;
GRANT USAGE  ON SCHEMA <logic_schema> TO <app_role>;

-- Grant object privileges
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE <data_schema>.orders TO <app_role>;
GRANT EXECUTE ON FUNCTION <logic_schema>.do_something(text) TO <app_role>;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA <data_schema> TO <dev_role>;

-- Default privileges
ALTER DEFAULT PRIVILEGES IN SCHEMA <data_schema>
    GRANT ALL PRIVILEGES ON TABLES TO <dev_role>;
ALTER DEFAULT PRIVILEGES IN SCHEMA <logic_schema>
    GRANT ALL PRIVILEGES ON FUNCTIONS TO <app_role>;
```

---

## 9. Minimal “Harden This Database” Script Template

Adapt this for each new database:

```sql
-- Run as postgres or cluster admin
\c postgres postgres

-- Assume DB already exists: <db_name>, owner <owner_role>
REVOKE CONNECT   ON DATABASE <db_name> FROM PUBLIC;
REVOKE TEMPORARY ON DATABASE <db_name> FROM PUBLIC;

GRANT CONNECT   ON DATABASE <db_name> TO <owner_role>;
GRANT TEMPORARY ON DATABASE <db_name> TO <owner_role>;
GRANT CONNECT   ON DATABASE <db_name> TO <dev_role>;
GRANT TEMPORARY ON DATABASE <db_name> TO <dev_role>;
GRANT CONNECT   ON DATABASE <db_name> TO <app_role>;
GRANT TEMPORARY ON DATABASE <db_name> TO <app_role>;

\c <db_name> <owner_role>

-- Schemas
CREATE SCHEMA <data_schema>;
CREATE SCHEMA <logic_schema>;

ALTER SCHEMA <data_schema>  OWNER TO <owner_role>;
ALTER SCHEMA <logic_schema> OWNER TO <owner_role>;

-- Schema privileges
GRANT USAGE, CREATE ON SCHEMA <data_schema>  TO <owner_role>;
GRANT USAGE, CREATE ON SCHEMA <logic_schema> TO <owner_role>;

GRANT USAGE, CREATE ON SCHEMA <data_schema>  TO <dev_role>;
GRANT USAGE, CREATE ON SCHEMA <logic_schema> TO <dev_role>;

GRANT USAGE ON SCHEMA <logic_schema> TO <app_role>;
-- (Optionally USAGE on <data_schema> if app needs direct table access)

-- Default privileges (run as <owner_role> or the main DDL role)
ALTER DEFAULT PRIVILEGES IN SCHEMA <data_schema>
    GRANT ALL PRIVILEGES ON TABLES TO <dev_role>;
ALTER DEFAULT PRIVILEGES IN SCHEMA <data_schema>
    GRANT ALL PRIVILEGES ON SEQUENCES TO <dev_role>;

ALTER DEFAULT PRIVILEGES IN SCHEMA <logic_schema>
    GRANT ALL PRIVILEGES ON FUNCTIONS TO <dev_role>;
ALTER DEFAULT PRIVILEGES IN SCHEMA <logic_schema>
    GRANT ALL PRIVILEGES ON PROCEDURES TO <dev_role>;

ALTER DEFAULT PRIVILEGES IN SCHEMA <logic_schema>
    GRANT ALL PRIVILEGES ON FUNCTIONS TO <app_role>;
ALTER DEFAULT PRIVILEGES IN SCHEMA <logic_schema>
    GRANT ALL PRIVILEGES ON PROCEDURES TO <app_role>;
```

---

## 10. Final Checklist for Your Design

- [ ] `postgres` is the only superuser; hardened and not used by apps.
- [ ] `<owner_role>` is not a superuser; owns `<db_name>`, `<data_schema>`, `<logic_schema>`.
- [ ] `<dev_role>` and `<app_role>` have only `LOGIN` as attributes.
- [ ] `PUBLIC` cannot `CONNECT` or create `TEMP` tables on `<db_name>`.
- [ ] Only `<owner_role>`, `<dev_role>`, `<app_role>` have `CONNECT` and `TEMPORARY`.
- [ ] `<app_role>` has:
  - `USAGE` on `<logic_schema>` (and `<data_schema>` only if needed).
  - `EXECUTE` on required procedures/functions.
  - Minimal or no direct table privileges.
- [ ] `<dev_role>` has full privileges on `<data_schema>` and `<logic_schema>` for development.
- [ ] Default privileges are set so new objects inherit correct grants.
- [ ] Schema design (`<data_schema>` vs `<logic_schema>`) matches your security and architectural goals.

Use this as a reference when designing new databases or auditing existing ones.