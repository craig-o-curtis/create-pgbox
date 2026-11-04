# Postgres Cheatsheets

Start here. This page tells you which sheet to open, which database to connect
to, and how to plan a query. The detail lives in the topic sheets below.

---

## Which sheet do I need?

| Sheet | Open it when you want to… | Practice data |
| --- | --- | --- |
| [PSQL-CLI](./PSQL-CLI_CHEATSHEET.md) | connect, list tables, use `\d`, `\x`, `\timing` | any |
| [SQL Basics](./SQL-BASICS_CHEATSHEET.md) | create a table, insert/update/delete, simple `WHERE`, constraints, foreign keys | `people`, `users`, `orders` |
| [SQL Queries](./SQL-QUERIES_CHEATSHEET.md) | filter, aggregate, `GROUP BY`/`HAVING`, subqueries, `EXISTS`, joins, `UNION` | **dvdrental** |
| [SQL Functions](./SQL-FUNCTIONS_CHEATSHEET.md) | dates & times, math, strings, `CAST`, `CASE`, `COALESCE`, `NULLIF` | dvdrental, `employee_db` |
| [SQL Schema](./SQL-SCHEMA_CHEATSHEET.md) | design tables, pick data types, `ALTER`, `DELETE` vs `TRUNCATE` vs `DROP`, views | **`employee_db`** |
| [SQL Advanced](./SQL-ADVANCED_CHEATSHEET.md) | CTEs (`WITH`), window functions, monthly reports, `EXPLAIN`, indexes | dvdrental |

---

## The databases on this server

One Postgres server (`pg-pgdb`) hosts several databases. They don't share
tables, so **connect to the right one first**.

| Database | What's in it | Connect |
| --- | --- | --- |
| `dvdrental` | Sample DVD-rental shop: 15 tables, 1000 films, 599 customers, 14,596 payments. Used for most query practice. | `docker compose exec postgres psql -U postgres -d dvdrental` |
| `employee_db` | Your own practice tables: `employee` and `order_details` | `docker compose exec postgres psql -U postgres -d employee_db` |
| `pgdb` | The project's default database (empty right now) | `docker compose exec postgres psql -U postgres -d pgdb` |
| `postgres` | Built-in admin database. Leave it alone. | — |

Already inside psql? Switch with `\c dvdrental`. In pgAdmin, open the Query Tool
from the database you want (right-click it → **Query Tool**), and check the
database name in the tab title before running anything.

**Server name vs database name:** in pgAdmin, the top-level `pgdb` under
**Servers** is only the label you typed when you registered the server. The
`pgdb` under **Databases** is a real database (created because
`docker-compose.yaml` sets `POSTGRES_DB: pgdb`). Turn on "show system objects"
and you'll also see `template0` and `template1`, the read-only templates
Postgres copies whenever it creates a new database.

