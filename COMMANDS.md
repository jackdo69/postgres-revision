# PostgreSQL Command Reference

## Database Management Commands

### Creating Databases
- [ ] `createdb -h localhost -U devuser dbname` - Create a new database (shell command)
- [ ] `CREATE DATABASE dbname;` - Create a new database (SQL command)

### Dropping Databases
- [ ] `dropdb -h localhost -U devuser dbname` - Delete a database (shell command)
- [ ] `DROP DATABASE dbname;` - Delete a database (SQL command, must not be connected to it)

**Force drop with active connections:**
```sql
-- Terminate all connections to the database first
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'dbname' AND pid <> pg_backend_pid();

-- Then drop it
DROP DATABASE dbname;
```

## Essential psql Meta-Commands

Master these essential **psql meta-commands** regardless of client choice:

- [ ] `\l+` - List databases with sizes
- [ ] `\dt+` - List tables with sizes
- [ ] `\d+ tablename` - Detailed table structure
- [ ] `\di` - List indexes
- [ ] `\x auto` - Toggle expanded display
- [ ] `\timing` - Show query execution time
- [ ] `\e` - Edit query in $EDITOR
- [ ] `\conninfo` - Show current connection information
- [ ] `\c` - Show current connection or connect to new database

## Schema Commands

- [ ] `SHOW search_path;` - Show current schema search path
- [ ] `SELECT current_schema();` - Show current schema (first in search path)
- [ ] `SELECT current_schemas(true);` - Show all schemas in search path
- [ ] `SET search_path TO schema_name, public;` - Change schema search path for session
- [ ] `\dn` - List all schemas in database
