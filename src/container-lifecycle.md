# Container Lifecycle

## `docker run -d -p 8080:3000 --name myapp myimage`

This single command does a lot. Let's break down every piece:

```
docker run                          →  Create a container and start it
  -d                                →  Detached mode (run in background)
  -p 8080:3000                      →  Port mapping (host:container)
  --name myapp                      →  Give it a name instead of a random one
  myimage                           →  The image to create the container from
```

What's actually happening under the hood:

1. Docker creates a **writable layer** on top of the image's read-only layers
2. Sets up an **isolated environment** using kernel features:
   - New **PID namespace** — the process sees itself as PID 1
   - New **mount namespace** — the image layers become the container's root filesystem
   - New **network namespace** — the container gets its own network stack
   - New **UTS namespace** — the container gets its own hostname
   - **Cgroup limits** — CPU and memory constraints (if specified)
3. Creates an **iptables rule** on the host: "forward traffic from host port 8080 to this container's port 3000"
4. Runs the image's `CMD` or `ENTRYPOINT` as PID 1 inside the container

The `-p 8080:3000` flag is worth understanding deeply:

```
  Your browser                    Host machine                     Container
  ───────────                     ────────────                     ─────────
  localhost:8080  ──────────→  iptables catches it  ──────────→  port 3000
                               on host port 8080                 (container's
                               and forwards to the               network namespace)
                               container's namespace
```

Without `-p`, the container's port 3000 exists only inside its network namespace — invisible to the host.

## `docker create myimage`

Same as `docker run` but **skips step 4** — it sets up the container (writable layer, namespaces, network) but doesn't start the process. The container sits in "Created" state until you run `docker start`.

## `docker start myapp`

Starts a stopped or created container. The writable layer and configuration already exist — Docker just launches the process with the existing namespace setup.

## `docker stop myapp`

Sends **SIGTERM** to PID 1 inside the container. If the process doesn't exit within 10 seconds (configurable with `-t`), Docker sends **SIGKILL**.

The container still exists after stopping — its writable layer, logs, and configuration are preserved. You can `docker start` it again.

## `docker rm myapp`

Removes a stopped container. This is when the **writable layer is deleted** — any files you modified or created inside the container are gone forever. The image is unaffected.

This is why data disappears: `docker compose down` both stops AND removes containers. The writable layers are gone. Next `docker compose up` creates fresh containers from the same image.

## `docker run` vs `docker start` — The Key Difference

```
docker run   =  docker create  +  docker start    (new container every time)
docker start =  resume an existing stopped container (same writable layer)
```

Running `docker run` repeatedly creates a **new container** each time. Old ones pile up in stopped state unless you add `--rm` (which auto-removes the container when it stops).
