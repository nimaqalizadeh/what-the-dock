# Volumes

## Why Volumes Exist — COPY vs Mounts

In the image layers chapter we saw that `COPY` bakes files into the
image at **build time**. It's a one-time snapshot — if you change
`app.js` on your host afterwards, the image still has the old version.
You'd need to rebuild the image to pick up changes.

That works for production, where you want a self-contained image. But
during development you want to edit code on your host and see changes
inside the container **immediately**, without rebuilding.

That's what volumes and bind mounts are for — they connect a directory
on the host to a directory in the container at **runtime**, so both
sides see the same files on disk:

```
  COPY (build time)              Bind mount (runtime)
  ─────────────────              ────────────────────
  One-time copy into             Kernel mount — same
  the image layer.               files on disk.
  No sync after build.           Live sync both ways.

  Host: ./app/app.js             Host: ./app/app.js
        │                              │
        │ copied at build              │ mounted at runtime
        ▼                              ▼
  Image layer:                   Container:
  /app/app.js (frozen)           /app/app.js (live)
```

- **`COPY`** — production. Image is self-contained, depends on nothing
  from the host.
- **Bind mount** — development. Fast iteration without rebuilding.

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
