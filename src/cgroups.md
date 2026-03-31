# Cgroups — Resource Limits for Containers

Without cgroups, a container is just a regular process — it could use all
the CPU and eat all the RAM on the host, starving every other container
and the host itself.

Cgroups set **hard resource limits** so a container can't consume more
than its fair share:

- **CPU** — "This container gets at most 2 CPU cores" — even if the host
  has 16
- **Memory** — "This container gets at most 1 GB of RAM" — if it tries
  to use more, the kernel kills it (OOM killed)
- **Disk I/O** — "This container gets limited disk throughput" — it can't
  saturate the disk

## Namespaces vs Cgroups — The Complete Picture

These two kernel features work together to make a container:

```
  Namespaces = what you can SEE        Cgroups = what you can USE
  (isolation)                          (resource limits)

  ┌────────────────────────┐           ┌────────────────────────┐
  │ PID:     own process   │           │ CPU:    max 2 cores    │
  │          tree           │           │                        │
  │ Mount:   own filesystem │           │ Memory: max 1 GB      │
  │                        │           │                        │
  │ Network: own IP/ports  │           │ Disk:   limited I/O    │
  │                        │           │                        │
  │ User:    own user IDs  │           │                        │
  └────────────────────────┘           └────────────────────────┘
        the WALLS                           the RATIONS
```

Namespaces are the **walls** — they control what the process can see.
Cgroups are the **rations** — they control how much the process can
consume. You need both to make a container behave.

You can even create a container without Docker using a single Linux command: `unshare`. The process thinks it's PID 1 on a fresh machine. Docker just wraps this in image management, networking, and a nice CLI.
