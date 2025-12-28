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

### EXTRACT() / DATE_PART() Functions
- **Purpose**: Extract specific parts (year, month, day, hour, etc.) from a date/timestamp
- **Note**: `EXTRACT()` and `DATE_PART()` are functionally identical in PostgreSQL
- **Syntax**:
  ```sql
  -- Using EXTRACT (SQL standard)
  EXTRACT(field FROM source)

  -- Using DATE_PART (PostgreSQL-specific)
  DATE_PART('field', source)
  ```
- **Common fields**: `year`, `month`, `day`, `hour`, `minute`, `second`, `dow` (day of week)
- **Examples**:
  ```sql
  -- Filter by year
  SELECT *
  FROM cd.bookings
  WHERE EXTRACT(YEAR FROM starttime) = 2012;

  -- Filter by month and year
  SELECT *
  FROM cd.bookings
  WHERE EXTRACT(YEAR FROM starttime) = 2012
    AND EXTRACT(MONTH FROM starttime) = 9;

  -- Group by month
  SELECT EXTRACT(MONTH FROM starttime) AS month, COUNT(*) AS bookings
  FROM cd.bookings
  GROUP BY EXTRACT(MONTH FROM starttime)
  ORDER BY month;

  -- Group by facility and month
  SELECT facid,
         EXTRACT(MONTH FROM starttime) AS month,
         SUM(slots) AS total_slots
  FROM cd.bookings
  WHERE EXTRACT(YEAR FROM starttime) = 2012
  GROUP BY facid, EXTRACT(MONTH FROM starttime)
  ORDER BY facid, month;

  -- Using DATE_PART (alternative syntax)
  SELECT DATE_PART('year', starttime) AS year,
         DATE_PART('month', starttime) AS month
  FROM cd.bookings;
  ```
