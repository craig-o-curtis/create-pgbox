# psql CLI Cheatsheet

`psql` is the PostgreSQL command-line client. In this project Postgres runs
inside the `pg-pgdb` container, so you connect by running `psql` from inside that
container via Docker Compose.

> **Config values:** the examples use this project's defaults — user `postgres`,
> database `pgdb`, container/service `postgres` (`pg-pgdb`), port `5432`. The
> user and password come from `.env` (`POSTGRES_USER` / `POSTGRES_PASSWORD`) and
> the names/ports from `docker-compose.yaml`. If you change those, adjust the
> commands to match. No password is ever hardcoded below.

---

## Spin up first

The container must be running before you can connect. From the project folder:

```bash
cp .env.example .env       # first time only
docker compose up -d       # start Postgres + pgAdmin
docker compose ps          # confirm postgres is "healthy"
```

Stop with `docker compose down` (keeps data) or `docker compose down -v` (wipes
data). See the [README](./README.md) for full setup details.

---

## Connecting

### One-liner (connect directly)

```bash
docker compose exec postgres psql -U postgres -d pgdb
```

### Two-step (shell in, then run psql)

```bash
# open a shell inside the postgres container
docker compose exec postgres bash

# then connect to the pgdb database as the postgres user
psql -U postgres -d pgdb
```

**Flags:**

- `-U postgres` — user (matches `POSTGRES_USER` in `.env`)
- `-d pgdb` — database to connect to
- `-h <host>` — host (only needed when connecting from your machine)
- `-p 5432` — port

No password is needed from _inside_ the container (trusted local socket). When
connected, the prompt becomes `pgdb=#`.

### From your host machine (not the container)

Postgres is published on `localhost:5432`, so if you have `psql` installed
locally you can connect directly (you'll be prompted for the password):

```bash
psql -h localhost -p 5432 -U postgres -d pgdb
```

---

## Meta-commands (psql shortcuts)

These start with a backslash and are specific to psql (not SQL).

```text
\l              list databases
\c pgdb         connect to / switch to the pgdb database
\dt             list tables
\d people       describe a table (columns, types, keys)
\d+ people      describe a table with extra detail (size, comments)
\dn             list schemas
\du             list roles / users
\df             list functions
\dv             list views
\di             list indexes
\x              toggle expanded (vertical) row output
\timing         toggle showing how long queries take
\e              open the last query in an editor
\i file.sql     run SQL from a file
\! <cmd>        run a shell command
\?              help on meta-commands
\h SELECT       SQL syntax help for a command
\q              quit psql
```

---

## Quick session check

```sql
SELECT version();   -- confirm connection / server version
SELECT current_database(), current_user;
\dt                 -- list tables
```

---

## Exiting

- `\q` — quit psql
- `exit` (or **Ctrl-D**) — leave the container shell (only if you used the
  two-step method)

**Tip:** the **one-liner** avoids the extra container shell entirely — `\q` drops
you straight back to your own terminal, no `exit` needed:

```bash
docker compose exec postgres psql -U postgres -d pgdb
```

If you used the two-step method and land at a container prompt like
`root@<id>:/#` after `\q`, that's the container's bash shell — type `exit` to
return to your machine. (The container keeps running.)
