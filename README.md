# Postgres + pgAdmin Tutorial

This is a small project whose only job is to stand up a **PostgreSQL database** and a **pgAdmin** web UI using a single `docker-compose.yaml` file. The idea is: instead of installing Postgres on your Mac directly, you run it inside Docker containers. One command starts both services; one command stops them.

This README is written as a tutorial for someone new to Docker Compose. Read it top to bottom, then read the `docker-compose.yaml` again with the explanations in mind.

## What problem does this solve?

Normally, to use Postgres you'd install it, configure it, and manage a running server process on your machine. Docker Compose lets you describe the whole setup in one file and run it anywhere (Mac, Linux, a server) without that manual setup. "Compose" just means "run several containers together that work as a system."

## The two services

A **service** in Compose is one container (one running program). We define two:

1. **`postgres`** — the actual database server.
2. **`pgadmin`** — a web-based GUI to click around your database instead of using the command line.

Each service pulls a ready-made image from Docker Hub, so you don't build anything yourself.

## Walking through `docker-compose.yaml`

### `services:` (line 1)

The top-level key that holds all your containers.

### `postgres:` (line 2)

```yaml
image: postgres:18
```

`image` is the prebuilt blueprint for the container. We pin it to `postgres:18` (the current stable major as of this writing). Pinning a major version keeps things reproducible—you get Postgres 18.x and won't be surprised by an automatic jump to 19. If you want to freeze the exact patch too, use `postgres:18.4`. (Avoid `postgres:latest`, which silently pulls whatever is newest and can break you on upgrade—see "What you're missing" below.)

```yaml
container_name: pg-pgdb
```

The human-readable name Docker gives the running container. Without it, Docker auto-names it `<project>_<service>_<number>`.

```yaml
environment:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: postgres
  POSTGRES_DB: pgdb
```

These are environment variables the Postgres image reads on first startup to initialize the database: the admin user, its password, and a default database to create. The official `postgres` image knows these specific names.

```yaml
ports:
  - "5432:5432"
```

Maps **host port : container port**. `5432` is Postgres's default port. This lets an app on your Mac (or a tool like psql) reach the database at `localhost:5432`. The left number is your machine; the right is inside the container. Change the left one if 5432 is already taken (e.g. another local Postgres is running).

```yaml
volumes:
  - postgres_data:/var/lib/postgresql
```

This is the important persistence line. Containers are **ephemeral**—if you delete the container, its data goes away. A volume mounts a persistent storage area from your host (managed by Docker) onto the directory where Postgres keeps its data. Note the mount point is `/var/lib/postgresql` (the parent), **not** `/var/lib/postgresql/data`—the `postgres:18` image expects a single mount at the parent so it can store data in a version-specific subdirectory (e.g. `/var/lib/postgresql/18`). Mounting at the old `/var/lib/postgresql/data` path makes PG 18 refuse to start. Result: your data survives container restarts and recreations.

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres -d pgdb"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 10s
```

A **healthcheck** is how Docker decides if the container is actually _ready_, not just _running_. `pg_isready` is a tool bundled in the Postgres image that asks the server "are you accepting connections yet?" The `test` is the only required field; `interval`, `timeout`, `retries`, and `start_period` are optional (Docker defaults: `30s`/`30s`/`3`/`0s`) and are written out here explicitly for clarity. `interval: 10s` = run the check every 10s; `timeout: 5s` = each check must finish within 5s; `retries: 5` = mark `unhealthy` after 5 consecutive failures; `start_period: 10s` = give Postgres a 10s grace window to boot before failures count. The resulting `healthy` status is what the `pgadmin` service waits on via `depends_on`.

### `pgadmin:` (lines 14–21)

Same shape as above, but for `dpage/pgadmin4:9` (pgAdmin 4, major version 9):

- `PGADMIN_DEFAULT_EMAIL` / `PGADMIN_DEFAULT_PASSWORD` are the login for the pgAdmin web UI.
- `ports: "5050:80"` — pgAdmin listens on port 80 inside the container; we expose it on `5050` on your Mac so it doesn't collide with anything else. You'll open `http://localhost:5050` in a browser.
- `depends_on` with `condition: service_healthy` — tells Compose: "don't start pgAdmin until the `postgres` container reports `healthy`." This prevents the classic bug where pgAdmin (or any client) starts first and fails to connect because Postgres isn't ready yet. Note `depends_on` controls _start order_, not _network readiness_—the healthcheck is what makes the wait meaningful.

