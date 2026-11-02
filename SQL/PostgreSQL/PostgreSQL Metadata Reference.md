# PostgreSQL Metadata Reference

This reference lists the primary PostgreSQL metadata sources for inspecting database roles, databases, schemas, relations, columns, constraints, privileges, ownership, and operational objects.

PostgreSQL metadata is primarily available through:

1. `information_schema` — SQL-standard metadata views. Prefer this for portable, readable inspection queries.
2. `pg_catalog` — PostgreSQL system catalogs containing native, detailed metadata.
3. PostgreSQL system views — convenience views such as `pg_roles`, `pg_tables`, `pg_indexes`, and `pg_policies`.

> Read system catalogs and views, but never modify `pg_catalog` tables directly.

---

## Main metadata objects

| Need | Recommended source | Key fields / purpose |
|---|---|---|
| Roles and login attributes | `pg_roles` | `rolname`, `rolcanlogin`, `rolsuper`, `rolcreaterole`, `rolcreatedb`, `rolvaliduntil` |
| Role memberships | `pg_auth_members` | Membership relationships between roles; `admin_option` |
| Databases | `pg_database` | Database name, owner, encoding, connection limit, connection allowance |
| Schemas | `information_schema.schemata`, `pg_namespace` | Schema name and owner |
| Tables | `information_schema.tables`, `pg_tables`, `pg_class` | Table name, schema, type, owner and relation details |
| Columns | `information_schema.columns`, `pg_attribute` | Column names, types, defaults, nullability, ordinal order |
| Constraints | `information_schema.table_constraints`, `pg_constraint` | Primary keys, foreign keys, unique constraints, check constraints |
| Indexes | `pg_indexes`, `pg_index` | Index name and complete index definition |
| Sequences | `information_schema.sequences`, `pg_sequences` | Sequence configuration and ownership |
| Views | `information_schema.views`, `pg_views` | View names, owners and definitions |
| Materialized views | `pg_matviews` | Materialized-view definitions, owner and population state |
| Functions and procedures | `pg_proc` | Function/procedure names, arguments, return types and languages |
| Triggers | `information_schema.triggers`, `pg_trigger` | Trigger timing, event, table and function |
| Extensions | `pg_extension`, `pg_available_extensions` | Installed and available extensions |
| Types and enums | `pg_type`, `pg_enum` | Custom types, domains and enum labels |
| Table privileges | `information_schema.role_table_grants`, `information_schema.table_privileges` | Grants assigned to roles on tables/views |
| Column privileges | `information_schema.column_privileges` | Per-column grants |
| Routine privileges | `information_schema.role_routine_grants` | Execute grants on functions/procedures |
| Default privileges | `pg_default_acl` | Privileges automatically applied to objects created later |
| Row-level security policies | `pg_policies`, `pg_policy` | RLS policy roles, commands and expressions |
| Dependencies | `pg_depend` | Object dependency graph |
| Object ownership | `pg_class`, `pg_namespace`, `pg_get_userbyid()` | Owner of tables, views, sequences and schemas |

---

## Roles and memberships

### Roles and attributes

```sql
SELECT
    rolname AS role_name,
    rolcanlogin AS can_login,
    rolsuper AS is_superuser,
    rolcreaterole AS can_create_roles,
    rolcreatedb AS can_create_databases,
    rolreplication AS can_replicate,
    rolbypassrls AS bypasses_row_level_security,
    rolinherit AS inherits_privileges,
    rolvaliduntil AS password_valid_until
FROM pg_roles
ORDER BY rolname;
```

### Inspect selected roles

```sql
SELECT
    rolname,
    rolcanlogin,
    rolsuper,
    rolcreatedb,
    rolcreaterole,
    rolvaliduntil
FROM pg_roles
WHERE rolname IN ('superalert_user', 'superalert_dev');
```

### Role membership

```sql
SELECT
    member_role.rolname AS member_role,
    parent_role.rolname AS member_of_role,
    membership.admin_option
FROM pg_auth_members AS membership
JOIN pg_roles AS member_role
    ON member_role.oid = membership.member
JOIN pg_roles AS parent_role
    ON parent_role.oid = membership.roleid
ORDER BY member_role.rolname, parent_role.rolname;
```

---

## Databases and schemas

### Databases in the current PostgreSQL cluster

```sql
SELECT
    datname AS database_name,
    pg_get_userbyid(datdba) AS owner,
    pg_encoding_to_char(encoding) AS encoding,
    datcollate AS collation,
    datctype AS character_type,
    datconnlimit AS connection_limit,
    datallowconn AS allows_connections
FROM pg_database
ORDER BY datname;
```

### Schemas: portable form

```sql
SELECT
    schema_name,
    schema_owner
FROM information_schema.schemata
ORDER BY schema_name;
```

### Schemas and owners: PostgreSQL-native form

```sql
SELECT
    n.nspname AS schema_name,
    pg_get_userbyid(n.nspowner) AS schema_owner
FROM pg_namespace AS n
WHERE n.nspname NOT LIKE 'pg_%'
  AND n.nspname <> 'information_schema'
ORDER BY n.nspname;
```

