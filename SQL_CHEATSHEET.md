# PostgreSQL SQL Cheatsheet

Quick reference for everyday Postgres SQL. Run statements in the pgAdmin Query
Tool (**F5** to execute) or in `psql`.

**Quoting rule:** string values use **single quotes** (`'Mike'`). Double quotes
(`"..."`) are for identifiers like table/column names — not text values.

> **Config values:** examples use this project's default database `pgdb` and the
> `users` / `people` tables. Table names, columns, and data are just examples —
> swap them for your own. See `docker-compose.yaml` / `.env` for the database
> name and credentials. No credentials appear below.

---

## Spin up first

Start the stack, then run these queries in pgAdmin or psql:

```bash
cp .env.example .env                                  # first time only
docker compose up -d                                  # start Postgres + pgAdmin
docker compose exec postgres psql -U postgres -d pgdb # open a SQL prompt
```

pgAdmin is at <http://localhost:5050>. See the [README](./README.md) for setup
and [`PSQL-CLI_CHEATSHEET.md`](./PSQL-CLI_CHEATSHEET.md) for the psql client.

---

## Common data types

| Type              | Use for                                   | Example value                  |
| ----------------- | ----------------------------------------- | ------------------------------ |
| `INT` / `INTEGER` | Whole numbers                             | `42`                           |
| `SERIAL`          | Auto-incrementing integer (great for IDs) | `1, 2, 3...` (auto)            |
| `VARCHAR(n)`      | Text with a max length                    | `'hello'`                      |
| `TEXT`            | Text of any length                        | `'a long string'`             |
| `BOOLEAN` / `BOOL`| True / false                              | `TRUE`, `FALSE`                |
| `NUMERIC(p,s)`    | Exact decimals (money, etc.)              | `19.99`                        |
| `DATE`            | Date only                                 | `2026-07-16`                   |
| `TIMESTAMP`       | Date + time (no time zone)                | `2026-07-16 11:25:39.123456`   |
| `TIMESTAMPTZ`     | Date + time **with** time zone (preferred)| `2026-07-16 11:25:39.123456+03`|

### Dates, time & time zones (UTC vs local)

Short answer: it depends on the type **and** the session's `TimeZone` setting.

- **`DATE`** — a calendar date only. No time, no zone.
- **`TIMESTAMP`** (a.k.a. *without time zone*) — stores exactly the numbers you
  give it. **No zone awareness, no conversion** — it's just a "wall clock"
  reading. You are responsible for knowing whether it means UTC or local.
- **`TIMESTAMPTZ`** — despite the name, it does **not** store a zone. On INSERT
  it interprets your input using the session's time zone, converts it to
  **UTC**, and stores that UTC instant. On SELECT it converts back to the
  session's time zone for display. So: **stored as UTC, shown as local.**

The "local" zone comes from the session parameter `TimeZone`, not directly from
your OS:

```sql
SHOW timezone;          -- what zone this session uses
SELECT now();           -- current timestamptz (shown in session zone)
SELECT now() AT TIME ZONE 'UTC';   -- same instant expressed as UTC
SET timezone = 'America/New_York'; -- change it for this session
```

> **This project:** the `postgres` Docker image defaults to **`Etc/UTC`**, *not*
> your machine's zone. So even on a UTC+3 laptop, `now()` prints UTC time (e.g.
> `10:50+00` while your Mac clock says `13:50`). Timestamps here are effectively
> UTC unless you change `TimeZone`.

