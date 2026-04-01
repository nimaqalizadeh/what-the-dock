# Volumes

## Why Volumes Exist

Containers are processes, and by default, when a container is removed,
everything inside it is gone — the writable layer disappears. But real
applications need persistent data. Databases need to store records.
Applications need to save uploads. Log files need to survive restarts.

That's what volumes are for — storage that lives **outside the
container's lifecycle**, managed by Docker, persisting even after the
container is removed.

In the image layers chapter we saw that `COPY` bakes files into the
image at **build time**. It's a one-time snapshot — if you change a
file on your host afterwards, the image still has the old version.
You'd need to rebuild the image to pick up changes.

```
  COPY (build time)              Volume/Bind mount (runtime)
  ─────────────────              ──────────────────────────
  One-time copy into             Kernel mount — same
  the image layer.               files on disk.
  No sync after build.           Live sync both ways.

  Host: ./src/main.rs            Host: ./src/main.rs
        │                              │
        │ copied at build              │ mounted at runtime
        ▼                              ▼
  Image layer:                   Container:
  /app/src/main.rs (frozen)      /app/src/main.rs (live)
```

There are two main ways to handle persistent storage: **named volumes**
and **bind mounts**.

## Named Volumes

Named volumes are fully managed by Docker. You create them, Docker
decides where they're stored on disk, and you reference them by name.

```bash
docker volume create mydata
```

This creates a directory at `/var/lib/docker/volumes/mydata/_data` on
Linux. This directory persists independently of any container.

```bash
docker run -v mydata:/var/lib/postgresql/data postgres
```

```
Host filesystem                                    Container's mount namespace
─────────────────                                  ──────────────────────────
/var/lib/docker/volumes/mydata/_data            →  /var/lib/postgresql/data
```

Postgres writes its data to `/var/lib/postgresql/data` inside the
container, but the data actually lives in the volume on the host.
Remove the container — the volume stays. Attach it to a new container
— all the data is still there.

Named volumes are ideal for **production data**: database files,
application state, anything that needs to survive container
replacements.

### See it yourself

```bash
# Create a volume and write data to it
docker run --rm -v mydata:/data alpine sh -c "echo 'persistent' > /data/test.txt"

# Container is gone. Start a new one with the same volume:
docker run --rm -v mydata:/data alpine cat /data/test.txt
# persistent  ← data survived the container removal
```

## Bind Mounts

Bind mounts map a **specific directory on your host** directly into
the container. These are what make Docker practical for development.
Without them, every time you change a line of code, you'd have to
rebuild the entire image to see the result.

```bash
docker run -v $(pwd)/src:/app/src myimage
```

Your local `./src` folder is now visible inside the container at
`/app/src`. Edit a file on your laptop and the container sees the
change immediately. No rebuild needed.

### See it yourself

```bash
mkdir /tmp/bindtest && echo "version 1" > /tmp/bindtest/config.txt

docker run -d --name bindbox -v /tmp/bindtest:/data alpine sleep 3600

# Container sees the file
docker exec bindbox cat /data/config.txt
# version 1

# Edit on host — container sees it instantly
echo "version 2" > /tmp/bindtest/config.txt
docker exec bindbox cat /data/config.txt
# version 2

# Edit in container — host sees it instantly
docker exec bindbox sh -c "echo 'version 3' > /data/config.txt"
cat /tmp/bindtest/config.txt
# version 3
```

## The Key Difference: Obscuring vs Copying

Bind mounts and named volumes behave differently when mounted to a
path that already has files in the image.

**Bind mounts obscure.** When you bind mount a directory to a path
inside the container, it completely **covers** whatever was at that
path in the image. If your image has files at `/app/src` and you bind
mount an empty directory there, those image files are hidden. They're
still in the image layer — they're just covered up by the mount.

```bash
# Image has files at /etc/nginx/
docker run --rm nginx ls /etc/nginx/
# nginx.conf  conf.d/  mime.types  ...

# Bind mount an empty directory over it — image files are hidden
mkdir /tmp/empty
docker run --rm -v /tmp/empty:/etc/nginx nginx ls /etc/nginx/
# (empty — image files are obscured)
```

**Named volumes copy on first use.** If you mount an **empty** named
volume to a path that already has files in the image, Docker copies
those image files into the volume the first time. After that, the
volume's contents take over.

```bash
# First use: empty volume → Docker copies image files into it
docker run --rm -v nginxconf:/etc/nginx nginx ls /etc/nginx/
# nginx.conf  conf.d/  mime.types  ...  ← copied from image

# Volume now has its own copy. If image updates, volume keeps old files.
```

This is how database images like Postgres seed their data directory —
the image contains initialization scripts, and on first use with an
empty volume, Docker copies them in. But it only happens **once**, on
the first use of an empty volume.

It's a subtle distinction, but it can be confusing when you expect bind
mounts and named volumes to behave the same way.

## When to Use Which

| | Named volumes | Bind mounts |
|---|---|---|
| **Managed by** | Docker | You (specific host path) |
| **Use case** | Production data (databases, uploads, state) | Development (live code editing) |
| **Persists after container removal** | Yes | Yes (it's your host directory) |
| **Empty mount on existing path** | Copies image files into volume | Obscures image files |
| **Portability** | Works anywhere Docker runs | Depends on host path existing |
