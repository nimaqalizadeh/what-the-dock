# User Namespace

User namespaces remap **user identities** between the container and the host.

```
            ── User Namespace ─────────────────────────────

  Inside container               On host
  ┌──────────────────────┐       ┌──────────────────────┐
  │                      │       │                      │
  │       root           │       │       user           │
  │     (UID 0)          │──────>│     (UID 1000)       │
  │                      │       │                      │
  └──────────────────────┘       └──────────────────────┘

              root inside ≠ root on host
```

- **Inside the container:** the process runs as `root` (UID 0) — full
  admin privileges, can install packages, modify system files, do anything
- **On the host:** that same process is actually `user` (UID 1000) — a
  regular unprivileged user with no special powers

This is a **security feature**. If an attacker breaks out of the container,
they don't land as root on the host — they land as an unprivileged user
(UID 1000) who can't do much damage. The container _thinks_ it has full
admin access, but the kernel remaps its UID so it doesn't actually have
it on the real system.

## All Four Namespaces — The Same Trick

Every namespace is the kernel remapping one identity so the container
feels powerful but is actually constrained:

```
  Namespace     Inside container      On host
  ─────────     ────────────────      ───────────────────────────
  PID           PID 1                 PID 48372
  Mount         /                     /var/lib/docker/overlay2/...
  Network       172.17.0.2:80         own isolated network stack
  User          root (UID 0)          user (UID 1000)
```

One process. One kernel. Four layers of illusion.

## See it yourself

Check who you are inside a container vs on the host:

```bash
docker run --rm alpine whoami
# root
docker run --rm alpine id
# uid=0(root) gid=0(root)
```

The process thinks it's running as root (UID 0) inside the container.
Now check the same process from the host's perspective:

```bash
docker run -d --name usertest alpine sleep 3600
HOST_PID=$(docker inspect --format '{{.State.Pid}}' usertest)
ps -o user= -p $HOST_PID
```

Depending on your Docker configuration, you'll see either `root` or a
remapped unprivileged user. With user namespace remapping enabled, the
container's UID 0 maps to a high-numbered UID on the host — the process
has no real root privileges on the host system.

You can also verify the user namespace ID:

```bash
ls -la /proc/$HOST_PID/ns/user
# user -> user:[4026531837]
```

Compare this with the host's init process to see if they share the same
user namespace or if the container has been remapped to its own.
