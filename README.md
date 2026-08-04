<img width="500" height="310" alt="backgroundimage" src="https://github.com/user-attachments/assets/e25c244c-f6e6-4ceb-8ff1-d08a96061e49" />

# Setting up PostgreSQL in Docker with Adminer

> A clean local database setup using Docker Compose — with environment variables, health checks, and a browser UI out of the box.

**Stack:** PostgreSQL 17 · Docker Compose · Adminer · Alpine Linux  
**Read time:** ~5 minutes

---

Running a local database shouldn't require a full install. With Docker Compose, you get an isolated, reproducible Postgres instance that spins up in seconds — and tears down just as cleanly.

This guide walks through setting it up from scratch, covering environment config, volumes for persistence, health checks, and Adminer for a quick browser UI.

---

## Prerequisites

You need Docker Desktop (or Docker Engine + Compose plugin) installed. That's it.

---

## Project structure

```
your-project/
├── .env
├── compose.yaml
└── data/
    └── db/        ← postgres data lives here
```

> **Key rule:** `.env` and `compose.yaml` must live in the same directory. Compose auto-loads `.env` only when they're co-located — if they're in different folders, variables won't resolve and you'll get empty environment errors.

---

## Step 1 — Create your .env file

Keep credentials out of your compose file. Define them once in `.env`:

```env
DB_NAME=postgres
DB_USER=kavinn
DB_PASSWORD=passcode@2005
DB_PORT=5432
```

> Add `.env` to your `.gitignore`. Never commit credentials, even for local dev.

---

## Step 2 — Write compose.yaml

Two services: `db` (Postgres) and `database-ui` (Adminer). Compose puts them on the same network automatically so they can talk by service name.

```yaml
services:
  db:
    image: postgres:17-alpine
    container_name: postgres
    restart: always
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - ${DB_PORT}:5432
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${DB_NAME} -U $${DB_USER}"]
      interval: 10s
      timeout: 30s
      retries: 5
    volumes:
      - ./data/db:/var/lib/postgresql/data
    attach: false

  database-ui:
    image: adminer
    restart: always
    ports:
      - 8080:8080
    depends_on:
      db:
        condition: service_healthy
```

### What each part does

**1. Environment block** — Postgres reads `POSTGRES_PASSWORD` only on first init. If the data directory already exists, it skips this step and just boots.

**2. Healthcheck** — `pg_isready` polls until Postgres accepts connections. Note the `$$` — inside `compose.yaml`, double dollar signs escape the variable so it's passed literally to the shell inside the container, not interpolated by Compose itself.

**3. Volume** — `./data/db` persists data between container restarts. Deleting this folder resets the database completely.

**4. depends_on with condition** — Adminer only starts after the `db` healthcheck passes. Without this, Adminer may try to connect before Postgres is ready.

---

## Step 3 — Start the stack

```bash
docker compose up -d
```

On first run, Compose pulls both images, initializes the Postgres data directory with your credentials, and starts both services. Give it 10–15 seconds for Postgres to finish init.

To confirm everything is healthy:

```bash
docker compose ps
```

The `db` service should show `(healthy)` in the status column.

---

## Step 4 — Open Adminer

Navigate to `http://localhost:8080` and fill in the connection form:

| Field    | Value                      |
|----------|----------------------------|
| System   | PostgreSQL                 |
| Server   | `db` — not localhost       |
| Username | doremonn                   |
| Password | doremonnn1940              |
| Database | postgres                   |

> **Why `db` and not `localhost`?** Inside Docker's network, services reach each other by their service name — not by `localhost`, which would point to Adminer's own container. The hostname `db` resolves to your Postgres container's internal IP automatically.

---

## Useful commands

```bash
# Stop and remove containers (data is preserved in ./data/db)
docker compose down

# View live logs from Postgres
docker logs -f postgres

# Confirm env vars are being picked up
docker compose config

# Connect via psql directly
docker exec -it postgres psql -U kavinn -d postgres
```

---

## Wrapping up

You now have a fully persistent Postgres 17 instance running locally with zero system install — just Docker. Adminer gives you a quick visual layer for inspecting tables and running queries during development.

From here you can add your app service to the same `compose.yaml` and connect it to `db` using the same service-name hostname.
