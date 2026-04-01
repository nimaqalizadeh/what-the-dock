# Networking

Container networking solves two problems:

1. How does your **host machine** talk to a container?
2. How do **containers** talk to each other?

## Host to Container — Port Mapping

Remember, a container has its own isolated network namespace. If your
app is listening on port 3000 inside the container, that port is only
visible inside that container. Your host machine can't reach it. If you
open `localhost:3000` in your browser, nothing happens.

The `-p` flag sets up **port forwarding** — using `iptables` rules
(DNAT) on Linux to grab traffic on a host port and redirect it to a
container's port:

```bash
docker run -p 3000:3000 myapp
```

This means: when something hits port 3000 on my host, forward it to
port 3000 inside the container. The syntax is always **host:container**.

```
  Host                                       Container
  ────                                       ─────────
  localhost:3000  ── port forwarding ──→      :3000 (app listening)
                    (iptables DNAT)
                         │
                         └── packet handed to the bridge
                             network to reach the container
```

Two things work together here:

- **Port forwarding** (`-p`) — makes the container reachable from
  outside. Without it, the packet never enters Docker's internal
  network.
- **Bridge network** (`docker0`) — the internal virtual switch that
  routes the packet to the right container. Without it, the forwarded
  packet has nowhere to go.

The numbers don't have to match:

```bash
docker run -p 8080:3000 myapp
```

Host port 8080, container port 3000. This is how you run multiple
copies of the same container — they all listen on port 3000 internally,
but you map each one to a different host port:

```bash
docker run -p 8080:3000 --name app1 myapp
docker run -p 8081:3000 --name app2 myapp
docker run -p 8082:3000 --name app3 myapp
```

Three containers, all listening on port 3000 inside (different network
namespaces, no conflict), exposed on ports 8080, 8081, 8082 on the host.

### See it yourself

```bash
docker run -d -p 8080:80 --name web nginx
curl localhost:8080
# <!DOCTYPE html> ... nginx welcome page
```

Port 80 inside the container is invisible to the host. The `-p 8080:80`
bridge makes it reachable at `localhost:8080`.

## Container to Container — Networks

Your web server needs to talk to your database. Your API needs to talk
to your cache. Docker solves this with **networks**.

### The default bridge

When you install Docker, it creates a default network called `bridge`.
Any container you run gets attached to this network unless you say
otherwise. Containers on the same network can reach each other — but
there's a caveat: on the default bridge, containers can only reach each
other by **IP address**. That's fragile because every time a container
is recreated (`docker rm` + `docker run`, or a restart), Docker assigns
it a **new IP** from the network pool. If your API container has
`172.17.0.3` hardcoded as the database address, and the database
container gets recreated and comes back as `172.17.0.5`, your API is
now connecting to nothing.

It's a catch-22: on the default bridge there's no DNS, so you're
forced to use IPs — but IPs aren't stable. That's exactly what custom
networks solve.

### Custom networks — DNS resolution

On a custom network (one you create yourself), Docker provides
**automatic DNS resolution**. Containers can reach each other **by
name**:

```bash
docker network create mynet

docker run -d --network mynet --name db postgres
docker run -d --network mynet --name api myapp
```

Now inside the `api` container, you can connect to the database at
`db:5432`. Docker resolves the name `db` to the correct container IP
automatically. No hard-coded addresses, no fragile configuration.

```
Container "api" on mynet          Container "db" on mynet
─────────────────────────          ─────────────────────────
curl http://db:5432          →    Docker DNS resolves "db"
                                  to the container's IP
                                  on the mynet bridge
```

### See it yourself

```bash
docker network create testnet

docker run -d --network testnet --name server nginx
docker run --rm --network testnet alpine sh -c "ping -c 2 server"
# PING server (172.18.0.2): 56 data bytes
# 64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.123 ms
```

The `alpine` container resolved `server` by name — no IP needed.

### Network isolation

Networks also provide **isolation between groups of containers**. Your
frontend containers don't need to talk to your database directly. Put
them on different networks:

```bash
docker network create frontend
docker network create backend

docker run -d --network frontend --name web myfront
docker run -d --network backend  --name api myapi
docker run -d --network backend  --name db postgres

# Connect api to both networks (it bridges frontend and backend)
docker network connect frontend api
```

```
  frontend network              backend network
  ┌────────────────┐            ┌────────────────┐
  │  web ← → api   │            │  api ← → db    │
  └────────────────┘            └────────────────┘

  web can reach api             api can reach db
  web CANNOT reach db           db CANNOT reach web
```

The frontend talks to the API. The API talks to the database. The
frontend can't reach the database at all. Clean separation.

## Always Use Custom Networks

The default bridge network works but isn't the best choice. Always
create a custom network for your application:

- **DNS resolution** — containers find each other by name
- **Better isolation** — only containers on the same network can
  communicate
- **More control** — you decide which containers can talk to which

If you've ever wondered how Docker Compose services find each other by
name — this is exactly the mechanism. Compose creates a custom network
behind the scenes. We'll see that in the next section.
