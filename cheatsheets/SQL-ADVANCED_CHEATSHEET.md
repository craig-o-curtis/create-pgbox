# SQL Advanced Cheatsheet (next steps)

The tools that come after joins and `GROUP BY`: CTEs, window functions,
reporting tricks, and checking query speed. All examples run on
**dvdrental**.

```bash
docker compose exec postgres psql -U postgres -d dvdrental
```

← Back to the [cheatsheet index](./CHEATSHEET.md) · comes after
[SQL Queries](./SQL-QUERIES_CHEATSHEET.md)

---

## Contents

- [CTEs: name a step with WITH](#ctes-name-a-step-with-with)
- [Window functions](#window-functions)
- [Reporting helpers](#reporting-helpers)
- [Upsert](#upsert)
- [Is my query slow? EXPLAIN and indexes](#is-my-query-slow-explain-and-indexes)

---

## CTEs: name a step with `WITH`

A **CTE** (common table expression) gives a subquery a name, so you can read
the query top to bottom instead of inside out.

```sql
WITH customer_totals AS (            -- step 1: total per customer
    SELECT customer_id, SUM(amount) AS total
    FROM payment
    GROUP BY customer_id
)
SELECT c.first_name, c.last_name, ct.total     -- step 2: use it like a table
FROM customer_totals ct
JOIN customer c USING (customer_id)
WHERE ct.total > (SELECT AVG(total) FROM customer_totals)   -- reuse it
ORDER BY ct.total DESC;
-- Eleanor Hunt 211.55, Karl Seal 208.58, Marion Snyder 194.61 …
```

Several steps, separated by commas:

```sql
WITH rentals_per_film AS (
    SELECT i.film_id, COUNT(*) AS rentals
    FROM rental r
    JOIN inventory i USING (inventory_id)
    GROUP BY i.film_id
),
ranked AS (
    SELECT f.title, f.rating, rpf.rentals
    FROM film f
    JOIN rentals_per_film rpf USING (film_id)
)
SELECT * FROM ranked ORDER BY rentals DESC LIMIT 10;
```

When to use one: if you find yourself nesting subqueries or copying the same
subquery twice.

---

## Window functions

`GROUP BY` **collapses** rows into one per group. A window function
**keeps every row** and adds a value calculated across related rows.

```text
function() OVER (PARTITION BY <groups> ORDER BY <order>)
                 └─ like GROUP BY,      └─ order inside each group
                    but rows stay
```

### Ranking

```sql
-- rank films by length within each rating
SELECT title, rating, length,
       RANK() OVER (PARTITION BY rating ORDER BY length DESC) AS rnk
FROM film;
```

| Function | Ties (three films at 185 min) |
| --- | --- |
| `ROW_NUMBER()` | 1, 2, 3, 4: every row gets a unique number |
| `RANK()` | 1, 1, 1, 4: ties share a rank, then it skips |
| `DENSE_RANK()` | 1, 1, 1, 2: ties share a rank, no gap |

### Top N per group

Window results can't go in `WHERE` (it runs before them). Wrap the query in a
subquery or CTE and filter outside:

```sql
-- the single longest film in each rating
SELECT title, rating, length
FROM (
    SELECT title, rating, length,
           ROW_NUMBER() OVER (PARTITION BY rating ORDER BY length DESC, title) AS rn
    FROM film
) t
WHERE rn = 1;
```

### Running totals and moving through rows

```sql
-- daily revenue + running total
SELECT payment_date::date                               AS day,
       SUM(amount)                                      AS daily,
       SUM(SUM(amount)) OVER (ORDER BY payment_date::date) AS running_total
FROM payment
GROUP BY day
ORDER BY day;
-- 2007-02-14  116.73   116.73
-- 2007-02-15 1188.92  1305.65 …
```

`SUM(SUM(amount))`: the inner `SUM` is the normal `GROUP BY` total for each
day. The outer `SUM … OVER` adds those daily totals up as it goes.

```sql
-- month-over-month change with LAG (the previous row's value)
SELECT month,
       revenue,
       revenue - LAG(revenue) OVER (ORDER BY month) AS change
FROM (
    SELECT DATE_TRUNC('month', payment_date)::date AS month, SUM(amount) AS revenue
    FROM payment
    GROUP BY 1
) m
ORDER BY month;
```

| Function | Gives |
| --- | --- |
| `LAG(col)` / `LEAD(col)` | value from the previous / next row |
| `FIRST_VALUE(col)` | first value in the window |
| `SUM/AVG/COUNT(...) OVER (...)` | running or group-wide totals without collapsing rows |

```sql
-- each payment as a % of that customer's total, rows kept
SELECT customer_id, amount,
       ROUND(100 * amount / SUM(amount) OVER (PARTITION BY customer_id), 1) AS pct_of_customer
FROM payment;
```

---

## Reporting helpers

### Group by month: `DATE_TRUNC`

```sql
SELECT DATE_TRUNC('month', payment_date)::date AS month,
       COUNT(*)    AS payments,
       SUM(amount) AS revenue
FROM payment
GROUP BY 1          -- 1 = the first SELECT column
ORDER BY 1;
-- 2007-02-01  2016   8351.84
-- 2007-03-01  5644  23886.56 …
```

### Count different things in one pass: `FILTER`

```sql
SELECT COUNT(*) FILTER (WHERE rating = 'G') AS g_films,
       COUNT(*) FILTER (WHERE rating = 'R') AS r_films,
       COUNT(*)                             AS all_films
FROM film;
-- 178 | 195 | 1000
```

This is cleaner than `SUM(CASE WHEN rating = 'G' THEN 1 ELSE 0 END)`.

### Combine values into one string: `STRING_AGG`

```sql
-- each film's cast on one line
SELECT f.title,
       STRING_AGG(a.first_name || ' ' || a.last_name, ', ' ORDER BY a.last_name) AS cast_list
FROM film f
JOIN film_actor fa USING (film_id)
JOIN actor a       USING (actor_id)
GROUP BY f.title;
```

### One row per group: `DISTINCT ON`

This is Postgres-only, and shorter than the window-function version:

```sql
-- each customer's most recent payment
SELECT DISTINCT ON (customer_id) customer_id, payment_date, amount
FROM payment
ORDER BY customer_id, payment_date DESC;
```

The `ORDER BY` must start with the `DISTINCT ON` columns. The rest of the
`ORDER BY` decides which row is kept.

---

## Upsert

Insert, or update if the row already exists. You need a `UNIQUE` or
`PRIMARY KEY` column to detect the conflict.

```sql
INSERT INTO employee (id, emp_name, emp_location, emp_salary)
VALUES (2, 'Barry', 'Mumbai', 35000)
ON CONFLICT (id) DO UPDATE
SET emp_salary = EXCLUDED.emp_salary;   -- EXCLUDED = the row you tried to insert

-- or skip duplicates silently
... ON CONFLICT (id) DO NOTHING;
```

More examples: [SQL Basics → insert only if not already there](./SQL-BASICS_CHEATSHEET.md#insert-only-if-not-already-there-avoid-duplicates).

---

## Is my query slow? `EXPLAIN` and indexes

```sql
EXPLAIN SELECT * FROM rental WHERE customer_id = 148;          -- the plan only
EXPLAIN ANALYZE SELECT * FROM rental WHERE customer_id = 148;  -- runs it, shows real timings
```

What to look for in the output:

| You see | Means |
| --- | --- |
| `Seq Scan on rental` | reads every row. Fine for small tables, slow for big ones. |
| `Index Scan` / `Bitmap Index Scan` | uses an index to jump to matching rows |
| `actual time=…` | real milliseconds (with `ANALYZE` only) |

> `EXPLAIN ANALYZE` actually **runs** the statement. On an
> `UPDATE`/`DELETE`, wrap it in `BEGIN; … ROLLBACK;`.

Add an index on columns you often filter or join on:

```sql
CREATE INDEX idx_payment_payment_date ON payment (payment_date);
DROP INDEX idx_payment_payment_date;
```

- Primary keys and `UNIQUE` columns get an index automatically.
- Foreign key columns **don't**. Index them if you join on them a lot.
- Every index slows down writes a little, so only add the ones you need.
- `\di` in psql lists indexes. `\timing` shows how long each query took.
