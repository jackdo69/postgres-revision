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

---

## SQL Syntax Reference

### DISTINCT
- **Purpose**: Remove duplicate rows from query results
- **Syntax**:
  ```sql
  SELECT DISTINCT column1, column2
  FROM table_name;
  ```
- **Example**:
  ```sql
  SELECT DISTINCT f.name, m.surname
  FROM cd.bookings b
  JOIN cd.members m ON b.memid = m.memid
  JOIN cd.facilities f ON b.facid = f.facid;
  ```

### JOIN
- **Purpose**: Combine rows from two or more tables based on a related column
- **Syntax**:
  ```sql
  SELECT columns
  FROM table1
  JOIN table2 ON table1.column = table2.column;
  ```
- **Example**:
  ```sql
  SELECT m.surname, b.starttime
  FROM cd.bookings b
  JOIN cd.members m ON b.memid = m.memid;
  ```

### String Concatenation (||)
- **Purpose**: Combine multiple strings into one
- **Syntax**:
  ```sql
  SELECT string1 || 'separator' || string2 AS combined
  FROM table_name;
  ```
- **Example**:
  ```sql
  SELECT surname || ', ' || firstname AS full_name
  FROM cd.members;
  ```

### LIKE Pattern Matching
- **Purpose**: Search for a specified pattern in a column
- **Syntax**:
  ```sql
  SELECT columns
  FROM table_name
  WHERE column LIKE 'pattern%';  -- % matches any characters
  ```
- **Example**:
  ```sql
  SELECT name
  FROM cd.facilities
  WHERE name LIKE 'Tennis Court%';
  ```

### CASE Statement
- **Purpose**: Conditional logic in SQL (like if-else)
- **Syntax**:
  ```sql
  CASE
      WHEN condition1 THEN result1
      WHEN condition2 THEN result2
      ELSE default_result
  END
  ```
- **Example**:
  ```sql
  SELECT name,
      CASE
          WHEN memid = 0 THEN guestcost * slots
          ELSE membercost * slots
      END AS cost
  FROM cd.bookings b
  JOIN cd.facilities f ON b.facid = f.facid;
  ```

### DATE() Function
- **Purpose**: Extract the date part from a timestamp
- **Syntax**:
  ```sql
  SELECT columns
  FROM table_name
  WHERE DATE(timestamp_column) = 'YYYY-MM-DD';
  ```
- **Example**:
  ```sql
  SELECT *
  FROM cd.bookings
  WHERE DATE(starttime) = '2012-09-14';
  ```

### ORDER BY
- **Purpose**: Sort result set by one or more columns
- **Syntax**:
  ```sql
  SELECT columns
  FROM table_name
  ORDER BY column1 ASC, column2 DESC;  -- ASC (default) or DESC
  ```
- **Example**:
  ```sql
  SELECT surname, firstname
  FROM cd.members
  ORDER BY surname ASC, firstname ASC;
  ```

### Subqueries

#### Scalar Subquery (in SELECT clause)
- **Purpose**: Return exactly ONE value (single row, single column) for each row
- **Important**: Must return at most one row, or it will error. Use aggregate functions (COUNT, MAX, etc.) or WHERE on unique keys.
- **Syntax**:
  ```sql
  SELECT
      column1,
      (SELECT aggregate_or_single_value FROM table2 WHERE condition) AS related_value
  FROM table1;
  ```
- **Common patterns**:
  ```sql
  -- Pattern 1: Using aggregate function (always returns 1 row)
  SELECT name,
      (SELECT COUNT(*) FROM cd.bookings WHERE facid = f.facid) AS booking_count
  FROM cd.facilities f;

  -- Pattern 2: WHERE on unique key (returns 0 or 1 row)
  SELECT m1.firstname || ' ' || m1.surname AS member,
      (
          SELECT m2.firstname || ' ' || m2.surname
          FROM cd.members m2
          WHERE m2.memid = m1.recommendedby  -- memid is unique (primary key)
      ) AS recommender
  FROM cd.members m1;
  ```

#### Derived Table Subquery (in FROM clause)
- **Purpose**: Avoid duplicating calculated columns in WHERE/HAVING clauses
- **Syntax**:
  ```sql
  SELECT columns
  FROM (
      SELECT column1, calculated_expression AS calc_column
      FROM table_name
  ) AS subquery_alias
  WHERE calc_column > value;  -- Reference calculated column without recalculating
  ```
- **Example**:
  ```sql
  SELECT facility, member, cost
  FROM (
      SELECT
          f.name AS facility,
          m.surname || ', ' || m.firstname AS member,
          CASE
              WHEN b.memid = 0 THEN f.guestcost * b.slots
              ELSE f.membercost * b.slots
          END AS cost
      FROM cd.bookings b
      JOIN cd.members m ON b.memid = m.memid
      JOIN cd.facilities f ON b.facid = f.facid
      WHERE DATE(b.starttime) = '2012-09-14'
  ) AS bookings
  WHERE cost > 30  -- Use calculated 'cost' without duplicating CASE logic
  ORDER BY cost DESC;
  ```

### INSERT
- **Purpose**: Add new rows to a table
- **Syntax**:
  ```sql
  -- Single row
  INSERT INTO table_name (column1, column2, column3)
  VALUES (value1, value2, value3);

  -- Multiple rows
  INSERT INTO table_name (column1, column2, column3)
  VALUES
      (value1, value2, value3),
      (value4, value5, value6),
      (value7, value8, value9);
  ```