Setup steps for dvdrental are in the [README](../README.md#sample-data-dvdrental).
Scripts to recreate `employee` and `order_details` are in
[SQL Schema](./SQL-SCHEMA_CHEATSHEET.md#practice-tables-in-employee_db).

### dvdrental map (how the tables join)

```text
actor ──< film_actor >── film ──< film_category >── category
                          │  └── language            (language_id)
                          │
                       inventory          one row per physical copy (film_id, store_id)
                          │
customer ──< rental >─────┘               rental.inventory_id → inventory
   │            │
   │         payment                      payment.rental_id → rental
   │                                      payment.customer_id → customer
   └── address ── city ── country         (address_id, city_id, country_id)

staff ── address        store ── address
```

`──<` means "one to many". Every join key is named `<table>_id`
(`film_id`, `customer_id`, …), so the shared column is usually easy to spot.

Common paths:

| To get… | Join through |
| --- | --- |
| a film's actors | `film → film_actor → actor` |
| a film's category | `film → film_category → category` |
| what a customer rented | `customer → rental → inventory → film` |
| what a customer paid | `customer → payment` |
| a customer's address | `customer → address → city → country` |

---

## Plan a query in 3 steps

Before typing any SQL, answer these:

1. **What do I want to see?** List the output columns (for example: customer
   name, email, total spent).
2. **Which tables hold that, and which columns?** Use `\d table` in psql, or
   expand **Columns** in pgAdmin.
3. **What column do they share?** That's your `JOIN ... ON` (usually
   `<something>_id`).

Then build it up a piece at a time: `SELECT ... FROM` one table → add the join
→ add `WHERE` → add `GROUP BY` → add `ORDER BY`/`LIMIT`. Run it after each step.

---

## Clause order

You **write** clauses in one order, but Postgres **runs** them in another.
This explains most "column doesn't exist" errors.

| Write in this order | Postgres runs it | Notes |
| --- | --- | --- |
| `SELECT` | 5 | Aliases (`AS "Total"`) are created here |
| `FROM` / `JOIN` | 1 | Tables are combined first |
| `WHERE` | 2 | Filters **rows**. No aggregates, no `SELECT` aliases. |
| `GROUP BY` | 3 | Collapses rows into groups |
| `HAVING` | 4 | Filters **groups**. Aggregates allowed. |
| `ORDER BY` | 6 | Can use `SELECT` aliases |
| `LIMIT` / `OFFSET` | 7 | Last step |

---

## Gotchas I've hit

1. **Quotes:** `'text'` for values, `"Column Name"` for identifiers. `"Mike"`
   means a column called Mike.
2. **`UPDATE`/`DELETE` with no `WHERE`** changes every row. Run the same `WHERE`
   as a `SELECT` first.
3. **Aggregates go in `HAVING`, not `WHERE`.** `WHERE SUM(amount) > 100` is an
   error.
4. **Every non-aggregated column in `SELECT` must be in `GROUP BY`.**
5. **`DISTINCT` + `GROUP BY` is redundant.** Grouping already gives one row per
   value.
6. **`= NULL` never matches.** Use `IS NULL` / `IS NOT NULL`.
7. **`COUNT(*)` counts rows; `COUNT(col)` skips NULLs.** In a `LEFT JOIN`,
   count a column from the right-hand table to get 0 for "no match".
8. **In a chain of joins, count the table that holds the thing you're
   counting.** To count rentals, use `COUNT(r.rental_id)`, not
   `COUNT(i.inventory_id)`. See
   [rentals per rating](./SQL-QUERIES_CHEATSHEET.md#rentals-per-rating-3-tables-keep-unrented-films).
9. **An explicit `NULL` skips the `DEFAULT`.** Inserting `NULL` into
   `delivery_timezone DEFAULT 'UTC'` stores `NULL`. Leave the column out to get
   the default.
10. **Putting decimals in an `INT` column rounds them without warning.**
    `9.82` → `10`. Use `NUMERIC(p,s)` for money.
11. **Integer division truncates.** `7/2` = `3`, `7/2.0` = `3.5`.
12. **Strings are 1-indexed.** `SUBSTR('postgresql', 1, 4)` = `post`.
13. **`NOT IN` with a `NULL` in the list returns nothing.** Use `NOT EXISTS`.
14. **Group by an id, not a display name.** Two customers can share a name.

---

## Terms

### Postgres

- **PostgreSQL (Postgres):** an open-source relational database (RDBMS). It
  stores data in tables that relate to each other, and you query it with SQL.
- **Server / instance:** one running copy of Postgres. Here that's the
  `pg-pgdb` container, and it hosts several databases.
- **Database:** a named container for data (`pgdb`, `dvdrental`). Databases
  on the same server don't share tables.
- **Schema:** a folder *inside* a database that groups tables, views and other
  objects. The default schema is `public`.
- **Table / row / column:** the grid, one record in it, one named and typed
  field.
- **Primary key (PK) / foreign key (FK):** the column that uniquely identifies
  each row / a column that points at another table's PK.
- **Constraint:** a rule the data must follow (`NOT NULL`, `UNIQUE`, `CHECK`).
- **Index:** a lookup structure that makes filtering and joining faster.
- **View:** a saved query you can select from like a table.
- **Transaction:** a group of changes that all succeed or all fail together
  (`BEGIN` … `COMMIT` / `ROLLBACK`).
- **Role / user:** an account that can log in. Here it's `postgres`.

### Tools

- **psql:** the Postgres command-line client. See the
  [PSQL-CLI cheatsheet](./PSQL-CLI_CHEATSHEET.md).
- **pgAdmin:** the web GUI at <http://localhost:5050>.
- **Maintenance database:** the database pgAdmin connects to *first*, just to
  open a session (`pgdb` here).
- **Connection string:** host + port + database + user + password. Port
  `5432` is the Postgres default.

### Docker

- **Image:** a read-only blueprint for a container (`postgres:18`,
  `dpage/pgadmin4:9`).
- **Container:** a running image. Ours are `pg-pgdb` and `pgadmin-pgdb`.
- **Docker Compose / service:** starts several containers from
  `docker-compose.yaml`. The service names (`postgres`, `pgadmin`) also work as
  hostnames on the Compose network. That's why pgAdmin connects to host
  `postgres`, not `localhost`.
- **Volume:** storage outside the container, so data survives restarts
  (`postgres_data`). `docker compose down -v` deletes it.
- **Healthcheck:** Compose runs `pg_isready` to know when Postgres is ready,
  not just started.
- **Port mapping (`5050:80`):** your machine's port (left) → the container's
  port (right).
- **`.env`:** config values like `POSTGRES_PASSWORD`, kept out of git.
