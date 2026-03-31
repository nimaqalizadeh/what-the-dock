# Docker Compose

## `docker compose up`

What you see: your whole stack starts with one command.

What Docker Compose actually does — it translates your `docker-compose.yml` into individual Docker commands:

```yaml
services:
  api:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret

volumes:
  pgdata:
```

Compose translates this into:

```
1. docker network create myproject_default        ←  custom network (auto DNS)
2. docker volume create myproject_pgdata           ←  named volume
3. docker build -t myproject-api .                 ←  build from Dockerfile
4. docker run --network myproject_default           ←  start db first
              --name myproject-db-1
              -v myproject_pgdata:/var/lib/postgresql/data
              -e POSTGRES_PASSWORD=secret
              postgres:16
5. (wait for db health check to pass)
6. docker run --network myproject_default           ←  then start api
              --name myproject-api-1
              -p 3000:3000
              myproject-api
```

Because both containers are on the same custom network, the API can connect to the database at `db:5432` — Compose uses the service name as the hostname.

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
