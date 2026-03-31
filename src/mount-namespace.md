# Mount Namespace

Just like PID namespaces give each container its own process tree, mount
namespaces give each container its own **completely separate filesystem view**.

```
            ── Mount Namespace ──────────────────────────────────

  Container's /                       Host's /
  ┌──────────────────────┐            ┌──────────────────────┐
  │ Ubuntu image         │            │ Host's own files     │
  │ filesystem           │            │                      │
  │  ┌────────────────┐  │            │  ┌────────────────┐  │
  │  │  /bin           │  │            │  │  /home          │  │
  │  ├────────────────┤  │            │  ├────────────────┤  │
  │  │  /etc           │  │            │  │  /var           │  │
  │  ├────────────────┤  │            │  ├────────────────┤  │
  │  │  /usr           │  │            │  │  /opt           │  │
  │  ├────────────────┤  │            │  ├────────────────┤  │
  │  │  /var           │  │            │  │  /srv           │  │
  │  └────────────────┘  │            │  └────────────────┘  │
  └──────────────────────┘            └──────────────────────┘
              │                                  │
              └──── Same kernel underneath ──────┘
```

When you `docker run ubuntu`, the process doesn't see your host's files.
It sees the **Ubuntu image's filesystem** mounted as its root: `/bin`,
`/etc`, `/usr`, `/var`. These come from the image layers. Meanwhile the
host has its own completely separate root filesystem: `/home`, `/var`,
`/opt`, `/srv`. The container can't see any of these.

There is no second OS running. The kernel is doing the same trick it does
with PID namespaces: **showing different views of the same system to
different processes**.

- **PID namespace trick:** One process table exists. The kernel shows the
  container a filtered slice — the container sees PID 1, while the host
  sees the same process as PID 48372. Same reality, filtered view.
- **Mount namespace trick:** One kernel manages all filesystems. The kernel
  shows the container a different root (`/`) than the host sees. Same
  kernel handling all file operations, but each process gets a different
  answer to "what's at `/`?"

The kernel isn't copying anything or creating separate systems. It's just
**lying to the process about what exists** — hiding some things and
remapping others depending on which namespace the process belongs to.

## How the path remapping actually works

The container's filesystem might physically live at something like
`/var/lib/docker/overlay2/abc123/` on the host. But the mount namespace
makes the container see that path as `/` — its root.

So when the container reads `/etc/config.json`, it's actually reading
`/var/lib/docker/overlay2/abc123/etc/config.json` on the host. The kernel
silently translates the path. The container has no idea.

```
  Container sees:           What's really happening on the host:

  /                    -->  /var/lib/docker/overlay2/abc123/
  /etc/config.json     -->  /var/lib/docker/overlay2/abc123/etc/config.json
  /usr/bin/node        -->  /var/lib/docker/overlay2/abc123/usr/bin/node
```

Same trick as PID — the process is PID 48372 on the host but sees itself
as PID 1. The files live at a nested host path but the container sees
them as `/`.

This is why:

- A container "has its own filesystem" without needing a VM
- Files you create on the host don't appear in the container (and vice versa)
- When you remove a container, its files disappear — they lived in the
  container's isolated mount, not on the host
- You need **volumes** or **bind mounts** to bridge the gap between the
  two views