---

## Tables, columns, and relations

### Application tables

```sql
SELECT
    table_schema,
    table_name,
    table_type
FROM information_schema.tables
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY table_schema, table_name;
```

### Tables with owners and security indicators

```sql
SELECT
    schemaname AS schema_name,
    tablename AS table_name,
    tableowner AS owner,
    hasindexes,
    hastriggers,
    rowsecurity
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename;
```

### Columns, data types, defaults and nullability

```sql
SELECT
    table_schema,
    table_name,
    ordinal_position,
    column_name,
    data_type,
    udt_name AS postgres_data_type,
    is_nullable,
    column_default,
    character_maximum_length
FROM information_schema.columns
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY table_schema, table_name, ordinal_position;
```

### Columns for one table

```sql
SELECT
    ordinal_position,
    column_name,
    data_type,
    udt_name,
    is_nullable,
    column_default
FROM information_schema.columns
WHERE table_schema = 'superalert'
  AND table_name = 'users'
ORDER BY ordinal_position;
```

Replace `superalert` and `users` with your schema and table names.

---

## Constraints and relationships

### All table constraints

```sql
SELECT
    tc.table_schema,
    tc.table_name,
    tc.constraint_name,
    tc.constraint_type
FROM information_schema.table_constraints AS tc
WHERE tc.table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY tc.table_schema, tc.table_name, tc.constraint_type, tc.constraint_name;
```

### Foreign-key relationships

```sql
SELECT
    tc.table_schema,
    tc.table_name,
    kcu.column_name AS foreign_key_column,
    ccu.table_schema AS referenced_schema,
    ccu.table_name AS referenced_table,
    ccu.column_name AS referenced_column,
    tc.constraint_name
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
    ON tc.constraint_name = kcu.constraint_name
    AND tc.table_schema = kcu.table_schema
JOIN information_schema.constraint_column_usage AS ccu
    ON ccu.constraint_name = tc.constraint_name
    AND ccu.table_schema = tc.table_schema
WHERE tc.constraint_type = 'FOREIGN KEY'
ORDER BY tc.table_schema, tc.table_name, tc.constraint_name;
```

### PostgreSQL-native constraint definitions

```sql
SELECT
    n.nspname AS schema_name,
    c.relname AS table_name,
    con.conname AS constraint_name,
    pg_get_constraintdef(con.oid) AS definition
FROM pg_constraint AS con
JOIN pg_class AS c
    ON c.oid = con.conrelid
JOIN pg_namespace AS n
    ON n.oid = c.relnamespace
WHERE n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY n.nspname, c.relname, con.conname;
```

---

## Indexes, sequences, views, and routines

### Indexes

```sql
SELECT
    schemaname AS schema_name,
    tablename AS table_name,
    indexname AS index_name,
    indexdef AS index_definition
FROM pg_indexes
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename, indexname;
```

### Sequences

```sql
SELECT
    schemaname AS schema_name,
    sequencename AS sequence_name,
    sequenceowner AS owner,
    data_type,
    start_value,
    min_value,
    max_value,
    increment_by,
    cycle
FROM pg_sequences
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, sequencename;
```

### Views

```sql
SELECT
    schemaname AS schema_name,
    viewname AS view_name,
    viewowner AS owner,
    definition
FROM pg_views
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, viewname;
```

### Functions and procedures

```sql
SELECT
    n.nspname AS schema_name,
    p.proname AS routine_name,
    pg_get_function_identity_arguments(p.oid) AS arguments,
    pg_get_function_result(p.oid) AS return_type,
    l.lanname AS language,
    CASE p.prokind
        WHEN 'f' THEN 'function'
        WHEN 'p' THEN 'procedure'
        WHEN 'a' THEN 'aggregate'
        WHEN 'w' THEN 'window function'
    END AS routine_type
FROM pg_proc AS p
JOIN pg_namespace AS n
    ON n.oid = p.pronamespace
JOIN pg_language AS l
    ON l.oid = p.prolang
WHERE n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY n.nspname, p.proname;
```

---

## Privileges and ownership

### Explicit table privileges

```sql
SELECT
    table_schema,
    table_name,
    grantee,
    privilege_type,
    is_grantable
FROM information_schema.role_table_grants
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY table_schema, table_name, grantee, privilege_type;
```

### Privileges for a particular role

```sql
SELECT
    table_schema,
    table_name,
    privilege_type
FROM information_schema.role_table_grants
WHERE grantee = 'superalert_user'
ORDER BY table_schema, table_name, privilege_type;
```

### Check schema privileges for specific roles

```sql
SELECT
    n.nspname AS schema_name,
    r.rolname AS role_name,
    has_schema_privilege(r.rolname, n.oid, 'USAGE') AS has_usage,
    has_schema_privilege(r.rolname, n.oid, 'CREATE') AS has_create
FROM pg_namespace AS n
CROSS JOIN pg_roles AS r
WHERE n.nspname = 'superalert'
  AND r.rolname IN ('superalert_user', 'superalert_dev')
ORDER BY n.nspname, r.rolname;
```