**Rule of thumb:** prefer **`TIMESTAMPTZ`** for anything that marks a real moment
in time (created_at, logins, events). It keeps everything anchored to UTC
internally and displays correctly for any user's zone. Use plain `TIMESTAMP`
only for zone-less "wall clock" values (e.g. a store's 9:00 AM opening time).

---

## Constraints

| Constraint     | Meaning                                              |
| -------------- | --------------------------------------------------- |
| `PRIMARY KEY`  | Uniquely identifies each row (auto `NOT NULL` + `UNIQUE`) |
| `FOREIGN KEY`  | Points to a primary key in another table            |
| `NOT NULL`     | Value is required (can't be empty/NULL)             |
| `UNIQUE`       | No two rows can share this value                    |
| `DEFAULT <x>`  | Value used automatically when none is given         |
| `CHECK (...)`  | Value must satisfy a condition                       |

**NULL vs NOT NULL:** `NULL` means "no value / unknown". Columns allow `NULL` by
default; add `NOT NULL` to make a column required. Rule of thumb — ask *"can a
row legitimately exist without this value?"* If no, use `NOT NULL`.

### The `UNIQUE` constraint

`UNIQUE` stops two rows from sharing the same value in a column — good for
emails, usernames, SKUs, etc. Attempting a duplicate `INSERT`/`UPDATE` fails.

```sql
-- inline on a single column
CREATE TABLE users (
    id    SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE   -- no two users can share an email
);

-- multi-column (UNIQUE together): the COMBINATION must be unique,
-- so the same room can be booked on different days
CREATE TABLE bookings (
    room_id INT NOT NULL,
    day     DATE NOT NULL,
    UNIQUE (room_id, day)
);

-- add a UNIQUE constraint to an existing table
ALTER TABLE users ADD CONSTRAINT uq_users_email UNIQUE (email);
```

Notes & gotchas:

- **`NULL`s are exempt.** `NULL` is treated as "unknown", so a `UNIQUE` column
  can hold many `NULL` rows. Add `NOT NULL` too if you also want the value
  required.
- `UNIQUE` pairs with `ON CONFLICT` for upserts — it's the constraint that
  detects the conflict (see [Insert only if not already there](#insert-only-if-not-already-there-avoid-duplicates)).
- `PRIMARY KEY` is essentially `UNIQUE` + `NOT NULL`; a table has one primary key
  but can have many `UNIQUE` constraints.

### The `CHECK` constraint

`CHECK (...)` rejects any row where the condition isn't true — it guards the
*content* of a value (a range, a set of allowed strings, a relationship between
columns), not just its type. A failing `INSERT`/`UPDATE` errors out.

```sql
CREATE TABLE products (
    id      SERIAL PRIMARY KEY,
    name    VARCHAR(100) NOT NULL,
    price   DECIMAL(10,2) NOT NULL CHECK (price >= 0),        -- no negative prices
    stock   INT NOT NULL DEFAULT 0 CHECK (stock >= 0),
    -- named, multi-column check (can compare columns to each other)
    sale_price DECIMAL(10,2),
    CONSTRAINT chk_sale CHECK (sale_price IS NULL OR sale_price <= price)
);

-- restrict to a set of allowed values
CREATE TABLE people (
    id     SERIAL PRIMARY KEY,
    age    INT CHECK (age BETWEEN 0 AND 150),
    status VARCHAR(20) CHECK (status IN ('active', 'inactive', 'pending'))
);

-- add a CHECK to an existing table
ALTER TABLE people ADD CONSTRAINT chk_age CHECK (age >= 0);
ALTER TABLE people DROP CONSTRAINT chk_age;   -- remove it by name
```

Notes & gotchas:

- **`NULL` passes.** A `CHECK` only fails on a definitively *false* result; if the
  column is `NULL` the condition is "unknown" and the row is allowed. Add
  `NOT NULL` if the value is also required.
- Name the constraint (`CONSTRAINT chk_… CHECK (…)`) so error messages are clear
  and you can drop it later by name.
- Use it for simple invariants (ranges, allowed sets, column relationships). It
  can't reference other tables — use a `FOREIGN KEY` for that.

### All together: `CHECK` + `FOREIGN KEY` + `REFERENCES`

A single table often uses several constraints at once — a foreign key to relate
it to a parent, plus checks to keep its own values sane:

```sql
CREATE TABLE orders (
    id       SERIAL PRIMARY KEY,
    -- FOREIGN KEY: user_id must point at an existing users row
    user_id  INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    quantity INT NOT NULL CHECK (quantity > 0),           -- CHECK: at least 1
    total    DECIMAL(10,2) NOT NULL CHECK (total >= 0),   -- CHECK: not negative
    status   VARCHAR(20) NOT NULL DEFAULT 'pending'
             CHECK (status IN ('pending', 'paid', 'shipped', 'cancelled'))
);
```

The same constraints written as **named, table-level** clauses (identical
effect, but easier to reference and drop later):

```sql
CREATE TABLE orders (
    id       SERIAL PRIMARY KEY,
    user_id  INT NOT NULL,
    quantity INT NOT NULL,
    total    DECIMAL(10,2) NOT NULL,
    CONSTRAINT fk_user    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT chk_qty    CHECK (quantity > 0),
    CONSTRAINT chk_total  CHECK (total >= 0)
);
```

> Inline `REFERENCES users(id)` and the table-level `FOREIGN KEY (user_id)
> REFERENCES users(id)` do the same thing — table-level just lets you name it
> and span multiple columns. `CHECK` guards this table's own values; the
> `FOREIGN KEY` guards the link to another table. They combine freely.

---

## Create tables

```sql
CREATE TABLE users (
    id         SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name  VARCHAR(100) NOT NULL,
    email      VARCHAR(255) NOT NULL UNIQUE,
    is_active  BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()  -- current date+time, ISO-8601 style
);
```

- `SERIAL PRIMARY KEY` — auto-incrementing unique ID (no need to supply it).
- `NOW()` — current **date + time**, ISO-8601 format `YYYY-MM-DD HH:MM:SS.ffffff`.
  It returns a `timestamptz`; use a `TIMESTAMPTZ` column to keep the time zone.

Simpler table (plain `INT` id must be supplied manually on insert):

```sql
CREATE TABLE IF NOT EXISTS people (
    id         INT PRIMARY KEY,
    first_name VARCHAR(255),
    last_name  VARCHAR(255),
    age        INT,
    job        VARCHAR(255),
    employed   BOOL
);
```

`IF NOT EXISTS` avoids an error if the table already exists.

---

## Insert (add rows)

```sql
-- Full row: list every value in column order
INSERT INTO people VALUES (1, 'Mike', 'Jones', 42, 'Developer', TRUE);

-- Named columns (safer): order doesn't matter, skip nullable columns
INSERT INTO people (id, first_name, last_name)
VALUES (2, 'Sara', 'Lee');

-- Multiple rows at once
INSERT INTO people (id, first_name, age) VALUES
    (3, 'Ada',   36),
    (4, 'Grace', 45);

-- Return the inserted row(s)
INSERT INTO people (id, first_name) VALUES (5, 'Alan') RETURNING *;
```

### With a SERIAL `id` (auto-increment)

If `id` is `SERIAL`, **don't supply it** — name the other columns and let
Postgres generate it. Positional `VALUES (...)` still expects *every* column
(including `id`), so use the named form:

```sql
-- one row (id auto-generated)
INSERT INTO people (first_name, last_name, age, job, employed)
VALUES ('Mike', 'Jones', 42, 'Developer', TRUE);

-- multiple rows at once (id auto-generated for each)
INSERT INTO people (first_name, last_name, age, job, employed) VALUES
    ('Sara', 'Lee',   29, 'Designer',  TRUE),
    ('Ada',  NULL,    36, NULL,        FALSE),
    ('Guy',  NULL,    NULL, NULL,      TRUE);
```

> Gotcha: `INSERT INTO people VALUES ('Mike', ...)` fails with
> `invalid input syntax for type integer` because the first value ('Mike') is
> matched to `id`. Always name columns when using `SERIAL`.

### Insert only if not already there (avoid duplicates)

Requires a `UNIQUE` or `PRIMARY KEY` column to detect the conflict.

```sql
-- skip the insert if a row with the same id already exists
INSERT INTO people (id, first_name) VALUES (1, 'Mike')
ON CONFLICT (id) DO NOTHING;

-- or: insert, but if it already exists, update instead ("upsert")
INSERT INTO people (id, first_name) VALUES (1, 'Michael')
ON CONFLICT (id) DO UPDATE SET first_name = EXCLUDED.first_name;

-- insert many, skipping any that conflict
INSERT INTO people (id, first_name) VALUES
    (1, 'Mike'),
    (2, 'Sara'),
    (3, 'Ada')
ON CONFLICT (id) DO NOTHING;
```

`EXCLUDED` refers to the row you tried to insert, so `DO UPDATE` can copy its
values into the existing row.

---

## Select (read rows) — beginner to intermediate

Builds up step by step. Examples use the `people` table.

### 1. The basics — pick rows and columns

```sql
SELECT * FROM people;                  -- every column, every row
SELECT first_name, age FROM people;    -- only the columns you name
SELECT first_name AS name FROM people; -- rename a column in the output (alias)
```

### 2. Find one row by id (`WHERE`)

`WHERE` filters which rows come back. Start with the primary key:

```sql
SELECT * FROM people WHERE id = 1;     -- the single row with id 1
SELECT first_name FROM people WHERE id = 3;
```

### 3. Compare values (operators)

```sql
SELECT * FROM people WHERE age = 42;    -- equals
SELECT * FROM people WHERE age <> 42;   -- not equal (also: !=)
SELECT * FROM people WHERE age > 40;    -- greater than
SELECT * FROM people WHERE age >= 40;   -- greater than or equal
SELECT * FROM people WHERE age < 30;    -- less than
SELECT * FROM people WHERE employed = TRUE;   -- booleans
```

### 4. Combine conditions with `AND` / `OR`

```sql
-- BOTH must be true
SELECT * FROM people
WHERE age >= 18 AND employed = TRUE;

-- EITHER can be true
SELECT * FROM people
WHERE job = 'Developer' OR job = 'Manager';

-- NOT flips a condition
SELECT * FROM people
WHERE NOT employed = TRUE;   -- i.e. not employed
```

Use **parentheses** when mixing `AND` and `OR` — `AND` binds tighter than `OR`,
so group to make intent clear:

```sql
-- "developers OR managers, who are also over 40"
SELECT * FROM people
WHERE (job = 'Developer' OR job = 'Manager')
  AND age > 40;
```

### 5. Handy condition shortcuts

```sql
SELECT * FROM people WHERE job IN ('Developer', 'Manager');  -- one of a list
SELECT * FROM people WHERE age BETWEEN 18 AND 65;            -- inclusive range
SELECT * FROM people WHERE first_name LIKE 'A%';   -- starts with A (case-sensitive)
SELECT * FROM people WHERE first_name ILIKE 'a%';  -- starts with A/a (any case)
SELECT * FROM people WHERE first_name LIKE '%son'; -- ends with 'son'
SELECT * FROM people WHERE first_name LIKE '%i%';  -- contains 'i'
```

`%` = any number of characters, `_` = exactly one character.

### 5a. Pattern matching (`LIKE`) — starts with / ends with / contains

Use `LIKE` with wildcards to match parts of text. The `%` wildcard stands for
"any characters (including none)":

```sql
-- last name STARTS WITH 'J'  (J at the start, anything after)
SELECT * FROM people WHERE last_name LIKE 'J%';

-- last name ENDS WITH 's'  (anything before, s at the end)
SELECT * FROM people WHERE last_name LIKE '%s';

-- last name CONTAINS 'on'  (anywhere in the string)
SELECT * FROM people WHERE last_name LIKE '%on%';

-- exact length with '_' : first name is exactly 3 letters
SELECT * FROM people WHERE first_name LIKE '___';

-- starts with 'J' AND is 5 characters long ('J' + four more)
SELECT * FROM people WHERE first_name LIKE 'J____';
```

Read the wildcards as:

| Pattern   | Matches                          | Example matches      |
| --------- | -------------------------------- | -------------------- |
| `'J%'`    | starts with J                    | Jones, Jackson, J    |
| `'%s'`    | ends with s                      | Jones, Adams, s      |
| `'%on%'`  | contains "on" anywhere           | Jones, Thompson      |
| `'_at'`   | one char, then "at" (3 chars)    | cat, bat, hat        |
| `'%'`     | anything (even empty)            | everything           |

**Case sensitivity:** `LIKE` is case-**sensitive** (`'j%'` won't match "Jones").
Use `ILIKE` for case-**insensitive** matching:

```sql
SELECT * FROM people WHERE last_name ILIKE 'j%';   -- matches Jones AND jones
```

> Tip: to match a literal `%` or `_` in your text, escape it:
> `WHERE note LIKE '100\%'` matches the string "100%".

### 5b. Match a list of values (`IN`)

`IN` tests whether a column matches **any** value in a list — a shorthand for a
chain of `OR` conditions.

```sql
-- these two are equivalent
SELECT * FROM people WHERE job IN ('Developer', 'Manager', 'Designer');
SELECT * FROM people WHERE job = 'Developer' OR job = 'Manager' OR job = 'Designer';

-- works with numbers too
SELECT * FROM people WHERE id IN (1, 3, 5);

-- NOT IN — match anything NOT in the list
SELECT * FROM people WHERE job NOT IN ('Developer', 'Manager');
```

You can also feed `IN` the results of a subquery instead of a hand-written list:

```sql
-- people who have placed at least one order
SELECT * FROM people
WHERE id IN (SELECT user_id FROM orders);
```

> **NULL gotcha:** `NOT IN` returns *no rows* if the list (or subquery) contains
> a `NULL`, because `x <> NULL` is never true. Filter NULLs out first, or prefer
> `NOT EXISTS` for subqueries. Plain `IN` is unaffected.

### 6. Working with NULL

NULL means "unknown", so you can't compare it with `=`. Use `IS`:

```sql
SELECT * FROM people WHERE job IS NULL;      -- no job set
SELECT * FROM people WHERE job IS NOT NULL;  -- has a job
```

### 7. Sort, limit, de-duplicate

```sql
SELECT * FROM people ORDER BY age DESC;      -- highest age first (ASC is default)
SELECT * FROM people ORDER BY job, age DESC; -- by job, then age within each job
SELECT * FROM people LIMIT 5;                -- first 5 rows
SELECT * FROM people ORDER BY age DESC LIMIT 3;  -- top 3 oldest
SELECT * FROM people LIMIT 5 OFFSET 10;      -- skip 10, then take 5 (paging)
SELECT DISTINCT job FROM people;             -- unique job values only
```

### 7a. Sorting results (`ORDER BY`)

`ORDER BY` sorts the returned rows. `ASC` (ascending, the default) goes
low→high / A→Z; `DESC` goes high→low / Z→A.

```sql
-- single column
SELECT * FROM people ORDER BY age;        -- youngest first (ASC is the default)
SELECT * FROM people ORDER BY age DESC;   -- oldest first
SELECT * FROM people ORDER BY last_name;  -- A→Z by last name

-- multiple columns: sort by the first, then break ties with the next
SELECT * FROM people ORDER BY job, age DESC;       -- by job A→Z, oldest first within each job
SELECT * FROM people ORDER BY job ASC, age DESC;   -- same thing, ASC spelled out
SELECT * FROM people ORDER BY last_name, first_name, age DESC;  -- 3 levels of tie-breaking

-- sort by a column you didn't select, or by an alias
SELECT first_name, age AS years FROM people ORDER BY years DESC;

-- top / bottom N: combine with LIMIT
SELECT * FROM people ORDER BY age DESC LIMIT 3;   -- 3 oldest people

-- where NULLs land (they sort last by default in ASC; flip with NULLS FIRST)
SELECT * FROM people ORDER BY job ASC NULLS FIRST;
```

> Tip: you can also sort by column position — `ORDER BY 2` sorts by the 2nd
> selected column — but naming the column is clearer.

### 8. Putting it together

A fuller query showing the usual clause order:

```sql
SELECT first_name, age, job
FROM people
WHERE employed = TRUE
  AND age BETWEEN 25 AND 50
ORDER BY age DESC
LIMIT 10;
```

> Clause order matters: `SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ...
> ORDER BY ... LIMIT`.

### Removing duplicates with `DISTINCT`

`DISTINCT` removes duplicate rows from the result, returning only unique values.
It applies to **all the columns you select**, considered together.

```sql
-- every job value, including repeats
SELECT job FROM people;

-- each job listed only once
SELECT DISTINCT job FROM people;

-- unique COMBINATIONS of job + employed (a pair is a duplicate only if
-- BOTH columns match another row)
SELECT DISTINCT job, employed FROM people;

-- count how many DIFFERENT jobs exist
SELECT COUNT(DISTINCT job) FROM people;

-- often paired with ORDER BY to get a clean sorted list
SELECT DISTINCT job FROM people ORDER BY job;
```

Notes & gotchas:

- `DISTINCT` looks at **every selected column**. `SELECT DISTINCT first_name, job`
  keeps a row unless *both* values repeat — so you may still see repeated
  `first_name` values.
- `NULL`s are treated as equal to each other, so all NULLs collapse into a
  single row.
- To find which values actually repeat (and how often), use `GROUP BY` +
  `COUNT` instead:

  ```sql
  SELECT job, COUNT(*)
  FROM people
  GROUP BY job
  HAVING COUNT(*) > 1;    -- only jobs that appear more than once
  ```

- Postgres also has `DISTINCT ON (col)` to keep the first row per value:

  ```sql
  -- one (oldest) person per job
  SELECT DISTINCT ON (job) job, first_name, age
  FROM people
  ORDER BY job, age DESC;
  ```

### Aggregates & grouping

Aggregate functions collapse many rows into one summary value:

```sql
SELECT COUNT(*), AVG(age), MIN(age), MAX(age), SUM(age) FROM people;
```

| Function   | Returns                          |
| ---------- | -------------------------------- |
| `COUNT(*)` | number of rows                   |
| `AVG(col)` | average (mean) of the values     |
| `MIN(col)` | smallest value                   |
| `MAX(col)` | largest value                    |
| `SUM(col)` | total of the values              |

### Average, min & max (`AVG`, `MIN`, `MAX`)

```sql
-- summary stats for one column, with friendly names
SELECT MIN(age) AS youngest,
       MAX(age) AS oldest,
       AVG(age) AS average_age
FROM people;

-- round the average (AVG returns a long decimal)
SELECT ROUND(AVG(age), 1) AS avg_age FROM people;

-- stats only over the rows you care about (WHERE runs first)
SELECT AVG(age) FROM people WHERE employed = TRUE;

-- per group: youngest / oldest / average within each job
SELECT job, MIN(age), MAX(age), ROUND(AVG(age), 1) AS avg_age
FROM people
GROUP BY job;
```

> `NULL`s are ignored by these functions — `AVG(age)` averages only rows that
> have an age. `MIN`/`MAX` also work on text (`MIN(last_name)` = alphabetically
> first) and dates (`MAX(created_at)` = most recent).

### Count per group (`SELECT x, COUNT(*) ... GROUP BY x`)

Pair a column with an aggregate and `GROUP BY` that column to get one row per
distinct value — e.g. how many people hold each job:

```sql
-- how many people per job
SELECT job, COUNT(*)
FROM people
GROUP BY job;

-- rename the count column and sort by it (biggest group first)
SELECT job, COUNT(*) AS headcount
FROM people
GROUP BY job
ORDER BY headcount DESC;

-- mix in other aggregates per group
SELECT job, COUNT(*), AVG(age), MAX(age)
FROM people
GROUP BY job;

-- filter groups with HAVING (WHERE filters rows, HAVING filters groups)
SELECT job, COUNT(*)
FROM people
GROUP BY job
HAVING COUNT(*) > 1;     -- only jobs shared by more than one person

-- filter rows first, then group (WHERE runs before grouping)
SELECT job, COUNT(*)
FROM people
WHERE employed = TRUE
GROUP BY job;
```

> Rule: every column in the `SELECT` that isn't inside an aggregate **must**
> appear in `GROUP BY`. Order: `WHERE` (filter rows) → `GROUP BY` → `HAVING`
> (filter groups) → `ORDER BY`.

### Group by multiple columns

List several columns in `GROUP BY` to get one row per **combination** of their
values — e.g. a count for each job/employed pairing:

```sql
-- one row per (job, employed) combination
SELECT job, employed, COUNT(*)
FROM people
GROUP BY job, employed;

-- same, tidied up: named count and sorted
SELECT job, employed, COUNT(*) AS headcount
FROM people
GROUP BY job, employed
ORDER BY job, employed;

-- add other aggregates per combination
SELECT job, employed, COUNT(*), ROUND(AVG(age), 1) AS avg_age
FROM people
GROUP BY job, employed;
```

> Adding more columns to `GROUP BY` makes the groups **finer** (more, smaller
> groups). A row counts as a duplicate group only when *every* grouped column
> matches — the same idea as `SELECT DISTINCT job, employed`.

### Filtering groups with `HAVING`

`WHERE` filters **rows** before grouping; `HAVING` filters **groups** after
grouping — so `HAVING` is where you put conditions on an aggregate like
`COUNT(*)` or `AVG(age)`.

```sql
-- only jobs held by more than one person
SELECT job, COUNT(*)
FROM people
GROUP BY job
HAVING COUNT(*) > 1;

-- only jobs whose average age is over 40
SELECT job, ROUND(AVG(age), 1) AS avg_age
FROM people
GROUP BY job
HAVING AVG(age) > 40;

-- WHERE and HAVING together: employed people only (WHERE),
-- then keep groups bigger than 2 (HAVING)
SELECT job, COUNT(*)
FROM people
WHERE employed = TRUE
GROUP BY job
HAVING COUNT(*) > 2
ORDER BY COUNT(*) DESC;
```

> Can't use `WHERE COUNT(*) > 1` — aggregates aren't allowed in `WHERE` because
> it runs *before* rows are grouped. Use `HAVING` for aggregate conditions.

### Naming columns with `AS` (aliases)

`AS` renames a column (or table) in the output. It's optional — a name straight
after the column works too — but it makes results readable and is handy for
computed/aggregate columns.

```sql
-- rename an output column
SELECT first_name AS name, age AS years FROM people;

-- AS is optional (this is identical to the line above)
SELECT first_name name, age years FROM people;

-- essential for computed / aggregate columns (otherwise they get ugly names)
SELECT job, COUNT(*) AS headcount, ROUND(AVG(age), 1) AS avg_age
FROM people
GROUP BY job;

-- table aliases shorten long names in joins
SELECT u.first_name, o.total
FROM users AS u
JOIN orders AS o ON o.user_id = u.id;
```

> A column alias defined in `SELECT` can be reused in `ORDER BY` and `GROUP BY`,
> but **not** in `WHERE` or `HAVING` (those run before the `SELECT` names exist).
> Quote an alias with spaces: `AS "Average Age"`.

### Limiting rows with `LIMIT` (and `OFFSET`)

`LIMIT n` returns at most `n` rows. `OFFSET n` skips the first `n` rows first —
together they page through results. `LIMIT` runs **last**, after `ORDER BY`, so
almost always pair it with `ORDER BY` to make "first N" meaningful.

```sql
-- just the first 5 rows
SELECT * FROM people LIMIT 5;

-- top N: sort, then cut  (the 3 oldest people)
SELECT * FROM people ORDER BY age DESC LIMIT 3;

-- bottom N: flip the sort  (the 3 youngest)
SELECT * FROM people ORDER BY age ASC LIMIT 3;

-- paging: skip some, then take some
SELECT * FROM people ORDER BY id LIMIT 10;            -- page 1 (rows 1–10)
SELECT * FROM people ORDER BY id LIMIT 10 OFFSET 10;  -- page 2 (rows 11–20)

-- quick peek at a table without dumping every row
SELECT * FROM people ORDER BY id LIMIT 20;
```

> Without `ORDER BY`, `LIMIT` returns *some* rows but in no guaranteed order —
> fine for a quick peek, not for "the top N". Large `OFFSET`s get slow (Postgres
> still scans the skipped rows); for deep paging, filter on the last id you saw
> instead: `WHERE id > 20 ORDER BY id LIMIT 10`.

---

## Update (change rows)

```sql
UPDATE people SET job = 'Manager' WHERE id = 1;            -- one column
UPDATE people SET age = 43, employed = TRUE WHERE id = 1;  -- multiple columns
```

`SET` names the columns to change; `WHERE` picks which rows. Any `WHERE`
condition from the [Select section](#4-combine-conditions-with-and--or) works:

```sql
-- update many rows at once (everyone matching the condition)
UPDATE people SET employed = FALSE WHERE age >= 65;

-- match a list of rows
UPDATE people SET job = 'Contractor' WHERE id IN (2, 3, 5);

-- compute the new value from the old one
UPDATE people SET age = age + 1 WHERE first_name = 'Ada';

-- fill in missing data
UPDATE people SET job = 'Unassigned' WHERE job IS NULL;

-- pattern match with LIKE (update rows whose text matches)
UPDATE people SET job = 'Engineer' WHERE job LIKE 'Dev%';       -- starts with 'Dev'
UPDATE people SET employed = FALSE WHERE last_name ILIKE '%son'; -- ends with 'son', any case

-- see what changed (returns the updated rows)
UPDATE people SET employed = TRUE WHERE id = 4 RETURNING *;
```

> **Warning:** `UPDATE` (or `DELETE`) **without a `WHERE`** affects *every* row.
> Tip: run the same `WHERE` as a `SELECT * FROM people WHERE ...` first to
> preview exactly which rows you're about to change.

---

## Delete (remove rows)

```sql
DELETE FROM people WHERE id = 4;   -- one row
DELETE FROM people;                -- ALL rows (careful!)
```

`WHERE` picks which rows to remove. Any condition from the
[Select section](#4-combine-conditions-with-and--or) works:

```sql
-- delete many rows at once (everyone matching the condition)
DELETE FROM people WHERE employed = FALSE;

-- match a list of rows
DELETE FROM people WHERE id IN (2, 3, 5);

-- range and combined conditions
DELETE FROM people WHERE age < 18;
DELETE FROM people WHERE job IS NULL AND employed = FALSE;

-- pattern match with LIKE
DELETE FROM people WHERE last_name LIKE 'J%';    -- last name starts with 'J'

-- see what was removed (returns the deleted rows)
DELETE FROM people WHERE id = 4 RETURNING *;
```

> **Warning:** `DELETE` **without a `WHERE`** removes *every* row. Preview first
> with `SELECT * FROM people WHERE ...` using the same condition. To empty a
> table fast, `TRUNCATE people;` is quicker than `DELETE FROM people;`.

---

## Change table structure (`ALTER TABLE`)

`ALTER TABLE` changes an *existing* table without recreating it — add, remove,
rename, or retype columns, and add or drop constraints.

### Add, drop & rename columns

```sql
-- add a column (existing rows get NULL, or the DEFAULT if given)
ALTER TABLE people ADD COLUMN email VARCHAR(255);
ALTER TABLE people ADD COLUMN active BOOLEAN NOT NULL DEFAULT TRUE;

-- add only if it isn't already there (no error if it exists)
ALTER TABLE people ADD COLUMN IF NOT EXISTS phone VARCHAR(30);

-- remove a column (and its data)
ALTER TABLE people DROP COLUMN email;
ALTER TABLE people DROP COLUMN IF EXISTS email;

-- rename a column / the whole table
ALTER TABLE people RENAME COLUMN job TO role;
ALTER TABLE people RENAME TO staff;
```

#### Removing a column (`DROP COLUMN`)

`DROP COLUMN` deletes the column **and all its data** — it can't be undone
(short of restoring a backup), so double-check the name first.

```sql
-- remove a column
ALTER TABLE people DROP COLUMN email;

-- no error if the column isn't there
ALTER TABLE people DROP COLUMN IF EXISTS email;

-- drop several columns in one statement
ALTER TABLE people DROP COLUMN phone, DROP COLUMN active;

-- force-drop a column other objects depend on (views, FKs, etc.)
ALTER TABLE people DROP COLUMN id CASCADE;
```

> By default a `DROP COLUMN` fails if something depends on it (a view, a foreign
> key). `CASCADE` drops those dependents too — powerful and easy to overreach,
> so prefer fixing the dependents unless you truly want them gone.

### Money & decimals (`DECIMAL` / `NUMERIC`)

`DECIMAL(p, s)` (a synonym for `NUMERIC(p, s)`) stores **exact** decimals — use
it for money and anything where rounding errors matter. `p` = total digits,
`s` = digits after the point. So `DECIMAL(10,2)` holds up to 99,999,999.99.

```sql
-- add a money column with a default
ALTER TABLE people ADD COLUMN salary DECIMAL(10,2) NOT NULL DEFAULT 0;

-- create a table using DECIMAL
CREATE TABLE products (
    id    SERIAL PRIMARY KEY,
    name  VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL,   -- e.g. 19.99
    tax   DECIMAL(5,4)              -- e.g. 0.0825  (4 decimal places)
);

INSERT INTO products (name, price, tax) VALUES ('Widget', 19.99, 0.0825);
```

> Use `DECIMAL`/`NUMERIC` for currency, **not** `FLOAT`/`REAL` — floats are
> approximate and cause rounding bugs (e.g. `0.1 + 0.2 ≠ 0.3`).

### Change a column's type or default

```sql
-- change a column's data type
ALTER TABLE people ALTER COLUMN age TYPE BIGINT;

-- widen a DECIMAL (e.g. allow bigger salaries)
ALTER TABLE people ALTER COLUMN salary TYPE DECIMAL(12,2);

-- when the conversion isn't automatic, tell Postgres how (USING)
ALTER TABLE people ALTER COLUMN age TYPE INT USING age::INT;

-- set or drop a default
ALTER TABLE people ALTER COLUMN active SET DEFAULT TRUE;
ALTER TABLE people ALTER COLUMN active DROP DEFAULT;

-- require / allow NULLs
ALTER TABLE people ALTER COLUMN email SET NOT NULL;
ALTER TABLE people ALTER COLUMN email DROP NOT NULL;
```

### Add & drop constraints

```sql
ALTER TABLE people ADD CONSTRAINT uq_email UNIQUE (email);
ALTER TABLE people ADD CONSTRAINT chk_age CHECK (age >= 0);
ALTER TABLE orders ADD CONSTRAINT fk_user
    FOREIGN KEY (user_id) REFERENCES users(id);

ALTER TABLE people DROP CONSTRAINT uq_email;   -- remove a constraint by name
```

---

## Table admin (DDL)

```sql
TRUNCATE people;             -- fast-empty a table (removes all rows)
DROP TABLE people;           -- delete the table + its data
DROP TABLE IF EXISTS people; -- no error if it doesn't exist
```

---

## Foreign keys (relating tables)

```sql
CREATE TABLE orders (
    id       SERIAL PRIMARY KEY,
    user_id  INT NOT NULL REFERENCES users(id),
    total    NUMERIC(10,2) NOT NULL
);
```

`user_id` must match an existing `users.id`. Add `ON DELETE CASCADE` to also
delete an order's rows when its user is deleted.

### The `REFERENCES` link (`x INT REFERENCES other(id)`)

The foreign key is that one line inside the child table. Read it left to right:

```sql
user_id  INT NOT NULL REFERENCES users(id)
--   ▲    ▲      ▲          ▲       ▲
--   |    |      |          |       └─ the column it points at (usually the PK)
--   |    |      |          └─ the parent table
--   |    |      └─ REFERENCES = "this value must exist over there"
--   |    └─ same type as the column it references (INT ↔ SERIAL/INT)
--   └─ the column that holds the link
```

- Match the **type**: a `SERIAL` primary key is stored as `INT`, so the
  referencing column is `INT` (not `SERIAL`).
- `NOT NULL` makes the link **required**; leave it off to allow "no parent yet".
- `(id)` can be any `UNIQUE`/`PRIMARY KEY` column, but the primary key is normal.

Choose what happens when the parent row is deleted:

```sql
user_id INT REFERENCES users(id) ON DELETE CASCADE   -- delete my rows too
user_id INT REFERENCES users(id) ON DELETE SET NULL  -- keep my rows, blank the link (column must allow NULL)
user_id INT REFERENCES users(id) ON DELETE RESTRICT  -- block the delete while I reference it (default)
```

### Worked example: users → orders (one-to-many)

A **relationship** lives in the data: each order stores the `id` of the user it
belongs to (a *foreign key*). One user can have many orders — a **one-to-many**
relationship. Here's the whole flow end to end.

```sql
-- 1. the "one" side: users
CREATE TABLE users (
    id         SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    email      VARCHAR(255) NOT NULL UNIQUE
);

-- 2. the "many" side: each order points at one user via user_id
CREATE TABLE orders (
    id       SERIAL PRIMARY KEY,
    user_id  INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    total    DECIMAL(10,2) NOT NULL,
    placed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 3. add some rows (users first — an order needs a user that exists)
INSERT INTO users (first_name, email) VALUES
    ('Mike', 'mike@example.com'),
    ('Sara', 'sara@example.com');

INSERT INTO orders (user_id, total) VALUES
    (1, 19.99),
    (1, 45.00),   -- Mike has two orders
    (2, 12.50);   -- Sara has one
```

Now query across the relationship with a **join** (matching `orders.user_id` to
`users.id`):

```sql
-- every order with the name of the user who placed it
SELECT u.first_name, o.id AS order_id, o.total
FROM users u
JOIN orders o ON o.user_id = u.id
ORDER BY u.first_name;

-- per-user summary: how many orders and how much they spent
SELECT u.first_name,
       COUNT(o.id)  AS order_count,
       SUM(o.total) AS total_spent
FROM users u
JOIN orders o ON o.user_id = u.id
GROUP BY u.first_name;

-- include users with NO orders too (LEFT JOIN keeps every user)
SELECT u.first_name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.first_name;
```

> The foreign key enforces integrity: inserting an order with a `user_id` that
> doesn't exist **fails**. With `ON DELETE CASCADE`, deleting a user also removes
> that user's orders; without it, the delete is blocked while orders still point
> at the user.

---

## Joins

```sql
-- Inner: only rows that match in both tables
SELECT u.first_name, o.total
FROM users u
INNER JOIN orders o ON o.user_id = u.id;

-- Left: all users, plus their orders if any (NULLs where none)
SELECT u.first_name, o.total
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
```

`INNER JOIN` and plain `JOIN` are **the same thing** — `JOIN` on its own always
means inner. Spelling out `INNER` makes the intent obvious next to `LEFT JOIN` /
`RIGHT JOIN`, so prefer it for clarity:

```sql
-- one order per row, tagged with who placed it (only matched rows)
SELECT u.first_name, o.id AS order_id, o.total
FROM users u
INNER JOIN orders o ON o.user_id = u.id   -- ON = the relationship
WHERE o.total > 20                         -- WHERE = a filter on the result
ORDER BY o.total DESC;
```

### Outer joins: `LEFT`, `RIGHT`, `FULL`

An **inner** join drops rows that don't match on both sides. **Outer** joins keep
the unmatched rows from one side (or both) and fill the missing columns with
`NULL`. Which side is kept:

| Join         | Keeps                                                   |
| ------------ | ------------------------------------------------------- |
| `INNER JOIN` | only rows that match in **both** tables                 |
| `LEFT JOIN`  | **all left** rows + matches (NULLs where none)          |
| `RIGHT JOIN` | **all right** rows + matches (NULLs where none)         |
| `FULL JOIN`  | **all rows from both** (NULLs on whichever is missing)  |

```sql
-- LEFT: every user, even those with no orders (o.* is NULL for them)
SELECT u.first_name, o.total
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;

-- RIGHT: every order, even any whose user is missing (u.* is NULL)
--   same as swapping the tables and using LEFT JOIN
SELECT u.first_name, o.total
FROM users u
RIGHT JOIN orders o ON o.user_id = u.id;

-- FULL: every user AND every order, matched where possible
SELECT u.first_name, o.total
FROM users u
FULL JOIN orders o ON o.user_id = u.id;

-- classic use of LEFT JOIN: find the UNMATCHED rows
--   (users who have never placed an order)
SELECT u.first_name
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;                    -- kept the user, but no order matched
```

> `LEFT` / `RIGHT` / `FULL JOIN` are all *outer* joins — the word `OUTER` is
> optional (`LEFT OUTER JOIN` ≡ `LEFT JOIN`). `LEFT` is by far the most common;
> any `RIGHT JOIN` can be rewritten as a `LEFT JOIN` by swapping the tables.

### Joining a third table (a join/bridge table)

A third table often exists *only to connect the other two* by their IDs — a
**join table** (a.k.a. bridge / junction table). It holds one row per pairing,
storing just the two foreign keys. This models a **many-to-many** link: a user
can have many products, and a product can belong to many users.

```sql
-- the bridge table: each row links one user to one product by their ids
CREATE TABLE user_products (
    user_id    INT NOT NULL REFERENCES users(id),
    product_id INT NOT NULL REFERENCES products(id),
    PRIMARY KEY (user_id, product_id)   -- one row per (user, product) pair
);
```

To get from `users` to `products`, join **through** the bridge table — each
`ON` just matches the ids that connect two tables:

```sql
-- who owns what: users ↔ products, connected only by the id columns
SELECT u.first_name, p.name AS product
FROM users u
JOIN user_products up ON up.user_id    = u.id   -- users.id  ↔ up.user_id
JOIN products      p  ON p.id          = up.product_id  -- up.product_id ↔ products.id
ORDER BY u.first_name;

-- all products belonging to one specific user
SELECT p.name
FROM users u
JOIN user_products up ON up.user_id = u.id
JOIN products      p  ON p.id       = up.product_id
WHERE u.first_name = 'Mike';
```

> The bridge table has no data of its own here — just the two id columns that
> connect the tables. Each `JOIN ... ON` matches one pair of ids, so two joins
> bridge `users → user_products → products`. Alias every table (`u`, `up`, `p`)
> to keep the `id` columns unambiguous.

### `JOIN ... ON` vs `WHERE` (the relationship vs the filter)

Use **`JOIN ... ON`** to state the *relationship* between the tables — the
condition that matches each order to its user. Then use **`WHERE`** for the
usual *filtering* of which resulting rows you want. Keep the two jobs separate:

```sql
-- ON links the tables; WHERE filters the joined rows
SELECT u.first_name, o.total
FROM users u
JOIN orders o ON o.user_id = u.id   -- ON: the relationship
WHERE o.total > 20                  -- WHERE: a regular filter
ORDER BY o.total DESC;

-- all orders for one specific person
SELECT o.id, o.total
FROM users u
JOIN orders o ON o.user_id = u.id   -- relationship
WHERE u.first_name = 'Mike';        -- filter

-- combine several filters with AND, still on top of the ON join
SELECT u.first_name, o.total
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE o.total > 20
  AND u.first_name = 'Mike';
```

> Rule of thumb: the join key goes in `ON`, everything else in `WHERE`. The old
> comma form (`FROM users u, orders o WHERE o.user_id = u.id`) works but is
> discouraged — forget the `WHERE` and you get a **cross join** (every user
> paired with every order). `JOIN ... ON` keeps the match condition explicit and
> is the only form that supports outer joins like `LEFT JOIN`.

---

## Transactions

Group changes so they all succeed or all fail together.

```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;    -- save changes  (use ROLLBACK; to undo instead)
```

---

## psql meta-commands

```text
\l           list databases
\c pgdb      connect to the pgdb database
\dt          list tables
\d people    describe a table (columns, types, keys)
\du          list roles/users
\dn          list schemas
\x           toggle expanded (vertical) row output
\q           quit psql
```

Connect to the running container (Docker Compose):

```bash
docker compose exec postgres bash
psql -U postgres -d pgdb
```
