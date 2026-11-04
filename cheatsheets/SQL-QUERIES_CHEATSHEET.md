# SQL Queries Cheatsheet (dvdrental)

Reading data: filters, aggregates, grouping, subqueries, joins and `UNION`.
Every example runs against the **dvdrental** sample database.

```bash
docker compose exec postgres psql -U postgres -d dvdrental
```

← Back to the [cheatsheet index](./CHEATSHEET.md) ·
table map: [dvdrental map](./CHEATSHEET.md#dvdrental-map-how-the-tables-join) ·
basics on small tables: [SQL Basics](./SQL-BASICS_CHEATSHEET.md)

---

## Contents

- [Filtering rows](#filtering-rows)
- [Sorting and limiting](#sorting-and-limiting)
- [Aggregates](#aggregates)
- [GROUP BY and HAVING](#group-by-and-having)
- [Subqueries](#subqueries)
- [EXISTS / NOT EXISTS](#exists--not-exists)
- [Joins](#joins)
- [UNION](#union)
- [Worked exercises](#worked-exercises)

---

## Filtering rows

```sql
-- unique values only
SELECT DISTINCT customer_id AS "CID"
FROM payment
WHERE amount >= 7.99
ORDER BY customer_id DESC;

-- AND / OR (use brackets when mixing them)
SELECT title, rental_rate, replacement_cost
FROM film
WHERE rental_rate > 2.99 OR replacement_cost > 19.99;

-- BETWEEN is inclusive at both ends
SELECT * FROM payment WHERE amount BETWEEN 1 AND 2;
SELECT * FROM payment WHERE amount NOT BETWEEN 1 AND 2;

-- IN = shorthand for a chain of ORs
SELECT * FROM payment WHERE amount IN (1.99, 2.99);
SELECT * FROM address WHERE district IN ('California', 'Alberta');
SELECT * FROM address WHERE district NOT IN ('California', 'Alberta');
```

> `amount IN (1, 2)` returns nothing: every payment ends in `.99`. Look at
> the real values first with `SELECT DISTINCT amount FROM payment ORDER BY 1;`.

### Pattern matching: `LIKE` / `ILIKE`

| Wildcard | Means |
| --- | --- |
| `%` | any run of characters, including none |
| `_` | exactly one character |

```sql
SELECT district FROM address WHERE district LIKE '%and%';     -- contains "and"
SELECT district FROM address WHERE district LIKE 'And%';      -- starts with "And"
SELECT district FROM address WHERE district LIKE '%and_';     -- "and" + exactly 1 more char at the end
SELECT district FROM address WHERE district LIKE '_alifornia'; -- any 1 char, then "alifornia"
SELECT district FROM address WHERE district ILIKE '%AnD%';    -- ILIKE = ignore case
```

`LIKE` is case-sensitive: `'%c%'` and `'%C%'` return different rows.

### Putting filters together

```sql
-- G or PG films, longer than 2 hours, with "Action" in the description
SELECT title, rating, length, description
FROM film
WHERE rating IN ('G', 'PG')
  AND length > 120
  AND description LIKE '%Action%';

-- films rented out for 4–6 days, most expensive to replace first, top 100
SELECT title, replacement_cost, rental_duration
FROM film
WHERE rental_duration BETWEEN 4 AND 6
ORDER BY replacement_cost DESC
LIMIT 100;
```

---

## Sorting and limiting

```sql
SELECT title, rental_rate FROM film ORDER BY rental_rate;        -- ASC is the default
SELECT title, rental_rate FROM film ORDER BY rental_rate DESC;

-- sort by one column, then break ties with another
SELECT title, rental_rate, replacement_cost
FROM film
ORDER BY rental_rate ASC, replacement_cost DESC;

-- top N = ORDER BY + LIMIT
SELECT title, replacement_cost FROM film ORDER BY replacement_cost DESC LIMIT 5;
SELECT amount, payment_date, customer_id FROM payment ORDER BY payment_date DESC LIMIT 10;
```

Without `ORDER BY`, `LIMIT` returns rows in no guaranteed order.

---

## Aggregates

Aggregates collapse many rows into one value.

```sql
SELECT COUNT(*)              FROM film;   -- 1000 rows
SELECT AVG(rental_rate)      FROM film;
SELECT MIN(replacement_cost) FROM film;
SELECT MAX(replacement_cost) FROM film;
SELECT SUM(amount)           FROM payment;

-- several at once, rounded and named
SELECT ROUND(AVG(rental_rate), 2) AS "Average rental cost",
       MAX(replacement_cost)      AS "Max replacement cost"
FROM film;
```

### The three kinds of `COUNT`

| Form | Counts |
| --- | --- |
| `COUNT(*)` | every row |
| `COUNT(col)` | rows where `col` is **not NULL** |
| `COUNT(DISTINCT col)` | different non-NULL values |

```sql
SELECT COUNT(*), COUNT(address2), COUNT(DISTINCT district) FROM address;
-- 603 rows, but only 599 have an address2
```

---

## GROUP BY and HAVING

`GROUP BY` gives one result row per value. `HAVING` then filters those groups.

```sql
-- revenue per staff member, biggest first
SELECT staff_id, SUM(amount) AS "Money earned"
FROM payment
GROUP BY staff_id
ORDER BY SUM(amount) DESC;

-- only within a date range (WHERE filters rows BEFORE grouping)
SELECT staff_id, SUM(amount)
FROM payment
WHERE payment_date BETWEEN '2007-02-01' AND '2007-02-28'
GROUP BY staff_id;

-- count and total per rating
SELECT rating, COUNT(*) AS films, SUM(replacement_cost)
FROM film
GROUP BY rating;

-- HAVING filters groups (aggregates allowed here, not in WHERE)
SELECT staff_id, SUM(amount)
FROM payment
GROUP BY staff_id
HAVING SUM(amount) > 31000;
```

### Most common actor first names

```sql
SELECT first_name, COUNT(*)
FROM actor
GROUP BY first_name
ORDER BY count DESC, first_name;   -- an unnamed COUNT(*) column is called "count"
```

> Two attempts that fail, and why:
> - `SELECT first_name, COUNT(first_name) FROM actor;` — error: `first_name`
>   must be in `GROUP BY` (or inside an aggregate).
> - `SELECT DISTINCT first_name, COUNT(*) ... GROUP BY first_name` works, but
>   `DISTINCT` does nothing here. `GROUP BY` already returns one row per name.

---

## Subqueries

A query inside a query. The inner one runs first, so write and test it on its
own before wrapping it.

```sql
-- step 1: the inner query on its own
SELECT AVG(rental_rate) FROM film;          -- 2.98

-- step 2: use it as a value
SELECT title, rental_rate
FROM film
WHERE rental_rate > (SELECT AVG(rental_rate) FROM film);

-- in HAVING: staff whose average sale beats the overall average
SELECT staff_id, AVG(amount) AS "Average Amount"
FROM payment
GROUP BY staff_id
HAVING AVG(amount) > (SELECT AVG(amount) FROM payment);

-- as a list for IN: customers who have paid more than $10 in one go
SELECT first_name, last_name
FROM customer
WHERE customer_id IN (SELECT customer_id FROM payment WHERE amount > 10);
```

---

## EXISTS / NOT EXISTS

`EXISTS (subquery)` is true if the subquery returns **at least one row**. It
stops at the first match, and it doesn't care what you select, so `SELECT 1`
is the usual convention.

```sql
-- films that have at least one copy in inventory  → 958
SELECT f.title
FROM film f
WHERE EXISTS (
    SELECT 1 FROM inventory i WHERE i.film_id = f.film_id
);

-- films with NO copies at all  → 42
SELECT f.title
FROM film f
WHERE NOT EXISTS (
    SELECT 1 FROM inventory i WHERE i.film_id = f.film_id
);
```

The inner query refers to `f` from the outer query, so Postgres runs it once
per film. This is called a **correlated subquery**.

> Prefer `NOT EXISTS` over `NOT IN (subquery)`. If the subquery returns even
> one `NULL`, `NOT IN` returns no rows.

---

## Joins

Joins combine tables on a shared column. Use the
[3-step plan](./CHEATSHEET.md#plan-a-query-in-3-steps): output → tables →
shared column.

| Join | Keeps |
| --- | --- |
| `JOIN` / `INNER JOIN` | only rows that match in **both** tables |
| `LEFT JOIN` | **every** row from the left table; `NULL`s where the right has no match |
| `RIGHT JOIN` | **every** row from the right table; `NULL`s where the left has no match |
| `FULL JOIN` | every row from **both** |

### Inner join + table aliases

```sql
-- mailing list: name + full address (customer.address_id = address.address_id)
SELECT CONCAT(c.first_name, ' ', c.last_name) AS "Name",
       a.address, a.address2, a.district, a.postal_code, a.phone
FROM customer c
JOIN address a ON c.address_id = a.address_id;
```

`FROM customer c` gives the table a short alias. Once aliased, use `c.`
everywhere in that query.

### Left join: find what's missing

```sql
-- films with no inventory (same 42 as the NOT EXISTS example)
SELECT f.film_id, f.title
FROM film f
LEFT JOIN inventory i ON f.film_id = i.film_id
WHERE i.inventory_id IS NULL;
```

`LEFT JOIN` + `WHERE right.id IS NULL` = "rows in the left table with no
match on the right." This pattern is called an **anti-join**.

### Right and full joins

```sql
-- every address, plus the staff member who lives there (mostly NULL)
SELECT CONCAT(s.first_name, ' ', s.last_name) AS "Staff", a.address, a.district
FROM staff s
RIGHT JOIN address a ON s.address_id = a.address_id;

-- unmatched rows on either side
SELECT *
FROM customer c
FULL JOIN address a ON c.address_id = a.address_id
WHERE c.customer_id IS NULL OR a.address_id IS NULL;
```

Any `RIGHT JOIN` can be rewritten as a `LEFT JOIN` by swapping the tables.
`LEFT` is easier to read, so most people stick with it.

### Three tables (through a bridge table)

`film_actor` holds only pairs of ids. It links films to actors
(many-to-many).

```sql
-- actors by number of films, most first
SELECT a.actor_id,
       CONCAT(a.first_name, ' ', a.last_name) AS "Actor",
       COUNT(fa.film_id)                     AS "Film Count"
FROM actor a
JOIN film_actor fa ON a.actor_id = fa.actor_id
GROUP BY a.actor_id                 -- group by the id, not the name
ORDER BY "Film Count" DESC;
-- Gina Degeneres 42, Walter Torn 41, Mary Keitel 40 …
```

You only need `film` itself if you want film columns (like the title). To
count films, `film_actor` is enough.

### `USING` shorthand

When both tables name the column the same, `USING` saves typing:

```sql
SELECT f.title, l.name AS language
FROM film f
JOIN language l USING (language_id);
```

---

## UNION

`UNION` stacks the results of two queries on top of each other.

```sql
SELECT name FROM category
UNION
SELECT name FROM language;
```

Rules: same number of columns, compatible types, same order. The column
names come from the first query.

| | Duplicates | Speed |
| --- | --- | --- |
| `UNION` | removed | slower (has to de-duplicate) |
| `UNION ALL` | kept | faster |

```sql
SELECT first_name FROM actor UNION     SELECT first_name FROM customer;  -- 647 rows
SELECT first_name FROM actor UNION ALL SELECT first_name FROM customer;  -- 799 rows
```

Join vs union: a **join** adds columns side by side; a **union** adds rows
underneath.

---

## Worked exercises

### Top 5 customers by spend

```sql
SELECT customer_id, SUM(amount) AS "Spend"
FROM payment
GROUP BY customer_id
ORDER BY "Spend" DESC
LIMIT 5;
-- 148 → 211.55, 526 → 208.58, 178 → 194.61 …
```

### Films with their language

```sql
SELECT f.title, f.rental_rate, l.name AS language
FROM film f
JOIN language l ON f.language_id = l.language_id
ORDER BY f.title;
```

(In this copy of dvdrental, every film is English.)

### Rentals per rating (3 tables, keep unrented films)

**Goal:** every rating, and how many times films with that rating were
rented. The path is `film → inventory → rental`, and films that were never
rented must still be counted (as 0).

```sql
SELECT f.rating,
       COUNT(r.rental_id) AS "Rentals"     -- count the RENTAL table
FROM film f
LEFT JOIN inventory i ON f.film_id      = i.film_id
LEFT JOIN rental    r ON i.inventory_id = r.inventory_id
GROUP BY f.rating
ORDER BY "Rentals" DESC;
-- PG-13 3585, NC-17 3293, PG 3212, R 3181, G 2773
```

> **Gotcha:** with `COUNT(i.inventory_id)`, PG shows **3213**, one too many.
> One PG copy has never been rented. It survives the `LEFT JOIN` with
> `r.* = NULL`, but its `inventory_id` isn't NULL, so it gets counted. Always
> count a column from the table that holds the thing you're counting.

The same result written with `RIGHT JOIN` (it reads backwards, which is why
`LEFT` is preferred):

```sql
SELECT f.rating, COUNT(r.rental_id) AS "Rentals"
FROM rental r
JOIN inventory i  ON r.inventory_id = i.inventory_id
RIGHT JOIN film f ON i.film_id      = f.film_id
GROUP BY f.rating;
```

### Customers who kept a rental 7+ days

```sql
SELECT CONCAT(c.first_name, ' ', c.last_name) AS "Customer",
       c.email,
       r.rental_date,
       r.return_date,
       AGE(r.return_date, r.rental_date)      AS "Rent Duration"
FROM rental r
JOIN customer c ON r.customer_id = c.customer_id
WHERE r.return_date IS NOT NULL                       -- not returned yet = skip
  AND AGE(r.return_date, r.rental_date) >= INTERVAL '7 days'
ORDER BY "Rent Duration" DESC;                        -- longest first
```

### Customer segments with `CASE`

```sql
SELECT CONCAT(c.first_name, ' ', c.last_name) AS "Customer",
       c.email,
       SUM(p.amount) AS "Total Spent",
       CASE
           WHEN SUM(p.amount) >= 200 THEN 'Elite'
           WHEN SUM(p.amount) >= 150 THEN 'Platinum'
           WHEN SUM(p.amount) >= 100 THEN 'Gold'
           ELSE 'Silver'
       END AS "Member Type"
FROM payment p
JOIN customer c ON p.customer_id = c.customer_id
GROUP BY c.customer_id               -- the id makes each group one person
ORDER BY "Total Spent" DESC;
-- Elite 2, Platinum 19, Gold 275, Silver 303
```

`GROUP BY c.customer_id` lets you select `c.first_name` and `c.email` too,
because Postgres knows each id has exactly one name and email (it's the
primary key). This is saved as a view in
[SQL Schema → Views](./SQL-SCHEMA_CHEATSHEET.md#views).