### `volumes:` (lines 22–23)

```yaml
volumes:
  postgres_data:
```

This declares the named volume referenced by the `postgres` service. The name must match the left side of the mount on line 12. Docker creates this volume automatically on `docker compose up`—it does not exist on your system until then, and the name is arbitrary (it's a Docker label, not a macOS folder).

## How to run it

From this folder:

```bash
docker compose up -d
```

### What the `-d` flag means

`-d` stands for **detached**. `docker compose up` has two modes:

- **Foreground (no `-d`)** — Compose starts the containers and attaches your terminal to their combined log stream. The command stays alive; `Ctrl+C` stops following the logs (and signals the containers to stop). Good for watching a fresh setup boot in real time.
- **Detached (`-d`)** — Compose starts the containers in the background and immediately returns your terminal prompt. The services keep running, but you don't see their logs live. You'll normally use this once a setup works, then inspect logs on demand with `docker compose logs -f`.

Other useful `up` flags:

| Flag | What it does |
| --- | --- |
| `--build` | Rebuild any images built from a `Dockerfile` in this project. Not needed here (we use prebuilt images), but essential once you add your own. |
| `--force-recreate` | Recreate the containers even if Compose thinks nothing changed—handy after editing `docker-compose.yaml` when Compose believes it's already up to date. |
| `--no-deps` | Start only the named service, skipping the ones it depends on. |
| `--pull` | Always pull newer images before starting. |
| `-d` | Run detached (background), as described above. |

### Why `down` doesn't take `-d`

`docker compose down` is a **one-shot teardown**: it stops and removes the containers and network, then the command *exits*. There's no long-running foreground process to detach from, so a `-d` flag would be meaningless—`down` already gives your prompt back right away. Use `-v` if you also want to delete the data volume (still just exits when done).

- To stop: `docker compose down`. Add `-v` to also delete the volume (and your data!).

### Accessing pgAdmin

pgAdmin is a web UI, not a command-line tool. Once the containers are up (Postgres `healthy`, pgAdmin `Up`), connect to it in your browser:

1. **Open the UI.** Go to <http://localhost:5050>. (pgAdmin listens on port `80` inside its container; we published it on host port `5050`, so that's the URL you use.)
2. **Log in.** Use the credentials from the compose file:
   - **Email:** `admin@example.com`
   - **Password:** `admin`
3. **Register the Postgres server.** pgAdmin does *not* know about your database automatically—you have to point it at the `postgres` container. In the left panel:
   - Right-click **Servers** → **Register** → **Server…**
   - **General** tab → **Name:** `pgdb` (any label you like).
   - **Connection** tab:
     - **Host name/address:** `postgres` ← this is the Compose *service name*, not `localhost`. Inside the Compose network, containers reach each other by service name. (Using `localhost` here would point at pgAdmin's own container, not the database.)
     - **Port:** `5432`
     - **Maintenance database:** `pgdb`
     - **Username:** `postgres`
     - **Password:** `postgres`
     - Tick **Save password** so you don't re-enter it each time.
   - Click **Save**.
4. **Explore.** Expand **Servers → pgdb → Databases → pgdb**. You can now right-click to create tables, open the **Query Tool** to run SQL, and browse data visually.

> **Note on run mode:** if you started with `docker compose up` (no `-d`), your terminal is busy streaming logs and the services stop if you press `Ctrl+C` in that terminal. The browser steps above work fine while it's running. To free your terminal later, stop it (`Ctrl+C`), then start it detached with `docker compose up -d`.


## Useful commands

Run these from this folder. Most `docker compose` commands read `docker-compose.yaml` in the current directory.

| Command | What it does |
| --- | --- |
| `docker compose up` | Build (if needed) and start all services in the foreground, streaming logs. Stop with `Ctrl+C`. |
| `docker compose up -d` | Same, but **detached** (background). You get your terminal back. |
| `docker compose ps` | List the services and their status (`Up`, `Healthy`, `Exit`). This is the quick "is it running?" check. |
| `docker compose logs` | Show logs from all services. Add a service name to narrow it. |
| `docker compose logs -f postgres` | Follow the Postgres logs live. Good for watching boot or debugging connection errors. |
| `docker compose down` | Stop and remove the containers and network. Volume data is kept. |
| `docker compose down -v` | Like `down`, but also **deletes the volume** (`postgres_data`) and all its data. Full reset. |
| `docker compose restart` | Stop and start the containers without rebuilding. |
| `docker compose exec postgres psql -U postgres -d pgdb` | Open a `psql` shell inside the running Postgres container (no local client needed). |
| `docker compose config` | Validate and print the resolved Compose config to confirm no YAML errors. |
| `docker volume ls` | List all Docker volumes on your machine (find `postgres-cc_postgres_data`). |
| `docker stats` | Live CPU/memory/network usage for running containers. |

### Quick health check

After `docker compose up -d`, confirm Postgres is healthy before connecting:

```bash
docker compose ps
```

You should see `pg-pgdb` with status `Up ... (healthy)` and `pgadmin-pgdb` `Up`. Then open `http://localhost:5050`.

## Reusing Postgres for other projects

A common early question: when you start a new app that needs a database, do you come back to *this* repo and spin it up, or do you put a Compose file in the new project?

**Short answer: give each project its own `docker-compose.yaml`.** This repo is one self-contained Postgres+pgAdmin environment. Your next project should have its own database instance, declared in its own `docker-compose.yaml` next to its code. That way the database lives with the project, starts and stops with it, and doesn't tangle with this one.

How to do it:

1. **Copy this file as a starting point.** Take `docker-compose.yaml` from here and drop it into the new project's folder.
2. **Make the names unique.** Change `container_name`, `POSTGRES_DB`, and the volume name so they don't collide with this repo (or any other running Compose project). Compose already prefixes volumes with the project folder name, but explicit, distinct names avoid confusion. For example, a `blog-api` project might use `container_name: pg-blog`, `POSTGRES_DB: blog`, and volume `blog_data`.
3. **Avoid port clashes.** If this repo's Postgres is using host port `5432`, a second Compose project can't also publish `5432` on your machine. Either stop this one first, or change the new project's host port (e.g. `"5433:5432"`). Inside each Compose network the container port stays `5432`; only the *host* side collides.
4. **Different credentials.** Don't reuse `postgres`/`postgres` across projects—set a distinct password per project (ideally via an `.env` file, as noted in "What you're missing").
5. **Run it from the new folder.** `cd` into the new project and `docker compose up -d`. Compose reads the `docker-compose.yaml` in the current directory, so each project is independently managed.

**When you *would* reuse this exact instance:** only if multiple small apps on your machine should share one database. In that case you can `docker compose up -d` here and point those apps at `localhost:5432` (or `5433`, etc.). But sharing one DB across projects is usually a bad habit—data, schemas, and lifecycles get coupled. Prefer one database per project.

**One pgAdmin is enough.** You don't need a pgAdmin container in every project. Keep this one running and register each project's server in it (host = the other project's service name *only if* they share a Docker network; otherwise use `host.docker.internal` or your machine's `localhost` plus the project's published host port). For simplicity when learning, copying pgAdmin per project is also fine.

## What you're missing (things to learn next)

This minimal file intentionally leaves out a lot. Here's what's commonly added and why:

- **Secrets via `.env` (done).** The credentials are no longer hardcoded in `docker-compose.yaml`. They live in a `.env` file in this folder, and the compose file references them with `${POSTGRES_PASSWORD}`, `${PGADMIN_DEFAULT_PASSWORD}`, etc. Docker Compose auto-loads `.env` and substitutes those values at startup. The `.env` is gitignored (see `.gitignore`), so the real secrets never get committed; `.env.example` is committed instead with placeholder values so collaborators know which keys to supply. This is the standard Compose approach for local dev. For production-grade secret handling, use **Docker secrets** or a secrets manager (e.g. AWS Secrets Manager, Vault)—those encrypt at rest and avoid plaintext files entirely.

- **A custom Docker network.** Compose already creates a default network so the two services can talk by name. You usually don't need to declare one, but knowing it exists explains why `host: postgres` works inside pgAdmin.

- **Resource limits / restart policy.** `restart: unless-stopped` keeps the DB running after a reboot or crash. You can also cap CPU/memory.

- **An init script.** If you want tables/seed data created automatically on first run, mount a `.sql` file into `/docker-entrypoint-initdb.d/` on the postgres service.

- **pgAdmin persistence.** pgAdmin currently stores its own config in its container with no volume, so your registered servers disappear on `docker compose down`. Add a volume for `/var/lib/pgadmin` if you want them to persist.

None of these are required to get a working local dev database—they're the next steps once the basics click.
