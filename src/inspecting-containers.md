# Inspecting Containers

## `docker ps`

Lists running containers. Add `-a` to see stopped containers too.

What each column means:

```
CONTAINER ID   IMAGE    COMMAND         STATUS          PORTS                    NAMES
a1b2c3d4e5f6   nginx    "nginx -g..."   Up 2 hours      0.0.0.0:8080->80/tcp     webserver
```

- **CONTAINER ID** — first 12 chars of the container's unique SHA256 hash
- **COMMAND** — the CMD/ENTRYPOINT that's running as PID 1
- **STATUS** — `Up` (running), `Exited` (stopped, writable layer still exists), `Created` (never started)
- **PORTS** — the `-p` mappings. `0.0.0.0:8080->80/tcp` means host port 8080 forwards to container port 80

## `docker logs myapp`

Reads the **stdout and stderr** of PID 1 inside the container. Docker captures these streams and stores them as JSON files on the host. Add `-f` to follow in real time (like `tail -f`).

## `docker exec -it myapp sh`

We covered this in the practical section — Docker calls `setns()` to start a new process (`sh`) in the **same namespaces** as the container's existing process. Same filesystem, same network, same PID tree.

The flags:

- `-i` — keep stdin open (so you can type commands)
- `-t` — allocate a pseudo-TTY (so you get a proper terminal prompt)

## `docker inspect myapp`

Dumps the container's full configuration as JSON — network settings, mount points, environment variables, PID on the host, cgroup paths, namespace IDs. This is the source of truth for everything Docker set up.
