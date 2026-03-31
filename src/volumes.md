# Volumes

## `docker volume create mydata`

Creates a named volume — a directory managed by Docker (typically at `/var/lib/docker/volumes/mydata/_data` on Linux). This directory persists independently of any container.

## `docker run -v mydata:/app/data myimage`

Mounts the named volume `mydata` at `/app/data` inside the container. What's happening:

```
Host filesystem                          Container's mount namespace
─────────────────                        ──────────────────────────
/var/lib/docker/volumes/mydata/_data  →  /app/data
```

The container reads and writes to `/app/data`, but the data actually lives on the host in Docker's volume directory. Remove the container — the volume stays. Attach it to a new container — all the data is there.

## `docker run -v $(pwd)/src:/app/src myimage`

This is a **bind mount** (uses a host path instead of a volume name). Maps your local `./src` directory directly into the container. Changes on either side are visible immediately — this is what makes development without rebuilding possible.

Key difference from named volumes: bind mounts **obscure** whatever was at that path in the image. Named volumes **copy** image files into the volume on first use.
