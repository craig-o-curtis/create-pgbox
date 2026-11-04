# SQL Functions Cheatsheet

Built-in functions for dates, numbers and text, plus conditional
expressions (`CASE`, `COALESCE`, `NULLIF`) and type conversion (`CAST`).
Results shown in comments are real output from this project's Postgres 18
container.

← Back to the [cheatsheet index](./CHEATSHEET.md)

Try any of these on their own. `SELECT` works without a table:

```sql
SELECT ROUND(8.43932, 2);   -- 8.44
```

---

## Contents

- [Dates and times](#dates-and-times)
- [Math](#math)
- [Strings](#strings)
- [CAST: converting types](#cast-converting-types)
- [CASE: if / else](#case-if--else)
- [COALESCE: fallback for NULL](#coalesce-fallback-for-null)
- [NULLIF: turn a value into NULL](#nullif-turn-a-value-into-null)

---

## Dates and times

### What time is it?

| Function | Returns | Example |
| --- | --- | --- |
| `CURRENT_DATE` | date | `2026-11-04` |
| `CURRENT_TIME` | time + zone | `16:20:11.53+00` |
| `CURRENT_TIMESTAMP` / `NOW()` | timestamptz | `2026-11-04 16:20:11.53+00` |

This container's time zone is `Etc/UTC`, so these show UTC, not your laptop's
clock. See [time zones](./SQL-SCHEMA_CHEATSHEET.md#time-zones-store-utc).

### Differences between dates: `AGE` and subtraction

```sql
SELECT AGE('2024-11-29', '2023-05-01');      -- 1 year 6 mons 28 days   (later date first)
SELECT AGE(TIMESTAMP '2023-05-01');          -- from that date until today

SELECT DATE '2024-01-10' - DATE '2024-01-01';                -- 9  (date - date = whole days)
SELECT DATE '2023-09-14' + 7;                                -- 2023-09-21
SELECT TIMESTAMP '2023-09-14 17:10:29' + INTERVAL '2 hours'; -- 2023-09-14 19:10:29
```

Compare a duration to an `INTERVAL`:

```sql
SELECT rental_id
FROM rental
WHERE return_date - rental_date >= INTERVAL '7 days';
```

### Text ↔ dates

```sql
SELECT TO_DATE('20230501', 'YYYYMMDD');                         -- 2023-05-01
SELECT TO_CHAR(TIMESTAMP '2023-09-14 17:10:29', 'YYYY-MM-DD HH24:MI'); -- 2023-09-14 17:10
SELECT TO_CHAR(TIMESTAMP '2023-09-14 17:10:29', 'Mon DD, YYYY');       -- Sep 14, 2023
SELECT '2025-12-03'::date;                                      -- ISO strings convert directly
```

### `EXTRACT`: pull one part out of a date

`EXTRACT(part FROM value)`. All examples use `TIMESTAMP '2023-09-14 17:10:29'`:

| Part | Result | Part | Result |
| --- | --- | --- | --- |
| `YEAR` | 2023 | `HOUR` | 17 |
| `QUARTER` | 3 | `SECOND` | 29.000000 |
| `MONTH` | 9 | `MILLISECOND` | 29000.000 |
| `WEEK` | 37 (ISO week) | `MICROSECOND` | 29000000 |
| `DAY` | 14 | `EPOCH` | 1694711429 (seconds since 1970-01-01) |
| `DOY` | 257 (day of year) | `DECADE` | 202 |
| `DOW` | 4 (Sun = 0) | `CENTURY` | 21 |
| `ISODOW` | 4 (Mon = 1, Sun = 7) | `MILLENNIUM` | 3 |

`MILLISECOND`/`MICROSECOND` include the whole seconds field (29 s =
29,000 ms). There's no `NANOSECOND`; it raises an error.

```sql
-- which months have rentals?
SELECT DISTINCT EXTRACT(MONTH FROM rental_date) AS month
FROM rental
ORDER BY month;
```

### `DATE_TRUNC`: round down to a period (for reports)

```sql
SELECT DATE_TRUNC('month', TIMESTAMP '2023-09-14 17:10:29');  -- 2023-09-01 00:00:00

-- revenue per month
SELECT DATE_TRUNC('month', payment_date)::date AS month,
       SUM(amount)                             AS revenue
FROM payment
GROUP BY month
ORDER BY month;
```

`EXTRACT(MONTH …)` gives just `9`, so September 2023 and September 2024 end
up in the same group. `DATE_TRUNC` keeps the year, which makes it the safer
choice for grouping.

---

## Math

| Function / operator | Example | Result |
| --- | --- | --- |
| `+ - * /` | `amount * 30` | |
| `/` on two integers | `7 / 2` | `3` ⚠️ truncated |
| `/` with a decimal | `7 / 2.0` or `7::numeric / 2` | `3.5` |
| `%` or `MOD(x, y)` | `10 % 3` | `1` |
| `ABS(x)` | `ABS(-7.99)` | `7.99` |
| `ROUND(x, n)` | `ROUND(8.43932, 2)` | `8.44` |
| `TRUNC(x, n)` | `TRUNC(8.43932, 2)` | `8.43` (cut, no rounding) |
| `CEIL(x)` / `FLOOR(x)` | `CEIL(4.1)` / `FLOOR(4.9)` | `5` / `4` |
| `POWER(x, y)` | `POWER(2, 10)` | `1024` |

Bitwise operators exist too (`&`, `|`, `#` XOR, `~` NOT, `<<`, `>>`), but you
rarely need them.

```sql
SELECT customer_id, amount * 30 AS "Monthly Payment" FROM payment;
SELECT ROUND(AVG(amount), 2) FROM payment;      -- 4.20
```

---

## Strings

Positions are **1-indexed**: the first character is position 1.

| Function | Example | Result |
| --- | --- | --- |
| `CONCAT(a, b, …)` | `CONCAT(first_name, ' ', last_name)` | `Mike Hillyer` |
| `a \|\| b` | `first_name \|\| ' ' \|\| last_name` | same, but `NULL` if any part is NULL |
| `CONCAT_WS(sep, …)` | `CONCAT_WS(', ', 'Austin', NULL, 'TX')` | `Austin, TX` (skips NULLs) |
| `LENGTH(s)` | `LENGTH('postgresql')` | `10` |
| `LEFT(s, n)` | `LEFT('peter', 2)` | `pe` |
| `RIGHT(s, n)` | `RIGHT('peter', 2)` | `er` |
| `SUBSTR(s, start)` | `SUBSTR('postgresql', 3)` | `stgresql` |
| `SUBSTR(s, start, count)` | `SUBSTR('postgresql', 3, 2)` | `st` |
| `POSITION(sub IN s)` | `POSITION('sql' IN 'postgresql')` | `8` |
| `UPPER` / `LOWER` | `UPPER('abc')` | `ABC` |
| `INITCAP(s)` | `INITCAP('mike hillyer')` | `Mike Hillyer` |
| `TRIM(s)` | `TRIM('  hi  ')` | `hi` |
| `REPLACE(s, from, to)` | `REPLACE('a-b-c', '-', '/')` | `a/b/c` |
| `SPLIT_PART(s, sep, n)` | `SPLIT_PART('a@b.com', '@', 2)` | `b.com` |
| `LPAD(s, len, fill)` | `LPAD('7', 3, '0')` | `007` |

```sql
-- the end of each title, from character 10 onward
SELECT title, SUBSTR(title, 10) FROM film;

-- only titles long enough to have a 16th character
SELECT title, LENGTH(title), SUBSTR(title, 16)
FROM film
WHERE LENGTH(title) >= 16;

-- email domain
SELECT email, SPLIT_PART(email, '@', 2) AS domain FROM customer;
```

> **`SUBSTR` with start 0:** `SUBSTR('Stephens', 0, 3)` returns `St`, not
> `Ste`. The 3 characters are counted from position 0, which doesn't exist,
> so you only get positions 1–2. Start at 1.

---

## CAST: converting types

```sql
SELECT CAST('50' AS INTEGER);        -- 50
SELECT '50'::INTEGER;                -- 50   (:: is Postgres shorthand for CAST)
SELECT CAST('2025-12-03' AS DATE);   -- 2025-12-03
SELECT 42::TEXT;                     -- '42'
SELECT CAST('50A' AS INTEGER);       -- ERROR: invalid input syntax for type integer

SELECT city_id, CAST(city_id AS VARCHAR) FROM address;
```

---

## CASE: if / else

```sql
CASE
    WHEN condition_1 THEN result_1
    WHEN condition_2 THEN result_2
    ELSE fallback            -- optional; without it, no match = NULL
END
```

Conditions are checked **top to bottom** and the first match wins, so put
the most specific condition first.

```sql
-- employee_db: salary bands
SELECT emp_name,
       emp_salary,
       CASE
           WHEN emp_salary <= 30000 THEN 'Low'
           WHEN emp_salary <= 50000 THEN 'Average'
           ELSE 'High'
       END AS "Salary Band"
FROM employee;
```

Because the first match wins, the second `WHEN` doesn't need
`emp_salary > 30000 AND …`: any salary that reaches it is already above 30000.

`CASE` works in `ORDER BY` too, to sort in a custom order:

```sql
SELECT title, rating
FROM film
ORDER BY CASE rating
             WHEN 'G'     THEN 1
             WHEN 'PG'    THEN 2
             WHEN 'PG-13' THEN 3
             ELSE 4
         END;
```

For a full example, see
[customer segments](./SQL-QUERIES_CHEATSHEET.md#customer-segments-with-case).

---

## COALESCE: fallback for NULL

`COALESCE(a, b, c, …)` returns the **first value that isn't NULL**, or NULL if
they all are. Use it to show a default instead of a blank.

```sql
SELECT COALESCE(NULL, NULL, 'x');    -- x

SELECT address, COALESCE(address2, 'N/A') AS address2
FROM address
WHERE address2 IS NULL;

-- employee_db: show 'UTC' where no time zone was stored
SELECT customer_first_name, COALESCE(delivery_timezone, 'UTC') AS tz
FROM order_details;
```

---

## NULLIF: turn a value into NULL

`NULLIF(a, b)` returns **NULL if `a = b`**, otherwise returns `a`.

```sql
SELECT NULLIF(20, 20);          -- NULL
SELECT NULLIF(20, 0);           -- 20
SELECT NULLIF('John', 'Peter'); -- John
```

Main use: **avoid divide-by-zero.** Dividing by zero is an error, but
dividing by NULL just gives NULL.

```sql
-- 10 / amount would error on any amount of 0
SELECT amount, 10 / NULLIF(amount, 0) AS ratio FROM payment;

-- then COALESCE the NULLs back to 0 if you want a number
SELECT amount, COALESCE(10 / NULLIF(amount, 0), 0) AS ratio FROM payment;
```