- **Date range filtering**:
  ```sql
  -- Method 1: Using EXTRACT
  WHERE EXTRACT(YEAR FROM starttime) = 2012
    AND EXTRACT(MONTH FROM starttime) = 9

  -- Method 2: Using date range (often more efficient)
  WHERE starttime >= '2012-09-01' AND starttime < '2012-10-01'
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

### LIMIT and OFFSET
- **Purpose**: Limit the number of rows returned and optionally skip rows (pagination)
- **Syntax**:
  ```sql
  -- Limit to N rows
  SELECT columns
  FROM table_name
  LIMIT N;

  -- Skip M rows, then return N rows
  SELECT columns
  FROM table_name
  LIMIT N OFFSET M;

  -- Alternative syntax (OFFSET first)
  SELECT columns
  FROM table_name
  OFFSET M ROWS FETCH FIRST N ROWS ONLY;  -- SQL standard syntax
  ```
- **Examples**:
  ```sql
  -- Get top 5 facilities by slots booked
  SELECT facid, SUM(slots) AS total_slots
  FROM cd.bookings
  GROUP BY facid
  ORDER BY total_slots DESC
  LIMIT 5;

  -- Get the single facility with most bookings
  SELECT facid, SUM(slots) AS total_slots
  FROM cd.bookings
  GROUP BY facid
  ORDER BY total_slots DESC
  LIMIT 1;

  -- Pagination: get rows 11-20 (skip first 10, get next 10)
  SELECT surname, firstname
  FROM cd.members
  ORDER BY surname
  LIMIT 10 OFFSET 10;

  -- Get rows 21-30
  SELECT surname, firstname
  FROM cd.members
  ORDER BY surname
  LIMIT 10 OFFSET 20;
  ```
- **Important notes**:
  - LIMIT is evaluated AFTER ORDER BY
  - Always use ORDER BY with LIMIT to ensure consistent results
  - Without ORDER BY, the rows returned are unpredictable
  - LIMIT is PostgreSQL/MySQL syntax; SQL standard uses FETCH FIRST

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

#### Subquery Comparison Operators (ALL, ANY, SOME)
- **Purpose**: Compare a value against all or any values returned by a subquery
- **Operators**:
  - `ALL` - true if comparison is true for ALL values in the subquery
  - `ANY` - true if comparison is true for ANY (at least one) value in the subquery
  - `SOME` - same as ANY (just a synonym)
- **Syntax**:
  ```sql
  -- ALL: value must satisfy condition for ALL subquery results
  SELECT columns
  FROM table_name
  WHERE column >= ALL (subquery);

  -- ANY/SOME: value must satisfy condition for ANY subquery result
  SELECT columns
  FROM table_name
  WHERE column > ANY (subquery);
  ```
- **Examples**:
  ```sql
  -- Find facility with most slots booked (without LIMIT)
  SELECT facid, SUM(slots) AS total_slots
  FROM cd.bookings
  GROUP BY facid
  HAVING SUM(slots) >= ALL (
      SELECT SUM(slots)
      FROM cd.bookings
      GROUP BY facid
  );

  -- Find facilities with more slots than Tennis Court 1
  SELECT name
  FROM cd.facilities
  WHERE facid IN (
      SELECT facid
      FROM cd.bookings
      GROUP BY facid
      HAVING SUM(slots) > ALL (
          SELECT SUM(slots)
          FROM cd.bookings
          WHERE facid = 0  -- Tennis Court 1
      )
  );

  -- Find members who joined later than ANY member from 2012
  SELECT surname, firstname, joindate
  FROM cd.members
  WHERE joindate > ANY (
      SELECT joindate
      FROM cd.members
      WHERE EXTRACT(YEAR FROM joindate) = 2012
  );

  -- Alternative to MAX using ALL
  SELECT MAX(monthlymaintenance) FROM cd.facilities;
  -- is equivalent to:
  SELECT monthlymaintenance
  FROM cd.facilities
  WHERE monthlymaintenance >= ALL (
      SELECT monthlymaintenance FROM cd.facilities
  );
  ```
- **Common equivalents**:
  - `= ANY(subquery)` is the same as `IN (subquery)`
  - `<> ALL(subquery)` is the same as `NOT IN (subquery)`
  - `> ALL(subquery)` finds values greater than the maximum
  - `< ALL(subquery)` finds values less than the minimum
  - `> ANY(subquery)` finds values greater than the minimum
  - `< ANY(subquery)` finds values less than the maximum
- **Note**: ALL/ANY/SOME are useful but often less readable than using MAX/MIN or LIMIT. Use them when you specifically need to avoid those approaches.

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

### GROUP BY and Aggregate Functions
- **Purpose**: Combine rows with the same values in specified columns and perform calculations on each group
- **Common aggregate functions**: `SUM()`, `COUNT()`, `AVG()`, `MAX()`, `MIN()`
- **Syntax**:
  ```sql
  SELECT column1, aggregate_function(column2)
  FROM table_name
  GROUP BY column1;
  ```
- **The Rule**: Use GROUP BY when you mix aggregate columns with non-aggregate columns
  ```sql
  -- ✅ CORRECT: All non-aggregated columns are in GROUP BY
  SELECT facid, SUM(slots)
  FROM cd.bookings
  GROUP BY facid;

  -- ✅ CORRECT: Only aggregated column, no GROUP BY needed
  SELECT SUM(slots)
  FROM cd.bookings;

  -- ❌ ERROR: facid is not aggregated and not in GROUP BY
  SELECT facid, SUM(slots)
  FROM cd.bookings;
  ```
- **Examples**:
  ```sql
  -- Count recommendations per member
  SELECT recommendedby, COUNT(*) AS count
  FROM cd.members
  WHERE recommendedby IS NOT NULL
  GROUP BY recommendedby
  ORDER BY recommendedby;

  -- Total slots booked per facility
  SELECT facid, SUM(slots) AS total_slots
  FROM cd.bookings
  GROUP BY facid
  ORDER BY facid;

  -- Average cost per facility
  SELECT facid, AVG(membercost) AS avg_cost
  FROM cd.facilities
  GROUP BY facid;

  -- Group by multiple columns
  SELECT facid, memid, SUM(slots) AS total_slots
  FROM cd.bookings
  GROUP BY facid, memid
  ORDER BY facid, memid;
  ```
- **Think of it this way**: GROUP BY creates separate "piles" for each unique value, then calculates aggregates for each pile

### HAVING Clause
- **Purpose**: Filter groups AFTER aggregation (whereas WHERE filters rows BEFORE aggregation)
- **SQL Execution Order**:
  ```
  1. FROM       - Get the table(s)
  2. WHERE      - Filter individual rows (BEFORE grouping)
  3. GROUP BY   - Group rows
  4. Aggregates - Calculate SUM(), COUNT(), AVG(), etc.
  5. HAVING     - Filter groups (AFTER aggregation)
  6. SELECT     - Choose columns to return
  7. ORDER BY   - Sort results
  ```
- **The Rule**:
  - Use **WHERE** to filter rows before grouping (e.g., `WHERE starttime > '2012-01-01'`)
  - Use **HAVING** to filter groups after aggregation (e.g., `HAVING SUM(slots) > 1000`)
- **Syntax**:
  ```sql
  SELECT column1, aggregate_function(column2)
  FROM table_name
  WHERE condition_on_rows          -- Filter rows BEFORE grouping
  GROUP BY column1
  HAVING condition_on_aggregates   -- Filter groups AFTER aggregation
  ORDER BY column1;
  ```
- **Examples**:
  ```sql
  -- ❌ WRONG: Can't use aggregate in WHERE
  SELECT facid, SUM(slots) AS total_slots
  FROM cd.bookings
  WHERE SUM(slots) > 1000  -- ERROR: aggregate functions not allowed in WHERE
  GROUP BY facid;

  -- ✅ CORRECT: Use HAVING for aggregate filtering
  SELECT facid, SUM(slots) AS total_slots
  FROM cd.bookings
  GROUP BY facid
  HAVING SUM(slots) > 1000  -- Filter groups after aggregation
  ORDER BY facid;

  -- ✅ Using both WHERE and HAVING
  SELECT facid, SUM(slots) AS total_slots
  FROM cd.bookings
  WHERE starttime >= '2012-09-01'  -- Filter rows first (before grouping)
  GROUP BY facid
  HAVING SUM(slots) > 1000         -- Filter groups after aggregation
  ORDER BY total_slots DESC;

  -- Filter facilities with average member cost > 10
  SELECT facid, AVG(membercost) AS avg_cost
  FROM cd.facilities
  GROUP BY facid
  HAVING AVG(membercost) > 10;

  -- Count members per recommender, show only those who recommended 5+
  SELECT recommendedby, COUNT(*) AS count
  FROM cd.members
  WHERE recommendedby IS NOT NULL
  GROUP BY recommendedby
  HAVING COUNT(*) >= 5
  ORDER BY count DESC;
  ```
- **Performance tip**: Use WHERE to filter as early as possible to reduce the data being grouped
- **IMPORTANT - Column aliases in HAVING**:
  - **You CANNOT use column aliases in HAVING** - you must repeat the full aggregate expression
  - Column aliases work in ORDER BY (evaluated last) but NOT in HAVING or WHERE
  ```sql
  -- ❌ WRONG: Can't use alias 'revenue' in HAVING
  SELECT f.name, SUM(
      CASE WHEN b.memid = 0 THEN f.guestcost * b.slots
      ELSE f.membercost * b.slots END
  ) AS revenue
  FROM cd.facilities f
  JOIN cd.bookings b ON b.facid = f.facid
  GROUP BY f.name
  HAVING revenue < 1000;  -- ERROR: column "revenue" does not exist

  -- ✅ CORRECT: Repeat the full expression in HAVING
  SELECT f.name, SUM(
      CASE WHEN b.memid = 0 THEN f.guestcost * b.slots
      ELSE f.membercost * b.slots END
  ) AS revenue
  FROM cd.facilities f
  JOIN cd.bookings b ON b.facid = f.facid
  GROUP BY f.name
  HAVING SUM(
      CASE WHEN b.memid = 0 THEN f.guestcost * b.slots
      ELSE f.membercost * b.slots END
  ) < 1000
  ORDER BY revenue;  -- ✅ Alias works here in ORDER BY

  -- ✅ ALTERNATIVE: Use subquery to avoid repeating expression
  SELECT name, revenue
  FROM (
      SELECT f.name, SUM(
          CASE WHEN b.memid = 0 THEN f.guestcost * b.slots
          ELSE f.membercost * b.slots END
      ) AS revenue
      FROM cd.facilities f
      JOIN cd.bookings b ON b.facid = f.facid
      GROUP BY f.name
  ) AS subquery
  WHERE revenue < 1000  -- Now you can use the alias
  ORDER BY revenue;
  ```

### ROLLUP
- **Purpose**: Generate hierarchical subtotals and grand totals in GROUP BY queries
- **What it does**: Creates grouping sets from right to left, adding NULL for each level of aggregation
- **Syntax**:
  ```sql
  SELECT column1, column2, aggregate_function(column3)
  FROM table_name
  GROUP BY ROLLUP (column1, column2);
  ```
- **How ROLLUP works**:
  ```sql
  -- ROLLUP (col1, col2) creates these grouping sets:
  -- 1. GROUP BY col1, col2    (most detailed)
  -- 2. GROUP BY col1           (col2 becomes NULL)
  -- 3. GROUP BY ()             (grand total - both become NULL)
  ```
- **Examples**:
  ```sql
  -- Monthly booking totals with facility subtotals and grand total
  SELECT facid,
         EXTRACT(MONTH FROM starttime) AS month,
         SUM(slots) AS slots
  FROM cd.bookings
  WHERE EXTRACT(YEAR FROM starttime) = 2012
  GROUP BY ROLLUP(facid, EXTRACT(MONTH FROM starttime))
  ORDER BY facid, month;

  -- Output structure:
  -- facid | month | slots
  -- ------+-------+-------
  --   0   |   1   |  100   (facility 0, January)
  --   0   |   2   |  150   (facility 0, February)
  --   ...
  --   0   | NULL  |  1500  (facility 0 total - all months)
  --   1   |   1   |  200   (facility 1, January)
  --   ...
  --   1   | NULL  |  2000  (facility 1 total - all months)
  --  NULL | NULL  |  5000  (grand total - all facilities, all months)

  -- Member recommendations with subtotals
  SELECT recommendedby,
         memid,
         COUNT(*) AS recommendations
  FROM cd.members
  WHERE recommendedby IS NOT NULL
  GROUP BY ROLLUP(recommendedby, memid)
  ORDER BY recommendedby, memid;

  -- Sales by region and product with totals
  SELECT region,
         product,
         SUM(revenue) AS total_revenue
  FROM sales
  GROUP BY ROLLUP(region, product)
  ORDER BY region, product;
  -- Shows: each region-product combo, region totals, and grand total
  ```
- **Important notes**:
  - **Order matters**: ROLLUP creates hierarchy from left to right
    - `ROLLUP(year, month)` gives: (year, month), (year), ()
    - `ROLLUP(month, year)` gives: (month, year), (month), ()
  - **NULLs indicate aggregation levels**: NULL values show where subtotals occur
  - **Sorting with NULLs**: PostgreSQL sorts NULLs last by default, which works well for ROLLUP
  - **Common mistake**: Forgetting the year filter can include wrong data in totals
- **Related functions**:
  - `CUBE(col1, col2)` - Creates all possible combinations of grouping sets
  - `GROUPING SETS(...)` - Manually specify which grouping sets to create
  - `GROUPING(column)` - Returns 1 if column is NULL due to ROLLUP/CUBE, 0 otherwise
