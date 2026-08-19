# PostgreSQL Basic Administration Commands

This README contains essential PostgreSQL commands for databases, users/roles, permissions, ownership, backups, and restores.

## 1. Login to PostgreSQL

From a Linux terminal, log in as the PostgreSQL administrator:

```bash
sudo -u postgres psql
```

Log in to a specific database:

```bash
sudo -u postgres psql -d database_name
```

Log in with a password:

```bash
psql -h localhost -U username -d database_name -W
```

Exit `psql`:

```sql
\q
```

## 2. List Databases

```sql
\l
```

Or:

```sql
SELECT datname FROM pg_database;
```

Show the current database name:

```sql
SELECT current_database();
```

## 3. Create a Database

```sql
CREATE DATABASE mydb;
```

Create it with a specific owner:

```sql
CREATE DATABASE mydb OWNER myuser;
```

With encoding and locale settings:

```sql
CREATE DATABASE mydb
    WITH OWNER = myuser
    ENCODING = 'UTF8'
    TEMPLATE = template0;
```

## 4. Rename a Database and Change Its Owner

No one can be connected to the database while renaming it:

```sql
ALTER DATABASE olddb RENAME TO newdb;
```

Change the owner:

```sql
ALTER DATABASE mydb OWNER TO myuser;
```

Set the database connection limit:

```sql
ALTER DATABASE mydb CONNECTION LIMIT 50;
```

## 5. Drop/Delete a Database

```sql
DROP DATABASE mydb;
```

If the database cannot be dropped because other users are connected, terminate those connections first:

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'mydb'
  AND pid <> pg_backend_pid();
```

Then:

```sql
DROP DATABASE mydb;
```

> `DROP DATABASE` is irreversible. Make sure you have a backup before dropping a database.

## 6. List Users/Roles

```sql
\du
```

Or:

```sql
SELECT rolname, rolsuper, rolcreatedb, rolcreaterole, rolcanlogin
FROM pg_roles;
```

## 7. Create a User

A role without login permission or a password:

```sql
CREATE ROLE myuser;
```

Create a user with login permission and a password:

```sql
CREATE USER myuser WITH PASSWORD 'strong_password';
```

Equivalent syntax:

```sql
CREATE ROLE myuser
    LOGIN
    PASSWORD 'strong_password';
```

## 8. Change a User Password

Typing a password directly in SQL may leave it in shell history or logs. Use this safer method:

```sql
\password myuser
```

Or use SQL:

```sql
ALTER USER myuser WITH PASSWORD 'new_strong_password';
```

## 9. Change User/Role Privileges

Allow the user to create databases:

```sql
ALTER ROLE myuser CREATEDB;
```

Allow the user to create roles:

```sql
ALTER ROLE myuser CREATEROLE;
```

Make the user a superuser. Do not do this for a normal application user:

```sql
ALTER ROLE myuser SUPERUSER;
```

Remove elevated privileges:

```sql
ALTER ROLE myuser NOCREATEDB NOCREATEROLE NOSUPERUSER;
```

Disable/enable login:

```sql
ALTER ROLE myuser NOLOGIN;
ALTER ROLE myuser LOGIN;
```

## 10. Rename and Delete a User

```sql
ALTER ROLE olduser RENAME TO newuser;
```

Before deleting a user, reassign or remove objects owned by that user:

```sql
REASSIGN OWNED BY olduser TO postgres;
DROP OWNED BY olduser;
DROP ROLE olduser;
```

## 11. Grant Full Access to a Database

Make the user the database owner:

```sql
ALTER DATABASE mydb OWNER TO myuser;
```

Database-level privileges:

```sql
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;
```

Then connect to the target database and grant schema, table, sequence, and function permissions:

```sql
\c mydb

GRANT ALL ON SCHEMA public TO myuser;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO myuser;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO myuser;
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public TO myuser;
```

Automatically grant permissions on tables and sequences created in the future:

```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT ALL PRIVILEGES ON TABLES TO myuser;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT ALL PRIVILEGES ON SEQUENCES TO myuser;
```

## 12. View and Revoke Permissions

```sql
\l+
\dn+
\dp
```

Revoke permissions:

```sql
REVOKE ALL PRIVILEGES ON DATABASE mydb FROM myuser;
REVOKE ALL PRIVILEGES ON SCHEMA public FROM myuser;
REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM myuser;
```

## 13. List Tables and Schemas

```sql
\c mydb
\dn
\dt
\dt public.*
\d table_name
\d+ table_name
```

## 14. View Current Connections and Activity

```sql
SELECT current_user, session_user, current_database();
```

```sql
SELECT pid, usename, datname, client_addr, state, query
FROM pg_stat_activity;
```

Terminate a specific connection:

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE pid = 12345;
```

## 15. Backup

Create a custom-format backup:

```bash
pg_dump -U myuser -d mydb -F c -f mydb.dump
```

Create a plain SQL-format backup:

```bash
pg_dump -U myuser -d mydb -F p -f mydb.sql
```

Back up only the schema:

```bash
pg_dump -U myuser -d mydb --schema-only -f schema.sql
```

Back up only the data:

```bash
pg_dump -U myuser -d mydb --data-only -f data.sql
```

Back up the entire cluster, including all databases and roles:

```bash
sudo -u postgres pg_dumpall > all_databases.sql
```

## 16. Restore

Restore a custom `.dump` file:

```bash
pg_restore -U myuser -d mydb --clean --if-exists mydb.dump
```

Restore a plain SQL file:

```bash
psql -U myuser -d mydb -f mydb.sql
```

If peer authentication fails, run the restore as the PostgreSQL administrator:

```bash
sudo -u postgres pg_restore -d mydb mydb.dump
sudo -u postgres psql -d mydb -f mydb.sql
```

## 17. PostgreSQL Service Commands

```bash
sudo systemctl status postgresql
sudo systemctl start postgresql
sudo systemctl stop postgresql
sudo systemctl restart postgresql
sudo systemctl enable postgresql
```

Check the version:

```bash
psql --version
sudo -u postgres psql -c "SELECT version();"
```

## 18. Useful safety notes

- Do not use `SUPERUSER` for a normal application user.
- Use `\password username` instead of typing a database password directly on the command line.
- Use `DROP DATABASE`, `DROP ROLE`, `--clean`, and `pg_terminate_backend()` carefully.
- Verify database and user ownership before restoring a production backup.
- Creating a user alone is not enough for remote access; configure `listen_addresses`, `pg_hba.conf`, and the firewall/security group as well.

## 19. Basic Django Database User Setup

```sql
CREATE USER django_user WITH PASSWORD 'strong_password';
CREATE DATABASE django_db OWNER django_user;

\c django_db
GRANT ALL ON SCHEMA public TO django_user;
ALTER SCHEMA public OWNER TO django_user;
```

Then add these values to the Django `.env` file:

```env
DB_NAME=django_db
DB_USER=django_user
DB_PASSWORD=strong_password
DB_HOST=127.0.0.1
DB_PORT=5432
```
