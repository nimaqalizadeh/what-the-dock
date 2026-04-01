# Docker Compose

## Why Compose Exists

For anything beyond a single container, managing everything with CLI
commands is miserable. Look at what it takes to run a typical stack
manually:

```bash
docker network create my-network
docker volume create db-data
docker build -t my-api ./api
docker run -d --name db --network my-network \
    -v db-data:/var/lib/postgresql/data \
    -e POSTGRES_PASSWORD=pass \
    -e POSTGRES_USER=user \
    -e POSTGRES_DB=mydb \
    postgres:16
docker run -d --name api --network my-network \
    -p 3000:3000 \
    -e DATABASE_URL=postgres://user:pass@db:5432/mydb \
    --restart unless-stopped \
    my-api
```

That's a lot of flags, and you'd have to remember them every time.
Docker Compose lets you define your entire application stack — every
service, every network, every volume — in a single YAML file.

## `docker compose up`

What you see: your whole stack starts with one command.

What Docker Compose actually does — it translates your
`docker-compose.yml` into individual Docker commands:

```yaml
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/mydb
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: pass
      POSTGRES_USER: user
      POSTGRES_DB: mydb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

Two services: `api` and `db`. The API builds from a local Dockerfile.
The database uses the official Postgres image. A named volume keeps
the database data persistent. Environment variables configure both
services.

Compose translates this into:

```
1. docker network create myproject_default        ←  custom network (auto DNS)
2. docker volume create myproject_pgdata           ←  named volume
3. docker build -t myproject-api .                 ←  build from Dockerfile
4. docker run --network myproject_default           ←  start db first
              --name myproject-db-1
              -v myproject_pgdata:/var/lib/postgresql/data
              -e POSTGRES_PASSWORD=pass
              postgres:16
5. (wait for db health check to pass)
6. docker run --network myproject_default           ←  then start api
              --name myproject-api-1
              -p 3000:3000
              myproject-api
```

Compose is not a different system. It's not a separate orchestrator.
It's just a convenient way to run the same `docker build`, `docker run`,
`docker network create`, and `docker volume create` commands you'd run
manually. It takes your declarative YAML file and translates it into
those imperative Docker commands.

Because both containers are on the same custom network, the API can
connect to the database at `db:5432` — Compose uses the service name
as the hostname. This is exactly the DNS mechanism we discussed in the
networking chapter. You don't need to configure it yourself.

## `depends_on` — Startup Order vs Readiness

`depends_on` tells Compose to start the database **before** the API.
But watch out — in its short form, it only controls **startup order**:

```yaml
depends_on:
  - db
```

This does **not** wait for the database to actually be ready. Docker
will start the Postgres container first and then immediately start the
API — even if Postgres hasn't finished initializing. Your API might
crash on its first connection attempt because the database isn't
accepting connections yet.

The fix is to use the long form with `condition: service_healthy`:

```yaml
depends_on:
  db:
    condition: service_healthy
```

This tells Compose to actually wait for the `db` service's health check
to pass before starting the API. You'll need a `healthcheck` defined on
the `db` service (like the `pg_isready` check in the example above).

Alternatively, build retry logic into your application's database
connection code so it can handle a brief delay. Either way, don't
assume `depends_on` means "ready."

## `docker compose down`

The reverse of `up`:

1. Stops all containers (SIGTERM → SIGKILL)
2. **Removes** all containers (writable layers are gone)
3. Removes the custom network

Crucially: **named volumes survive** `docker compose down`. Your database data is safe. Add `-v` to also remove volumes — but that deletes everything.

## `docker compose up --build`

Forces a rebuild of images before starting. Without `--build`, Compose reuses the existing image even if your Dockerfile or source code changed.

## `docker compose down` vs `docker compose stop`

```
docker compose stop    →  stops containers (writable layers preserved, can restart)
docker compose down    →  stops AND removes containers (writable layers deleted)
docker compose down -v →  stops, removes containers, AND deletes volumes (all data gone)
```
