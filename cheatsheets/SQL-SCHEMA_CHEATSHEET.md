# SQL Schema Cheatsheet

Designing and changing tables: keys, data types, time zones, `ALTER`, the
three ways to remove data, and views. Examples use the practice tables in
**`employee_db`**.

```bash
docker compose exec postgres psql -U postgres -d employee_db
```

← Back to the [cheatsheet index](./CHEATSHEET.md) ·
more constraint and foreign key examples: [SQL Basics](./SQL-BASICS_CHEATSHEET.md#constraints)

---

## Contents

- [Practice tables in employee_db](#practice-tables-in-employee_db)
- [Keys and constraints](#keys-and-constraints)
- [Choosing data types](#choosing-data-types)
- [Time zones: store UTC](#time-zones-store-utc)
- [Databases](#databases)
- [Changing data safely](#changing-data-safely)
- [ALTER TABLE](#alter-table)
- [DELETE vs TRUNCATE vs DROP](#delete-vs-truncate-vs-drop)
- [Views](#views)

---

## Practice tables in employee_db

These scripts recreate both tables as they exist now. Run them in an empty
`employee_db` (`docker compose exec postgres createdb -U postgres employee_db`).

### `employee`

```sql
CREATE TABLE employee (
    id           SERIAL PRIMARY KEY,
    emp_name     VARCHAR(50)  NOT NULL,
    emp_location VARCHAR(100) NOT NULL,
    emp_salary   INT          NOT NULL
);

INSERT INTO employee (emp_name, emp_location, emp_salary) VALUES
    ('Barry', 'Mumbai',  30000),
    ('Emily', 'Atlanta', 500000),
    ('Pete',  'Tampa',   100000);
```

### `order_details`

```sql
CREATE TABLE order_details (
    id                  SERIAL PRIMARY KEY,
    customer_id         INT          NOT NULL,
    customer_first_name VARCHAR(50)  NOT NULL,
    product_name        VARCHAR(50)  NOT NULL,
    ordered_from        VARCHAR(50)  NOT NULL,        -- 'Store' | 'App' | 'Website'
    order_amount        NUMERIC(7,2) NOT NULL,        -- money: exact decimals
    order_date          TIMESTAMPTZ  NOT NULL,
    delivery_date       TIMESTAMPTZ,
    delivery_timezone   VARCHAR(50)  DEFAULT 'UTC',   -- IANA zone name
    cancel_date         TIMESTAMPTZ
);

INSERT INTO order_details
    (customer_id, customer_first_name, product_name, ordered_from, order_amount,
     order_date, delivery_date, delivery_timezone)
VALUES
    (1001,'Linda','Pen','Store',9.82,'2025-01-02T00:00:00Z','2025-01-06T00:00:00Z','America/New_York'),
    (1002,'Stephanie','Pencil','App',12.79,'2025-01-04T00:00:00Z','2025-01-08T00:00:00Z','Europe/London'),
    (1003,'Deborah','Scissors','Website',13.68,'2025-01-07T00:00:00Z','2025-01-12T00:00:00Z','Asia/Tokyo'),
    (1004,'Andrew','Pen','Website',7.62,'2025-01-07T00:00:00Z','2025-01-15T00:00:00Z','Australia/Sydney'),
    (1005,'Steven','Chair','Website',9.76,'2025-01-07T00:00:00Z','2025-01-11T00:00:00Z','America/Los_Angeles'),
    (1006,'Susan','Pencil','Website',10.8,'2025-01-10T00:00:00Z','2025-01-13T00:00:00Z',NULL),
    (1007,'Robert','Pen','App',5.1,'2025-01-13T00:00:00Z','2025-01-23T00:00:00Z','Europe/Paris'),
    (1008,'Melissa','Eraser','Website',9.13,'2025-01-13T00:00:00Z','2025-01-17T00:00:00Z','Asia/Dubai'),
    (1009,'David','Pencil','Website',6.26,'2025-01-13T00:00:00Z','2025-01-15T00:00:00Z','America/Chicago'),
    (1010,'Jennifer','Marker','Store',5.67,'2025-01-16T00:00:00Z','2025-01-21T00:00:00Z',NULL),
    (1011,'Thomas','Pen','Store',9.04,'2025-01-16T00:00:00Z','2025-01-22T00:00:00Z','Asia/Singapore'),
    (1012,'Lisa','Binder','App',7.13,'2025-01-17T00:00:00Z','2025-01-26T00:00:00Z','Europe/Berlin'),
    (1013,'Richard','Desk','Website',15.2,'2025-01-17T00:00:00Z','2025-01-18T00:00:00Z','America/Denver'),
    (1014,'Matthew','Binder','Website',9.24,'2025-01-17T00:00:00Z','2025-01-18T00:00:00Z','Pacific/Auckland'),
    (1015,'Charles','Envelope','Website',6.58,'2025-01-17T00:00:00Z','2025-01-18T00:00:00Z','Asia/Seoul'),
    (1016,'Michelle','Envelope','Store',9.17,'2025-01-18T00:00:00Z','2025-01-21T00:00:00Z',NULL),
    (1017,'Sandra','Envelope','App',6.86,'2025-01-19T00:00:00Z','2025-01-23T00:00:00Z','America/Phoenix'),
    (1018,'Mary','Pencil','App',14.31,'2025-01-22T00:00:00Z','2025-01-29T00:00:00Z','Europe/Rome'),
    (1019,'Michael','Pen Set','Website',7.95,'2025-01-25T00:00:00Z','2025-01-29T00:00:00Z','Asia/Hong_Kong'),
    (1020,'Donald','Eraser','App',11.13,'2025-01-26T00:00:00Z','2025-01-28T00:00:00Z','America/Toronto'),
    (1021,'Amanda','Pen Set','Website',12.54,'2025-01-27T00:00:00Z','2025-01-29T00:00:00Z','Europe/Madrid'),
    (1022,'Kenneth','Pen Set','App',6.82,'2025-01-31T00:00:00Z','2025-02-03T00:00:00Z',NULL);
```

### Lessons from building `order_details`

The first version had `order_amount INT` and `customer_name`. What happened
next:

```sql
-- 1. amounts like 9.82 were silently ROUNDED to 10 by the INT column
-- 2. switch to an exact decimal type
ALTER TABLE order_details ALTER COLUMN order_amount TYPE NUMERIC(7,2);
-- 3. the stored values were already rounded, so empty the table and re-insert
TRUNCATE TABLE order_details;
-- 4. rename a column and add a new nullable one
ALTER TABLE order_details RENAME COLUMN customer_name TO customer_first_name;
ALTER TABLE order_details ADD COLUMN cancel_date TIMESTAMPTZ;
```

> **The `NULL` rows didn't get `'UTC'`.** A `DEFAULT` applies only when you
> **leave the column out** of the `INSERT`, or write `DEFAULT` in its place.
> An explicit `NULL` is stored as `NULL`. The 4 rows above with `NULL` really
> are `NULL`.
>
> ```sql
> INSERT INTO order_details (..., delivery_timezone) VALUES (..., DEFAULT);  -- 'UTC'
> ```
>
> To make NULL impossible, add `NOT NULL` to the column. To display a fallback
> instead, use `COALESCE(delivery_timezone, 'UTC')`.

Quick report on the table:

```sql
SELECT product_name,
       COUNT(*)          AS "Orders",
       SUM(order_amount) AS "Total Sales"
FROM order_details
GROUP BY product_name
ORDER BY "Total Sales" DESC;
-- Pencil 4 / 44.16, Pen 4 / 31.58, Pen Set 3 / 27.31 …
```

---

## Keys and constraints

| Constraint | Rule | Notes |
| --- | --- | --- |
| `PRIMARY KEY` | unique + not null | One per table. Always have one. |
| `FOREIGN KEY` / `REFERENCES` | value must exist in another table's key | A table can have many |
| `NOT NULL` | value required | |
| `UNIQUE` | no duplicates | NULLs are allowed (and can repeat) |
| `CHECK (…)` | value must pass a condition | e.g. `CHECK (emp_salary >= 0)` |
| `DEFAULT x` | value used when the column is left out | |

- **Parent / child:** the table with the primary key is the **parent**. The
  table holding the foreign key that points at it is the **child**
  (`customer` is the parent of `payment`).
- **Column-level** constraints sit on the column line. **Table-level** ones go
  at the end and can span several columns and have a name:

```sql
CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,                         -- column level
    employee_id INT NOT NULL,
    amount      NUMERIC(7,2) NOT NULL CHECK (amount >= 0),  -- column level
    CONSTRAINT fk_orders_employee                           -- table level, named
        FOREIGN KEY (employee_id) REFERENCES employee(id) ON DELETE CASCADE
);
```

More detail (`ON DELETE` options, bridge tables, `UNIQUE` across columns) is
in [SQL Basics → Constraints](./SQL-BASICS_CHEATSHEET.md#constraints).

---

## Choosing data types

Full list: <https://www.postgresql.org/docs/current/datatype.html>

| Need | Use | Avoid / notes |
| --- | --- | --- |
| Auto id | `SERIAL` (`BIGSERIAL` for huge tables) or `INT GENERATED ALWAYS AS IDENTITY` | |
| Whole numbers | `INT` (±2.1 billion), `BIGINT`, `SMALLINT` | |
| Money / exact decimals | `NUMERIC(p,s)` (= `DECIMAL`) | `REAL` / `FLOAT` / `DOUBLE PRECISION` are approximate (`0.1 + 0.2 ≠ 0.3`) |
| Short text with a limit | `VARCHAR(n)` | `CHAR(n)` pads with spaces |
| Any text | `TEXT` | |
| A moment in time | `TIMESTAMPTZ` | `TIMESTAMP` has no zone, so you lose the meaning |
| Calendar date only | `DATE` | |
| A duration | `INTERVAL` | |
| Yes / no | `BOOLEAN` | |
| Flexible documents | `JSONB` | `JSON` (slower to query) |
| A list in one cell | `INT[]`, `TEXT[]` | Usually a child table is better |

Tips:

- **Think about the reports you'll need.** Store dates as dates and numbers as
  numbers so you can group, sum and filter on them.
- **Leave room to grow.** `VARCHAR(50)` for a product name is fine, but don't
  make it too tight.
- **Store phone numbers, postcodes and ids with leading zeros as text.**
  `'+1-555-0100'` and `'02134'` aren't numbers. You never do math on them,
  and an `INT` would drop the zero.
- **Prefer a full timestamp over a separate date and time.** You can always
  get the date back with `::date`.

---

## Time zones: store UTC

`TIMESTAMPTZ` stores an exact moment (internally in UTC) and **displays** it
in the session's time zone. This container is already `Etc/UTC`.

```sql
SHOW timezone;                                      -- Etc/UTC
SET TIME ZONE 'UTC';                                -- this session only
ALTER DATABASE employee_db SET timezone TO 'UTC';   -- default for new sessions on this db
ALTER ROLE postgres SET timezone TO 'UTC';          -- default for this user
```

`order_details` stores the moment as UTC and keeps the customer's zone name in
`delivery_timezone`. To show the delivery in the customer's local time:

```sql
SELECT customer_first_name,
       delivery_date,                                                    -- UTC
       delivery_date AT TIME ZONE COALESCE(delivery_timezone, 'UTC') AS local_delivery
FROM order_details;
```

`timestamptz AT TIME ZONE 'Asia/Tokyo'` returns a plain timestamp showing the
clock time in Tokyo at that moment.

---

## Databases

```sql
CREATE DATABASE employee_db;
DROP DATABASE employee_db;              -- fails if anyone (incl. you) is connected to it
DROP DATABASE IF EXISTS employee_db;
```

The same from the shell: `docker compose exec postgres createdb -U postgres employee_db`
and `dropdb`. You can't drop the database you're connected to, so `\c pgdb`
first.

---

## Changing data safely

```sql
INSERT INTO employee (emp_name, emp_location, emp_salary)
VALUES ('John', 'Chicago', 0);

-- several rows in one statement
INSERT INTO employee (emp_name, emp_location, emp_salary)
VALUES ('Emily', 'Atlanta', 500000), ('Pete', 'Tampa', 100000);

-- UPDATE: always with a WHERE
UPDATE employee SET emp_name = 'Barry' WHERE id = 2;
UPDATE employee SET emp_location = 'Mumbai', emp_salary = 30000 WHERE id = 2;
```

Safe workflow for `UPDATE` / `DELETE`:

```sql
-- 1. preview the exact rows
SELECT * FROM employee WHERE emp_salary = 0;

-- 2. run it inside a transaction and look at what changed
BEGIN;
DELETE FROM employee WHERE emp_salary = 0 RETURNING *;

-- 3. happy? save it. Not happy? undo it.
COMMIT;      -- or ROLLBACK;
```

> In pgAdmin, auto-commit is on by default, so every statement is saved
> immediately. Typing `BEGIN;` first switches that off until you
> `COMMIT`/`ROLLBACK`.

---

## ALTER TABLE

| Task | SQL |
| --- | --- |
| Add a column | `ALTER TABLE employee ADD COLUMN emp_designation VARCHAR(50);` |
| Rename a column | `ALTER TABLE employee RENAME COLUMN emp_designation TO emp_type;` |
| Drop a column | `ALTER TABLE employee DROP COLUMN emp_type;` |
| Change a type | `ALTER TABLE order_details ALTER COLUMN order_amount TYPE NUMERIC(7,2);` |
| Change type with conversion | `ALTER TABLE t ALTER COLUMN c TYPE INT USING c::INT;` |
| Require a value | `ALTER TABLE employee ALTER COLUMN emp_salary SET NOT NULL;` |
| Allow NULL again | `ALTER TABLE employee ALTER COLUMN emp_salary DROP NOT NULL;` |
| Set / remove a default | `ALTER TABLE t ALTER COLUMN c SET DEFAULT 0;` / `DROP DEFAULT` |
| Add a constraint | `ALTER TABLE employee ADD CONSTRAINT chk_salary CHECK (emp_salary >= 0);` |
| Drop a constraint | `ALTER TABLE employee DROP CONSTRAINT chk_salary;` |
| Rename the table | `ALTER TABLE employee RENAME TO employees;` |

`SET NOT NULL` fails if any existing row is already NULL. Fill those rows
first with `UPDATE ... WHERE col IS NULL`.

---

## DELETE vs TRUNCATE vs DROP

| | Removes | Keeps the table? | `WHERE`? | Resets `SERIAL`? | Speed |
| --- | --- | --- | --- | --- | --- |
| `DELETE FROM t WHERE …` | chosen rows | ✅ | ✅ | ❌ | row by row |
| `DELETE FROM t` | all rows | ✅ | — | ❌ | slow on big tables |
| `TRUNCATE t` | all rows | ✅ | ❌ | ❌ (add `RESTART IDENTITY`) | instant |
| `DROP TABLE t` | rows **and** the table, its indexes and constraints | ❌ | ❌ | — | instant |

```sql
TRUNCATE TABLE order_details;                    -- next id continues where it left off
TRUNCATE TABLE order_details RESTART IDENTITY;   -- next id starts at 1 again
DROP TABLE IF EXISTS order_details;
```

### `CASCADE`

If a view or a foreign key depends on a table, `DROP` refuses:

```text
ERROR:  cannot drop table employee because other objects depend on it
DETAIL:  view v_emp depends on table employee
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
```

```sql
DROP TABLE employee CASCADE;   -- also drops the view v_emp (and FK constraints pointing at it)
```

`CASCADE` removes things without listing them all first. Read the `DETAIL`
line before you use it.

---

## Views

A **view** is a saved query you can select from like a table. It stores no
data: the query runs each time you use it. Views are good for reports and
complicated joins you'd otherwise retype.

```sql
-- CREATE
CREATE VIEW high_earners AS
SELECT emp_name, emp_location, emp_salary
FROM employee
WHERE emp_salary > 50000;

-- READ
SELECT * FROM high_earners;
SELECT * FROM high_earners WHERE emp_location = 'Tampa';   -- filter it like a table

-- UPDATE the definition
CREATE OR REPLACE VIEW high_earners AS
SELECT emp_name, emp_location, emp_salary, id           -- new columns go at the END
FROM employee
WHERE emp_salary > 50000;

-- DELETE
DROP VIEW high_earners;
DROP VIEW IF EXISTS high_earners;
```

> `CREATE OR REPLACE VIEW` can only **add columns at the end**. Removing,
> renaming or reordering existing columns gives an error. `DROP VIEW` and
> create it again instead.

See a view's SQL:

```sql
SELECT pg_get_viewdef('high_earners');                       -- or: \sv high_earners  in psql
SELECT definition FROM pg_views WHERE viewname = 'high_earners';
```

List views: `\dv` in psql, or **Schemas → public → Views** in pgAdmin.
dvdrental ships with several, such as `film_list`, `customer_list` and
`sales_by_store`.

### Example: customer segments (dvdrental)

```sql
CREATE OR REPLACE VIEW customer_segments AS
SELECT c.customer_id,
       CONCAT(c.first_name, ' ', c.last_name) AS "Customer Name",
       c.email                                AS "Email",
       SUM(p.amount)                          AS "Total Rentals",
       CASE
           WHEN SUM(p.amount) >= 200 THEN 'Elite'
           WHEN SUM(p.amount) >= 150 THEN 'Platinum'
           WHEN SUM(p.amount) >= 100 THEN 'Gold'
           ELSE 'Silver'
       END                                    AS "Member Type"
FROM payment p
JOIN customer c ON p.customer_id = c.customer_id
GROUP BY c.customer_id;

SELECT "Member Type", COUNT(*) FROM customer_segments GROUP BY 1;
```

Leave `ORDER BY` out of views. Sort when you select from the view, so each
query can pick its own order.

> This adds `customer_id` as the **first** column, so it can't replace your
> existing `customer_segments` view (see the note above). Run
> `DROP VIEW customer_segments;` first.

### Materialized views (snapshot)

A **materialized** view *does* store its results. It's fast to read, but the
data goes stale until you refresh it.

```sql
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT DATE_TRUNC('month', payment_date)::date AS month, SUM(amount) AS revenue
FROM payment
GROUP BY 1;

REFRESH MATERIALIZED VIEW monthly_revenue;   -- re-run the query and update the stored data
```
