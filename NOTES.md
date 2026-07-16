# Notes on course - will delete later after added to README

## pgsql cli

Connect to Postgres via the `pg-pgdb` container:

```bash
docker compose exec postgres psql -U postgres -d pgdb
```

Full reference (flags, meta-commands, host connection): see
[PSQL-CLI_CHEATSHEET.md](./PSQL-CLI_CHEATSHEET.md).

## Adding a new server in pg admin

Open pgAdmin at [http://localhost:5050](http://localhost:5050) and log in with the
`PGADMIN_DEFAULT_EMAIL` / `PGADMIN_DEFAULT_PASSWORD` values from your `.env`.

Then register the Postgres container as a server:

1. In the left **Object Explorer**, right-click **Servers** -> **Register** -> **Server...**
   (or use the menu: **Object** -> **Register** -> **Server...**).
2. On the **General** tab:
   - **Name**: `pgdb` (any friendly label you like).
3. Switch to the **Connection** tab and fill in:
   - **Host name/address**: `postgres` OR the container name `pg-pgdb`
     (the Docker Compose service name, since pgAdmin connects over the
     Compose network — do **not** use `localhost` or `127.0.0.1` when using Docker).
   - **Port**: `5432`
   - **Maintenance database**: `pgdb`
   - **Username**: value of `POSTGRES_USER` from `.env` (default `postgres`)
   - **Password**: value of `POSTGRES_PASSWORD` from `.env`
   - Optionally tick **Save password?** so you don't re-enter it each time.
4. Click **Save**.

pgAdmin will connect and the new server appears under **Servers**. Expand
**pgdb** -> **Databases** -> **pgdb** to browse schemas and tables.

> Note: `postgres` works as the host because both containers share the Compose
> network. If you instead connect from your host machine (e.g. `psql` on your
> laptop), use `localhost` and port `5432`, which is published in
> `docker-compose.yaml`.

### Why are there two databases (`pgdb` and `postgres`)?

Under the server, `Databases` lists more than one entry:

```
Servers
└── pgdb              ← the SERVER name (just the label I chose in pgAdmin)
    └── Databases
        ├── postgres  ← built-in default admin database (leave it alone)
        └── pgdb      ← MY database (from POSTGRES_DB: pgdb in docker-compose.yaml)
```

- The **server** is also called `pgdb`, but that's only the display name I typed
  when registering the server — it's unrelated to the database of the same name.
- A single Postgres server hosts **many databases**. Every fresh install ships
  with a default **`postgres`** database used by admin tools (pgAdmin, psql) as a
  guaranteed place to connect when starting a session. Don't put app data there.
- **`pgdb`** is my actual database, created because `docker-compose.yaml` sets
  `POSTGRES_DB: pgdb`. This is where my `people` / `users` tables live.
- If you enable "show system objects" you'll also see **`template0`** and
  **`template1`** — read-only templates Postgres copies when creating new
  databases.

> Gotcha: always work inside **`pgdb`**, not `postgres`. Running a query while
> connected to the wrong database is how you end up with tables/rows in the wrong
> place. This is why the CLI uses `-d pgdb` and the pgAdmin server's Maintenance
> database is `pgdb`. In the Query Tool, check the connected database shown in
> the title bar before running anything.

## Terminology

Plain-English definitions for the terms that come up in this course.

### Postgres 101 (the basics)

- **PostgreSQL (Postgres)**: An open-source relational database. It stores data
  in tables and lets you query it with SQL.
- **RDBMS (Relational Database Management System)**: The category of software
  Postgres belongs to — it manages data organized into related tables.
- **SQL (Structured Query Language)**: The language you use to read and change
  data (e.g. `SELECT`, `INSERT`, `UPDATE`, `DELETE`).
- **Server / instance**: One running copy of Postgres. A single server can host
  many databases.
- **Database**: A named container for your data. In this project the database is
  `pgdb`. One server can hold multiple databases that don't share data.
- **Schema**: A namespace _inside_ a database that groups objects (tables,
  views, etc.). The default schema is `public`. Think of it as a folder.
- **Table**: A grid of rows and columns where the actual data lives.
- **Row (record / tuple)**: One entry in a table (one horizontal line).
- **Column (field / attribute)**: One named piece of data in a table, with a
  type (e.g. `name TEXT`, `age INT`).
- **Data type**: What kind of value a column holds — `INTEGER`, `TEXT`,
  `BOOLEAN`, `TIMESTAMP`, etc.
- **Primary key (PK)**: A column (or set of columns) that uniquely identifies
  each row. No two rows can share the same primary key.
- **Foreign key (FK)**: A column that points to the primary key of another
  table — this is how tables get "related".
- **Index**: A lookup structure that makes queries faster (like a book index).
- **Query**: A command you send to the database, usually to fetch data.
- **Constraint**: A rule the data must follow (e.g. `NOT NULL`, `UNIQUE`).
- **Transaction**: A group of changes that either all succeed or all fail
  together (`BEGIN` ... `COMMIT` / `ROLLBACK`).
- **Role / user**: An account that can log in and has permissions. In this
  project the default role is `postgres`.

### Tools & this project's setup

- **psql**: The official Postgres command-line client for running SQL and admin
  commands. (See the [pgsql cli](#pgsql-cli) section above.)
- **pgAdmin**: A web-based GUI for browsing and managing Postgres, running here
  at [http://localhost:5050](http://localhost:5050).
- **Maintenance database**: The database pgAdmin/psql connects to _first_ just
  to establish the session. Here it's `pgdb`.
- **Connection string**: The details needed to connect — host, port, database,
  user, password.
- **Port `5432`**: The default port Postgres listens on.

### Docker terms (because we run in containers)

- **Docker**: Tool for packaging and running software in isolated containers.
- **Image**: A read-only blueprint for a container (e.g. `postgres:18`,
  `dpage/pgadmin4:9`).
- **Container**: A running instance of an image. Ours are named `pg-pgdb`
  (Postgres) and `pgadmin-pgdb` (pgAdmin).
- **Docker Compose**: A tool that starts multiple containers together from one
  `docker-compose.yaml` file.
- **Service**: A container definition in Compose. Our services are `postgres`
  and `pgadmin`. The service name doubles as the network hostname.
- **Volume**: Storage that lives outside the container so data survives
  restarts. Ours is `postgres_data`, keeping the database on disk.
- **Compose network**: The private network Compose creates so containers can
  reach each other by service name (why pgAdmin uses `postgres` as the host).
- **Healthcheck**: A test Compose runs to confirm a container is actually ready
  (here it uses `pg_isready` before pgAdmin starts).
- **Environment variables (`.env`)**: Config values (like `POSTGRES_USER`,
  `POSTGRES_PASSWORD`) kept out of the compose file and loaded at startup.
- **Port mapping (e.g. `5050:80`)**: Connects a port on your machine (left) to a
  port inside the container (right), so you can reach the app from your browser.

## Creating tables

There are two ways to create a table in pgAdmin: the GUI dialog (right-click) or
the Query Tool (writing SQL). Both end up in the same place.

First, in the **Object Explorer**, drill down to where the table should live:

`Servers` -> `pgdb` -> `Databases` -> `pgdb` -> `Schemas` -> `public` -> `Tables`

### Option A: GUI dialog (right-click)

1. Right-click **Tables** -> **Create** -> **Table...**.
2. On the **General** tab, give the table a **Name** (e.g. `users`).
3. Switch to the **Columns** tab and click the **+** button to add each column:
   - Set the column **Name** (e.g. `id`, `name`, `email`).
   - Pick a **Data type** (e.g. `serial`, `varchar`, `text`, `boolean`).
   - Toggle **Not NULL** and **Primary key** as needed.
4. (Optional) Use the **Constraints** tab for keys, unique rules, etc.
5. Click **Save**. The new table appears under **Tables**.

> Tip: This dialog is just building SQL for you. Click the **SQL** tab at any
> time to preview the exact `CREATE TABLE` statement it will run.

### Option B: Query Tool (write SQL)

1. Click the **pgdb** database in the Object Explorer to select it.
2. Right-click it (or the **Tables** node) and choose **Query Tool**
   (or use the toolbar / **Object** menu).
3. Type your `CREATE TABLE` statement in the editor.
4. Run it by clicking the **▶ Execute** button (or press **F5**).

Example — create a simple `users` table:

```sql
CREATE TABLE users (
    id         SERIAL PRIMARY KEY,
    first_name       VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email      VARCHAR(255) NOT NULL UNIQUE,
    is_active  BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()  -- NOW() = current date + time, e.g. 2026-07-16 11:25:39.123456 (ISO-8601 style)
);
```

What each part does:

- `SERIAL PRIMARY KEY` — auto-incrementing integer that uniquely identifies each row.
- `VARCHAR(100)` — text with a max length; `NOT NULL` means it's required.
- `UNIQUE` — no two rows can share the same email.
- `DEFAULT TRUE` / `DEFAULT NOW()` — values used automatically if none is given.
- `NOW()` — returns the current **date + time** (a datetime, not just a date),
  displayed in ISO-8601 style: `YYYY-MM-DD HH:MM:SS.ffffff`. It's a
  `timestamptz` (with time zone), but since `created_at` is declared as plain
  `TIMESTAMP` it's stored _without_ the offset. Use `TIMESTAMPTZ` instead if you
  want the time zone preserved (recommended for real apps).

After running it, refresh the **Tables** node (right-click -> **Refresh**) to see
`users` appear. You can also confirm it in psql:

```sql
\dt          -- list tables
\d users     -- describe the users table
```

Another example

```sql
CREATE TABLE IF NOT EXISTS people (
	id INT PRIMARY KEY,
	first_name VARCHAR(255),
	last_name VARCHAR(255),
	age INT,
	job VARCHAR(255),
	employed BOOL
);
```

## Common commands

Everyday SQL against the `people` table above. Run these in the pgAdmin Query
Tool (F5) or in psql.

> Note: string values use **single quotes** (`'Mike'`). Double quotes (`"..."`)
> are for identifiers like column/table names, not text values.

### Insert (add rows)

```sql
-- Full row: list every value in column order
INSERT INTO people VALUES (1, 'Mike', 'Jones', 42, 'Developer', TRUE);

-- Safer: name the columns, so order doesn't matter and you can skip nullable ones
INSERT INTO people (id, first_name, last_name)
VALUES (2, 'Sara', 'Lee');

-- Insert several rows at once
INSERT INTO people (id, first_name, age) VALUES
    (3, 'Ada', 36),
    (4, 'Grace', 45);
```

### Select (read rows)

```sql
SELECT * FROM people;                       -- all columns, all rows
SELECT first_name, age FROM people;         -- only some columns
SELECT * FROM people WHERE age > 40;        -- filter with WHERE
SELECT * FROM people WHERE employed = TRUE;
SELECT * FROM people ORDER BY age DESC;     -- sort
SELECT * FROM people LIMIT 5;               -- first 5 rows
SELECT COUNT(*) FROM people;                -- how many rows
```

### Update (change existing rows)

```sql
-- Change one person (WHERE targets the row by id)
UPDATE people SET job = 'Manager' WHERE id = 1;

-- Update multiple columns at once
UPDATE people SET age = 43, employed = TRUE WHERE id = 1;
```

> Warning: an `UPDATE` (or `DELETE`) **without a `WHERE`** hits _every_ row.
> Always include a `WHERE` unless you really mean all rows.

### Delete (remove rows)

```sql
DELETE FROM people WHERE id = 4;   -- delete one row
DELETE FROM people;                -- delete ALL rows (careful!)
```

### Table admin

```sql
DROP TABLE people;                 -- delete the whole table + its data
DROP TABLE IF EXISTS people;       -- no error if it doesn't exist
TRUNCATE people;                   -- fast-empty a table (removes all rows)
ALTER TABLE people ADD COLUMN email VARCHAR(255);   -- add a column
```

### Handy psql meta-commands

```text
\l           -- list databases
\c pgdb      -- connect to the pgdb database
\dt          -- list tables
\d people    -- describe the people table (columns, types, keys)
\q           -- quit psql
```

### pgsql commands

- connect to running docker - pgsl -h <host instance> -U <username> -d <dbname>
