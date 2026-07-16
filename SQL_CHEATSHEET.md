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

```sql
SELECT COUNT(*), AVG(age), MIN(age), MAX(age), SUM(age) FROM people;

SELECT job, COUNT(*)
FROM people
GROUP BY job
HAVING COUNT(*) > 1;     -- HAVING filters groups (WHERE filters rows)
```

---

## Update (change rows)

```sql
UPDATE people SET job = 'Manager' WHERE id = 1;            -- one column
UPDATE people SET age = 43, employed = TRUE WHERE id = 1;  -- multiple columns
```

> **Warning:** `UPDATE` (or `DELETE`) **without a `WHERE`** affects *every* row.

---

## Delete (remove rows)

```sql
DELETE FROM people WHERE id = 4;   -- one row
DELETE FROM people;                -- ALL rows (careful!)
```

---

## Table admin (DDL)

```sql
ALTER TABLE people ADD COLUMN email VARCHAR(255);   -- add a column
ALTER TABLE people DROP COLUMN email;               -- remove a column
ALTER TABLE people RENAME COLUMN job TO role;       -- rename a column
ALTER TABLE people RENAME TO staff;                 -- rename the table

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

---

## Joins

```sql
-- Inner: only rows that match in both tables
SELECT u.first_name, o.total
FROM users u
JOIN orders o ON o.user_id = u.id;

-- Left: all users, plus their orders if any (NULLs where none)
SELECT u.first_name, o.total
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
```

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
