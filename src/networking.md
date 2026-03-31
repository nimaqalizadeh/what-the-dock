# Networking

## `docker network create mynet`

Creates a custom bridge network with **DNS resolution** — containers on this network can reach each other by name.

## `docker run --network mynet --name api myimage`

Attaches the container to `mynet`. Now any other container on `mynet` can reach this one at hostname `api`.

```
Container "api" on mynet          Container "db" on mynet
─────────────────────────          ─────────────────────────
curl http://db:5432          →    Docker DNS resolves "db"
                                  to the container's IP
                                  on the mynet bridge
```

Without a custom network (on Docker's default bridge), containers can only reach each other by IP address — which changes every time a container is recreated.