- **Example**:
  ```sql
  -- Insert single facility
  INSERT INTO cd.facilities (facid, name, membercost, guestcost, initialoutlay, monthlymaintenance)
  VALUES (9, 'Spa', 20, 30, 100000, 800);

  -- Insert multiple facilities
  INSERT INTO cd.facilities (facid, name, membercost, guestcost, initialoutlay, monthlymaintenance)
  VALUES
      (9, 'Spa', 20, 30, 100000, 800),
      (10, 'Squash Court 2', 3.5, 17.5, 5000, 80);
  ```
- **Important notes**:
  - String values use single quotes: `'Spa'`
  - Numeric values have no quotes: `20, 3.5, 100000`
  - Multi-row inserts are atomic: if one fails, all fail
  - Primary key violations will cause the entire INSERT to fail

### INSERT with ON CONFLICT
- **Purpose**: Handle duplicate key conflicts gracefully (PostgreSQL 9.5+)
- **Syntax**:
  ```sql
  INSERT INTO table_name (column1, column2)
  VALUES (value1, value2)
  ON CONFLICT (unique_column) DO NOTHING;  -- Skip if exists

  -- Or update on conflict
  ON CONFLICT (unique_column) DO UPDATE SET column2 = EXCLUDED.column2;
  ```
- **Example**:
  ```sql
  -- Skip if facid already exists
  INSERT INTO cd.facilities (facid, name, membercost, guestcost, initialoutlay, monthlymaintenance)
  VALUES (9, 'Spa', 20, 30, 100000, 800)
  ON CONFLICT (facid) DO NOTHING;

  -- Update if facid already exists
  INSERT INTO cd.facilities (facid, name, membercost, guestcost, initialoutlay, monthlymaintenance)
  VALUES (9, 'Spa', 20, 30, 100000, 800)
  ON CONFLICT (facid) DO UPDATE SET
      name = EXCLUDED.name,
      membercost = EXCLUDED.membercost,
      guestcost = EXCLUDED.guestcost;
  ```

### UPDATE
- **Purpose**: Modify existing rows in a table
- **Syntax**:
  ```sql
  -- Update single column
  UPDATE table_name
  SET column1 = new_value
  WHERE condition;

  -- Update multiple columns
  UPDATE table_name
  SET column1 = value1,
      column2 = value2,
      column3 = value3
  WHERE condition;
  ```
- **Example**:
  ```sql
  -- Update single column
  UPDATE cd.facilities
  SET initialoutlay = 10000
  WHERE name = 'Tennis Court 2';

  -- Update multiple columns
  UPDATE cd.facilities
  SET membercost = 25,
      guestcost = 35
  WHERE name = 'Spa';

  -- Update with calculation
  UPDATE cd.facilities
  SET membercost = membercost * 1.1  -- Increase by 10%
  WHERE facid > 5;

  -- Update using subquery
  UPDATE cd.facilities
  SET monthlymaintenance = (SELECT AVG(monthlymaintenance) FROM cd.facilities)
  WHERE name = 'Spa';
  ```
- **CRITICAL WARNING**:
  - **Always include WHERE clause** unless you intentionally want to update ALL rows
  - Without WHERE, every row in the table will be updated!
  ```sql
  -- ❌ DANGEROUS - Updates ALL facilities
  UPDATE cd.facilities SET initialoutlay = 10000;

  -- ✅ SAFE - Updates only Tennis Court 2
  UPDATE cd.facilities SET initialoutlay = 10000 WHERE name = 'Tennis Court 2';
  ```

### DELETE
- **Purpose**: Remove rows from a table
- **Syntax**:
  ```sql
  -- Delete specific rows
  DELETE FROM table_name
  WHERE condition;

  -- Delete all rows (DANGEROUS!)
  DELETE FROM table_name;
  ```
- **Example**:
  ```sql
  -- Delete specific booking
  DELETE FROM cd.bookings
  WHERE bookid = 123;

  -- Delete all bookings before a date
  DELETE FROM cd.bookings
  WHERE starttime < '2012-01-01';

  -- Delete using NOT IN subquery
  DELETE FROM cd.members
  WHERE memid NOT IN (SELECT DISTINCT memid FROM cd.bookings);

  -- Delete using NOT EXISTS (more efficient)
  DELETE FROM cd.members m
  WHERE NOT EXISTS (
      SELECT 1 FROM cd.bookings b WHERE b.memid = m.memid
  );

  -- Delete all rows (removes all data, keeps table structure)
  DELETE FROM cd.bookings;
  ```
- **CRITICAL WARNING**:
  - **Always include WHERE clause** unless you intentionally want to delete ALL rows
  - Deletions are permanent (unless in a transaction with ROLLBACK)
  - **Best practice**: Run a SELECT first to verify what will be deleted
  ```sql
  -- ✅ Verify first
  SELECT * FROM cd.members WHERE memid = 5;
  -- Then delete
  DELETE FROM cd.members WHERE memid = 5;
  ```

### TRUNCATE
- **Purpose**: Quickly remove all rows from a table (faster than DELETE for large tables)
- **Syntax**:
  ```sql
  TRUNCATE TABLE table_name;

  -- Truncate multiple tables
  TRUNCATE TABLE table1, table2, table3;

  -- Truncate with cascade (also truncate dependent tables)
  TRUNCATE TABLE table_name CASCADE;
  ```
- **Example**:
  ```sql
  -- Remove all bookings
  TRUNCATE TABLE cd.bookings;
  ```
- **Differences from DELETE**:
  - **Much faster** for large tables (doesn't scan rows)
  - **Resets auto-increment sequences** back to initial value
  - **Cannot use WHERE clause** - always removes all rows
  - **Cannot be rolled back** in some databases
  - May require special permissions
- **When to use**:
  - Use TRUNCATE when deleting all rows from a large table
  - Use DELETE when you need WHERE clause or want to delete specific rows
