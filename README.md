# Mastering PostgreSQL: A CLI-first guide for senior engineers

PostgreSQL proficiency combines robust local setup, powerful command-line tooling, and progressive mastery of internals. This guide provides a structured path from installation through advanced performance optimization, emphasizing the CLI-first workflow that experienced engineers prefer. The key to efficiency is **pgcli** for daily work, **golang-migrate** for migrations, and **pg_stat_statements** for performance monitoring—tools that integrate seamlessly into terminal-based development.

## - [x] Local setup with Docker and native installations

The fastest path to a working PostgreSQL environment uses **Docker** for isolation and reproducibility. Create a `docker-compose.yml` file for instant database provisioning:

```yaml
services:
  db:
    image: postgres:17-alpine
    container_name: postgres_dev
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpass
      POSTGRES_DB: devdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
volumes:
  postgres_data:
```

Running `docker compose up -d` provides a production-grade database in seconds.

Configure `postgresql.conf` for development with verbose logging (`log_statement = 'all'`), relaxed synchronous commits, and appropriate memory settings—**256MB for shared_buffers** and **64MB for work_mem** work well for local development. The official documentation at postgresql.org/docs/current remains the authoritative reference for all configuration options.

## - [x] pgcli and essential CLI tools for daily work

Use **pgcli** (github.com/dbcli/pgcli) for syntax highlighting and intelligent autocomplete that dramatically accelerates query writing. Install via `brew install pgcli` or `pip install pgcli`, then connect: `pgcli -h localhost -U devuser -d devdb`.

For essential psql meta-commands, see **COMMANDS.md**.


## - [x] Sample databases and data generation for practice

**PGExercises** (pgexercises.com) provides the best hands-on SQL practice specifically for PostgreSQL, covering everything from basic queries through window functions and recursive CTEs. Download their `clubdata.sql` for local practice:

```bash
# Download clubdata.sql from pgexercises.com
# Then load it (it creates the 'exercises' database automatically)
psql -h localhost -U devuser -f clubdata.sql

# Connect to the exercises database
pgcli -h localhost -U devuser -d exercises

# Set search path to access tables easily
SET search_path TO cd, public;
```

The clubdata database contains three tables in the `cd` schema: `members`, `facilities`, and `bookings`. See **clubdata-schema.txt** for the complete schema diagram.

For quick benchmark data, **pgbench** ships with PostgreSQL: `pgbench -h localhost -U devuser -i -s 10 devdb` generates approximately 1.5GB of test data. For custom synthetic data, combine `generate_series()` with random functions directly in SQL:

```sql
CREATE TABLE test_data AS
SELECT generate_series(1, 1000000) AS id,
       md5(random()::text) AS random_text,
       now() - interval '1 day' * random() * 365 AS created_at;
```

## - [ ] Migration tools built for CLI workflows

**golang-migrate** (github.com/golang-migrate/migrate) offers the cleanest CLI-first migration experience with zero runtime dependencies:

```bash
# Install
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Create migration pair
migrate create -ext sql -dir db/migrations -seq create_users_table

# Apply all pending migrations
migrate -database "postgres://user:pass@localhost:5432/db?sslmode=disable" \
        -path db/migrations up

# Rollback one migration
migrate -database "..." -path db/migrations down 1
```

Each migration creates paired `.up.sql` and `.down.sql` files numbered sequentially. **Sqitch** (sqitch.org) provides an alternative with dependency tracking between migrations and native VCS integration.

For zero-downtime schema changes on production databases, **pgroll** (github.com/xataio/pgroll) implements the expand-contract pattern automatically, maintaining dual schema versions during transitions. **pg_repack** (github.com/reorg/pg_repack) handles online table reorganization when removing bloat without locking.

Critical migration practices include using `CREATE INDEX CONCURRENTLY` to avoid table locks, setting `lock_timeout = '5s'` before DDL operations, and batching large data updates rather than running single massive transactions.

## - [ ] Transactions, locking, and MVCC deep dive

PostgreSQL's **Multi-Version Concurrency Control (MVCC)** means readers never block writers and writers never block readers—each transaction sees a consistent snapshot. Understanding this architecture proves essential for debugging concurrency issues.

