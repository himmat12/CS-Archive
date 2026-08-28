# PostgreSQL Roles & Privileges Cheatsheet

## Two Layers of Permissions

PostgreSQL separates permissions into two distinct layers:

| Layer | Scope | Examples | Purpose |
|-------|-------|----------|---------|
| **Role Attributes** | Cluster-wide (server level) | `SUPERUSER`, `CREATEDB`, `CREATEROLE`, `LOGIN` | Define what powerful capabilities a role has in general |
| **Object Privileges** | Specific objects (database, schema, table, function, etc.) | `CONNECT`, `USAGE`, `CREATE`, `SELECT`, `UPDATE`, `EXECUTE` | Define what a role can do on each specific object |

---

## Layer 1: Role Attributes (Server Level)

Set with `CREATE ROLE` or `ALTER ROLE`. These are global flags on the role itself.

### Common Attributes

| Attribute | Description | Typical App Role Setting |
|-----------|-------------|--------------------------|
| `LOGIN` / `NOLOGIN` | Can the role connect to the database? | `LOGIN` (if it's a user) |
| `SUPERUSER` / `NOSUPERUSER` | Bypass all permission checks (cluster root) | `NOSUPERUSER` (almost always) |
| `CREATEDB` / `NOCREATEDB` | Can create new databases in the cluster | `NOCREATEDB` |
| `CREATEROLE` / `NOCREATEROLE` | Can create/alter/drop other roles | `NOCREATEROLE` |
| `REPLICATION` / `NOREPLICATION` | Can initiate replication | `NOREPLICATION` |
| `BYPASSRLS` / `NOBYPASSRLS` | Bypass row-level security policies | `NOBYPASSRLS` |
| `INHERIT` / `NOINHERIT` | Automatically inherit privileges from member roles | `INHERIT` (default) |

### Key Points

- Attributes are **cluster-wide**, not per database or schema. [web:16][web:18]
- They are **not inherited** like normal privileges; you must be (or `SET ROLE` to) that role to use them. [web:17]
- For application roles: usually just `LOGIN`, everything else off.

### Example: Create a Normal App Role

```sql
CREATE ROLE app_user
  WITH LOGIN PASSWORD 'secure_password'
  NOSUPERUSER
  NOCREATEDB
  NOCREATEROLE
  NOREPLICATION
  NOBYPASSRLS;
```

### Example: Modify Attributes

```sql
-- Enable login
ALTER ROLE app_user WITH LOGIN;

-- Disable login (e.g., for a group role)
ALTER ROLE app_user WITH NOLOGIN;
```

---

## Layer 2: Object Privileges (Database, Schema, Tables, etc.)

Granted with `GRANT` / `REVOKE` on specific objects. This is where you enforce least privilege.

### Privilege Types by Object

#### Database Level

| Privilege | Description |
|-----------|-------------|
| `CONNECT` | Allow connecting to the database |
| `CREATE` | Allow creating new schemas in the database |
| `TEMPORARY` / `TEMP` | Allow creating temporary tables |

```sql
-- Allow connecting
GRANT CONNECT ON DATABASE my_db TO app_user;

-- Allow creating schemas (optional, often not needed for app roles)
GRANT CREATE ON DATABASE my_db TO app_user;
```

#### Schema Level

| Privilege | Description |
|-----------|-------------|
| `USAGE` | Allow referencing objects inside the schema (required to use tables, functions, etc.) |
| `CREATE` | Allow creating new objects (tables, functions, etc.) in the schema |

```sql
-- Allow using the schema
GRANT USAGE ON SCHEMA data TO app_user;

-- Allow creating objects in the schema
GRANT CREATE ON SCHEMA data TO app_user;
```

**Note:** Without `USAGE` on a schema, the role cannot access any objects inside it, even if table-level privileges exist. [web:6][web:12]

#### Table Level (includes views, foreign tables)

| Privilege | Description |
|-----------|-------------|
| `SELECT` | Read rows |
| `INSERT` | Add rows |
| `UPDATE` | Modify rows |
| `DELETE` | Remove rows |
| `TRUNCATE` | Empty the table |
| `REFERENCES` | Create foreign key constraints |
| `TRIGGER` | Create triggers |
| `ALL PRIVILEGES` | All of the above |

```sql
-- All tables in a schema
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA data TO app_user;

-- Shortcut for all table privileges
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA data TO app_user;
```

#### Sequence Level

| Privilege | Description |
|-----------|-------------|
| `USAGE` | Use `nextval()`, `currval()`, `lastval()` |
| `SELECT` | Read sequence values |
| `UPDATE` | Modify sequence values |
| `ALL PRIVILEGES` | All of the above |

```sql
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA data TO app_user;
```

#### Function / Procedure Level

| Privilege | Description |
|-----------|-------------|
| `EXECUTE` | Call the function/procedure |
| `ALL PRIVILEGES` | Currently just `EXECUTE` for functions |

```sql
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA logic TO app_user;
```

---

## Typical App Role Setup (Minimal Privileges)

Scenario: One database, two schemas (`data` for tables, `logic` for stored procedures), one app role.

### 1. Create the Role

```sql
CREATE ROLE app_user
  WITH LOGIN PASSWORD 'secure_password'
  NOSUPERUSER
  NOCREATEDB
  NOCREATEROLE;
```

### 2. Grant Database Connect

```sql
GRANT CONNECT ON DATABASE my_db TO app_user;
```

### 3. Create Schemas (run as superuser or DB owner)

```sql
CREATE SCHEMA data;
CREATE SCHEMA logic;
```

### 4. Grant Schema-Level Privileges

```sql
-- Allow using both schemas
GRANT USAGE ON SCHEMA data TO app_user;
GRANT USAGE ON SCHEMA logic TO app_user;

-- Allow creating objects in these schemas (optional)
GRANT CREATE ON SCHEMA data TO app_user;
GRANT CREATE ON SCHEMA logic TO app_user;
```

### 5. Grant Object-Level Privileges

```sql
-- Existing tables and sequences in data schema
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA data TO app_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA data TO app_user;

-- Existing functions in logic schema
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA logic TO app_user;
```

### 6. Set Default Privileges for Future Objects

Run these as the role that will create the objects (or use `FOR ROLE`):

```sql
-- Future tables and sequences in data schema
ALTER DEFAULT PRIVILEGES IN SCHEMA data
    GRANT ALL PRIVILEGES ON TABLES TO app_user;

ALTER DEFAULT PRIVILEGES IN SCHEMA data
    GRANT ALL PRIVILEGES ON SEQUENCES TO app_user;

-- Future functions in logic schema
ALTER DEFAULT PRIVILEGES IN SCHEMA logic
    GRANT ALL PRIVILEGES ON FUNCTIONS TO app_user;
```

If another role (`creator_role`) will create objects:

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE creator_role IN SCHEMA data
    GRANT ALL PRIVILEGES ON TABLES TO app_user;
```

---

## Key Notes & Mental Model

### 1. Two-Layer Model

- **Role attributes** = "What can this role do at the server level?"  
  - `SUPERUSER`, `CREATEDB`, `CREATEROLE`, etc.
  - Global, powerful, rarely needed for app roles.

- **Object privileges** = "What can this role do on this specific object?"  
  - `CONNECT`, `USAGE`, `CREATE`, `SELECT`, `UPDATE`, `EXECUTE`, etc.
  - Fine-grained, per database/schema/table/function.

### 2. Default Behavior

- New roles have **no attributes** except `NOLOGIN` (unless you specify `LOGIN`). [web:20]
- New roles have **no privileges** on any objects until explicitly granted. [web:23]
- Object creators are the **owners** of those objects and have full control by default. [web:3][web:8]

### 3. Schema Access Requires Two Things

To use a table `data.my_table`, a role typically needs:

1. `USAGE` on the schema `data`.
2. At least `SELECT` (or `ALL PRIVILEGES`) on the table `data.my_table`.

Without `USAGE` on the schema, table privileges alone are not enough. [web:6][web:12]

### 4. Least Privilege Pattern for App Roles

For most application roles:

- Role attributes: `LOGIN` only, everything else off.
- Database: `CONNECT` only (no `CREATE` unless the app must create schemas).
- Schemas: `USAGE` + `CREATE` only on the schemas the app owns.
- Tables/sequences/functions: precise privileges (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `EXECUTE`) or `ALL PRIVILEGES` if the app fully owns them.

### 5. Superuser Warning

- `SUPERUSER` bypasses **all** permission checks (except login). [web:16][web:24]
- Never use `SUPERUSER` for application roles.
- Use a superuser only for initial setup, then drop to a restricted role.

### 6. Inheritance vs Attributes

- **Privileges** granted to roles are inherited by members (if `INHERIT` is set, which is default). [web:17][web:19]
- **Role attributes** (`SUPERUSER`, `CREATEDB`, etc.) are **not inherited**; you must actually be that role or `SET ROLE` to it. [web:17]

Example:

```sql
-- Group role with privileges
CREATE ROLE app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA data TO app_readonly;

-- User role that inherits from group
CREATE ROLE reader_user WITH LOGIN PASSWORD '...';
GRANT app_readonly TO reader_user;

-- reader_user can SELECT on tables via inheritance
```

---

## Quick Reference Commands

```sql
-- Inspect role attributes
\du

-- Inspect role membership
\du+

-- List privileges on tables
\dp schema_name.table_name

-- List privileges on schemas
\dn+

-- List default privileges
\ddp
```

```sql
-- Revoke all on a schema (example)
REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA data FROM app_user;
REVOKE USAGE ON SCHEMA data FROM app_user;
```

---

## Checklist for Your Setup

- [ ] Role has `LOGIN` and no dangerous attributes (`NOSUPERUSER`, `NOCREATEDB`, `NOCREATEROLE`).
- [ ] Role has `CONNECT` on the target database only.
- [ ] Role has `USAGE` (and optionally `CREATE`) only on required schemas.
- [ ] Role has appropriate privileges on existing tables, sequences, functions.
- [ ] Default privileges are set so future objects are automatically accessible.
- [ ] No `SUPERUSER` or cluster-wide privileges for app roles.

---

## References

- PostgreSQL Documentation: Role Attributes – https://www.postgresql.org/docs/current/role-attributes.html [web:16][web:18]
- PostgreSQL Documentation: Database Roles – https://www.postgresql.org/docs/current/user-manag.html [web:21]
- PostgreSQL Documentation: GRANT – https://www.postgresql.org/docs/current/sql-grant.html [web:1]
- PostgreSQL Documentation: Privileges – https://www.postgresql.org/docs/current/ddl-priv.html [web:3]