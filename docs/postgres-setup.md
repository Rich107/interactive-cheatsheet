# Postgres: create database, schema, and table

Quick reference for the standard `psql` commands to spin up a new database, then create a schema and table inside it. Works against any reachable Postgres instance (local, Docker, or remote).

## 1. Connect to Postgres

```bash
# Connect as a superuser (default install)
psql -U postgres

# Or against a specific host/port
psql -h localhost -p 5432 -U postgres
```

Using a connection string:

```bash
psql "postgres://postgres:password@localhost:5432/postgres"
```

## 2. Create the database

From inside `psql` (connected to the default `postgres` database):

```sql
CREATE DATABASE my_new_db;
```

With optional flags:

```sql
CREATE DATABASE my_new_db
  OWNER my_user
  ENCODING 'UTF8'
  LC_COLLATE 'en_US.UTF-8'
  LC_CTYPE   'en_US.UTF-8'
  TEMPLATE template0;
```

Or from the shell without entering psql:

```bash
createdb -U postgres my_new_db
```

## 3. Connect to the new database

Inside psql:

```sql
\c my_new_db
```

Or start a fresh session:

```bash
psql -U postgres -d my_new_db
```

## 4. Create a schema

```sql
CREATE SCHEMA my_schema;

-- Owned by a specific role
CREATE SCHEMA my_schema AUTHORIZATION my_user;

-- Idempotent variant
CREATE SCHEMA IF NOT EXISTS my_schema;
```

Optionally set it as the default for the current session:

```sql
SET search_path TO my_schema, public;
```

## 5. Create a table

```sql
CREATE TABLE my_schema.users (
  id          BIGSERIAL PRIMARY KEY,
  email       TEXT NOT NULL UNIQUE,
  full_name   TEXT,
  is_active   BOOLEAN NOT NULL DEFAULT true,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## 6. Verify

```sql
\l                   -- list databases
\dn                  -- list schemas in current DB
\dt my_schema.*      -- list tables in the schema
\d  my_schema.users  -- describe the table
```

## 7. (Optional) Tear down

```sql
DROP TABLE    my_schema.users;
DROP SCHEMA   my_schema CASCADE;
DROP DATABASE my_new_db;
```

## One-shot version (no interactive psql)

```bash
createdb -U postgres my_new_db

psql -U postgres -d my_new_db <<'SQL'
CREATE SCHEMA my_schema;

CREATE TABLE my_schema.users (
  id         BIGSERIAL PRIMARY KEY,
  email      TEXT NOT NULL UNIQUE,
  full_name  TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
SQL
```