PostgreSQL implements three isolation levels:

- **READ COMMITTED** (default): Sees only committed data at statement start
- **REPEATABLE READ**: Sees snapshot from transaction start; prevents phantom reads
- **SERIALIZABLE**: True serializability via SSI; detects write skew anomalies

Monitor locking issues with this diagnostic query:

```sql
SELECT blocked_locks.pid AS blocked_pid,
       blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_query
FROM pg_locks blocked_locks
JOIN pg_locks blocking_locks 
  ON blocking_locks.locktype = blocked_locks.locktype
JOIN pg_stat_activity blocked_activity 
  ON blocked_activity.pid = blocked_locks.pid
WHERE NOT blocked_locks.granted;
```

The free online book "The Internals of PostgreSQL" at **interdb.jp/pg** provides the deepest coverage of MVCC, vacuum processing, and buffer management. Complement this with "PostgreSQL 14 Internals" by Egor Rogov, available free at postgrespro.com/community/books/internals.

## - [ ] Performance optimization and profiling techniques

Enable **pg_stat_statements** immediately—it's the single most valuable performance tool:

```sql
-- Add to postgresql.conf: shared_preload_libraries = 'pg_stat_statements'
CREATE EXTENSION pg_stat_statements;

-- Find slowest queries by average time
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 10;
```

Master **EXPLAIN ANALYZE** with buffer statistics: `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT...`. Watch for large discrepancies between estimated and actual rows—they indicate stale statistics requiring `ANALYZE`.

Choose index types strategically:

- **B-tree**: Default choice for equality and range queries
- **GIN**: Required for JSONB containment (`@>`) and array operations
- **BRIN**: Efficient for large time-series tables with natural ordering
- **GiST**: Geometric data and full-text search

For connection pooling, **PgBouncer** in transaction mode (`pool_mode = transaction`) handles thousands of connections with minimal memory overhead. Configure `default_pool_size = 20` and `max_client_conn = 1000` for typical web applications.

CLI monitoring tools include **pg_activity** (`pip install pg_activity`) for htop-like real-time views, **pgmetrics** (github.com/rapidloop/pgmetrics) for JSON metrics collection, and **pgbadger** (github.com/darold/pgbadger) for log analysis. **PgHero** (github.com/ankane/pghero) provides a lightweight web dashboard identifying slow queries and missing indexes.

## - [ ] Structured learning progression

- [x] **Phase 1 (Weeks 1-2)**: Master pgcli workflow, load clubdata database from PGExercises, complete PGExercises basic and join sections, configure pg_stat_statements.

- [ ] **Phase 2 (Weeks 3-4)**: Study official documentation chapters on indexes and query planning, practice EXPLAIN ANALYZE interpretation, implement first migration project with golang-migrate.

- [ ] **Phase 3 (Weeks 5-8)**: Read "The Internals of PostgreSQL" (interdb.jp/pg) covering MVCC and vacuum, study all index types with practical examples, configure and benchmark PgBouncer.

- [ ] **Phase 4 (Weeks 9-12)**: Study "PostgreSQL 14 Internals" book, implement partitioning strategies, practice zero-downtime migrations with pgroll, set up comprehensive monitoring.

**Essential reading** includes "The Art of PostgreSQL" by Dimitri Fontaine for SQL mastery and "PostgreSQL Query Optimization" by Henrietta Dombrovskaya for systematic performance tuning. The **PostgreSQL for Everybody** Coursera specialization from University of Michigan provides structured video content covering fundamentals through database architecture.

## - [ ] Conclusion

Effective PostgreSQL mastery for senior engineers centers on three pillars: a reproducible Docker-based local environment, CLI tools that accelerate daily work (pgcli, golang-migrate, pg_stat_statements), and deep understanding of MVCC internals. Start with PGExercises clubdata for hands-on practice, progress through official documentation and "The Internals of PostgreSQL," then advance to production patterns with pgroll migrations and PgBouncer pooling.

The PostgreSQL ecosystem rewards investment—its transactional DDL, JSONB capabilities, and rich extension system (pg_stat_statements, auto_explain, pg_repack) create a development experience where the database becomes a powerful application component rather than just a storage layer.