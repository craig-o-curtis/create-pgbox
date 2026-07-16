# create-pgbox

Spin up a local **PostgreSQL** database and a **pgAdmin** web UI with one
command, using Docker Compose. No local Postgres install required — everything
runs in containers, and your data persists between restarts.

- **Postgres** — the database server (`postgres:18`), on `localhost:5432`
- **pgAdmin** — a browser GUI (`dpage/pgadmin4:9`), on `http://localhost:5050`

---

## 10-second spin-up

1. **Open Docker Desktop** (the Docker engine must be running).
2. **Create `.env`** — required, first time only:
   ```bash
   cp .env.example .env
   ```
3. **Start it:**
   ```bash
   docker compose up -d
   ```

pgAdmin → <http://localhost:5050>. Stop with `docker compose down`.

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (includes
  Docker Compose)

## Verify & access

After `docker compose up -d`, check both services are up (Postgres should show
`healthy`):

```bash
docker compose ps
```

Open pgAdmin at <http://localhost:5050> and log in with the
`PGADMIN_DEFAULT_EMAIL` / `PGADMIN_DEFAULT_PASSWORD` from your `.env`
(template defaults: `admin@example.com` / `change_me`).

To stop:

```bash
docker compose down          # stop; keeps your data
docker compose down -v       # stop and DELETE all data (full reset)
```

## Configuration (`.env`)

Credentials are read from a `.env` file (git-ignored, never committed). Copy the
template and adjust as needed:

| Variable                 | Default             | Purpose                    |
| ------------------------ | ------------------- | -------------------------- |
| `POSTGRES_USER`          | `postgres`          | Database admin user        |
| `POSTGRES_PASSWORD`      | `change_me`         | Database admin password    |
| `PGADMIN_DEFAULT_EMAIL`  | `admin@example.com` | pgAdmin login email        |
| `PGADMIN_DEFAULT_PASSWORD` | `change_me`       | pgAdmin login password     |

The database name (`pgdb`), ports (`5432`, `5050`), and container names
(`pg-pgdb`, `pgadmin-pgdb`) are set in `docker-compose.yaml`.

## Connect to the database

**pgAdmin (browser):** after logging in, register the server once:

1. Right-click **Servers** → **Register** → **Server…**
2. **General** tab → **Name:** `pgdb` (any label).
3. **Connection** tab:
   - **Host name/address:** `postgres` ← the Compose *service name*, not
     `localhost`. Containers reach each other by service name over the Compose
     network.
   - **Port:** `5432`
   - **Maintenance database:** `pgdb`
   - **Username / Password:** your `POSTGRES_USER` / `POSTGRES_PASSWORD`
   - Tick **Save password**, then **Save**.
4. Expand **Servers → pgdb → Databases → pgdb** to browse, run the Query Tool,
   and create tables.

**psql (CLI):**

```bash
docker compose exec postgres psql -U postgres -d pgdb
```

See [`PSQL-CLI_CHEATSHEET.md`](./PSQL-CLI_CHEATSHEET.md) for more, and
[`SQL_CHEATSHEET.md`](./SQL_CHEATSHEET.md) for SQL reference.

## Useful commands

Run from this folder (Compose reads `docker-compose.yaml` in the current dir):

| Command | What it does |
| --- | --- |
| `docker compose up -d` | Start all services detached (background). |
| `docker compose ps` | Show service status (`Up`, `healthy`, `Exit`). |
| `docker compose logs -f postgres` | Follow Postgres logs live (debug boot/connection). |
| `docker compose restart` | Restart containers without rebuilding. |
| `docker compose down` | Stop and remove containers + network (data kept). |
| `docker compose down -v` | Same, but also delete volumes (**all data lost**). |
| `docker compose exec postgres psql -U postgres -d pgdb` | Open a `psql` shell inside the container. |
| `docker compose config` | Validate/print the resolved Compose config. |
| `docker volume ls` | List Docker volumes (find `create-pgbox_postgres_data`). |

> `-d` = **detached** (runs in the background, returns your prompt). Without it,
> Compose streams logs in the foreground and `Ctrl+C` stops the services.
> `down` never takes `-d` — it's a one-shot teardown that exits immediately.

## How `docker-compose.yaml` works

A quick tour of the file:

- **`services:`** — the two containers we run: `postgres` and `pgadmin`.
- **`image:`** — prebuilt images from Docker Hub, pinned to a major version
  (`postgres:18`, `dpage/pgadmin4:9`) for reproducibility. Avoid `:latest`.
- **`environment:`** — startup config. The Postgres image reads `POSTGRES_USER`,
  `POSTGRES_PASSWORD`, and `POSTGRES_DB` on **first init** to create the admin
  user and the `pgdb` database. Credentials come from `.env` via `${VAR}`.
- **`ports:`** — `host:container`. `5432:5432` exposes Postgres to your machine;
  `5050:80` publishes pgAdmin's internal port 80 on host port 5050.
- **`volumes:`** — `postgres_data:/var/lib/postgresql` persists the database so it
  survives container recreation. (Mount at the parent `/var/lib/postgresql`, not
  `/var/lib/postgresql/data` — PG 18 stores data in a version subdirectory.)
  `pgadmin_data:/var/lib/pgadmin` persists pgAdmin's own settings/registered
  servers.
- **`healthcheck:`** — `pg_isready` tells Docker when Postgres is actually
  *ready*, not just *running*.
- **`depends_on: condition: service_healthy`** — pgAdmin waits for Postgres to be
  healthy before starting, avoiding "connection refused" on boot.
- **`restart: unless-stopped`** — both services come back after a reboot/crash,
  but stay down if you `docker compose down` them.

## Reusing this for other projects

Prefer **one `docker-compose.yaml` per project** rather than sharing this one.
To spin up a DB for a new app:

1. Copy `docker-compose.yaml` and `.env.example` into the new project.
2. Give things unique names (`container_name`, `POSTGRES_DB`, volume names) to
   avoid collisions.
3. Avoid host-port clashes: only one project can publish `5432` at a time —
   change the new project's host port (e.g. `"5433:5432"`).
4. Use distinct credentials per project via its own `.env`.
5. `cd` into the new project and `docker compose up -d`.

One pgAdmin can manage many servers, so you don't need a pgAdmin container in
every project.

## Optional & advanced ideas

Not included here (kept lightweight on purpose), but easy to add later:

- **Seed data** — drop `*.sql` / `*.sh` into a folder mounted at
  `/docker-entrypoint-initdb.d/`; they run once on a fresh `postgres_data` volume.
- **Pre-register pgAdmin servers** — mount a `servers.json` and set
  `PGADMIN_SERVER_JSON_FILE` to auto-add the server on first launch.
- **Resource limits** — cap CPU/memory per service via `deploy.resources.limits`.
- **Production secrets** — use Docker secrets or a secrets manager (Vault, AWS
  Secrets Manager) instead of a plaintext `.env`.