### Object ownership

```sql
SELECT
    n.nspname AS schema_name,
    c.relname AS object_name,
    CASE c.relkind
        WHEN 'r' THEN 'table'
        WHEN 'p' THEN 'partitioned table'
        WHEN 'v' THEN 'view'
        WHEN 'm' THEN 'materialized view'
        WHEN 'S' THEN 'sequence'
        WHEN 'f' THEN 'foreign table'
        ELSE c.relkind::text
    END AS object_type,
    pg_get_userbyid(c.relowner) AS owner
FROM pg_class AS c
JOIN pg_namespace AS n
    ON n.oid = c.relnamespace
WHERE n.nspname NOT IN ('pg_catalog', 'information_schema')
  AND c.relkind IN ('r', 'p', 'v', 'm', 'S', 'f')
ORDER BY n.nspname, c.relname;
```

### Default privileges for future objects

```sql
SELECT
    pg_get_userbyid(d.defaclrole) AS creating_role,
    n.nspname AS schema_name,
    d.defaclobjtype AS object_type,
    d.defaclacl AS default_acl
FROM pg_default_acl AS d
LEFT JOIN pg_namespace AS n
    ON n.oid = d.defaclnamespace
ORDER BY schema_name, creating_role, object_type;
```

---

## Security and operational metadata

### Row-level security policies

```sql
SELECT
    schemaname AS schema_name,
    tablename AS table_name,
    policyname AS policy_name,
    permissive,
    roles,
    cmd AS command,
    qual AS using_expression,
    with_check AS with_check_expression
FROM pg_policies
ORDER BY schemaname, tablename, policyname;
```

### Installed extensions

```sql
SELECT
    e.extname AS extension_name,
    e.extversion AS version,
    n.nspname AS installed_schema
FROM pg_extension AS e
JOIN pg_namespace AS n
    ON n.oid = e.extnamespace
ORDER BY e.extname;
```

### Available extensions

```sql
SELECT
    name,
    default_version,
    installed_version,
    comment
FROM pg_available_extensions
ORDER BY name;
```

### Current database connections

```sql
SELECT
    datname AS database_name,
    usename AS role_name,
    application_name,
    client_addr,
    state,
    backend_start,
    query_start,
    query
FROM pg_stat_activity
ORDER BY backend_start DESC;
```

> Access to the full query text and other sessions may be restricted for non-superuser roles.

---

## SuperAlert security audit query

Use this as a repeatable validation query after provisioning roles, schemas, tables, and grants.

```sql
-- Roles and administrative attributes
SELECT
    rolname AS role_name,
    rolcanlogin AS can_login,
    rolsuper AS is_superuser,
    rolcreaterole AS can_create_roles,
    rolcreatedb AS can_create_databases,
    rolinherit AS inherits_privileges,
    rolbypassrls AS bypasses_row_level_security,
    rolvaliduntil AS password_valid_until
FROM pg_roles
ORDER BY rolname;

-- Role membership
SELECT
    member_role.rolname AS member_role,
    parent_role.rolname AS member_of_role,
    membership.admin_option
FROM pg_auth_members AS membership
JOIN pg_roles AS member_role
    ON member_role.oid = membership.member
JOIN pg_roles AS parent_role
    ON parent_role.oid = membership.roleid
ORDER BY member_role.rolname, parent_role.rolname;

-- Schemas and owners
SELECT
    n.nspname AS schema_name,
    pg_get_userbyid(n.nspowner) AS schema_owner
FROM pg_namespace AS n
WHERE n.nspname NOT LIKE 'pg_%'
  AND n.nspname <> 'information_schema'
ORDER BY n.nspname;

-- Application relations and owners
SELECT
    schemaname AS schema_name,
    tablename AS table_name,
    tableowner AS table_owner,
    rowsecurity AS row_level_security_enabled
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename;

-- Explicit table grants
SELECT
    table_schema,
    table_name,
    grantee,
    privilege_type
FROM information_schema.role_table_grants
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY table_schema, table_name, grantee, privilege_type;
```

---

## `psql` meta-commands

These commands work in the `psql` terminal client. They are not SQL statements and generally do not work in pgAdmin’s Query Tool.

```text
\du              -- roles
\l               -- databases
\dn              -- schemas
\dt              -- tables
\dv              -- views
\di              -- indexes
\d schema.table  -- full table definition
\dp              -- tables/sequences and access privileges
\drg             -- role grants and memberships
```

---

## Recommended practice

- Prefer `information_schema` for portable scripts and human-readable reports.
- Use `pg_catalog` when you need PostgreSQL-specific attributes, ownership, policies, default ACLs, object dependencies, or full object definitions.
- Use `pg_roles` to audit role attributes, and `pg_auth_members` to audit inherited group privileges.
- Treat object ownership as a security permission: the owner has powerful implicit control even if explicit grants appear limited.
- Keep the runtime application role non-superuser, non-owner, and limited to the specific schema/object privileges it needs.
- Keep the provisioning/migration owner distinct from the application runtime role where possible.
