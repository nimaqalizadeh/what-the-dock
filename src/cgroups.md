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

## See it yourself

Set a memory limit and inspect the cgroup:

```bash
docker run -d --name limited --memory=256m nginx
```

Check the kernel-enforced limit:

```bash
docker inspect --format '{{.HostConfig.Memory}}' limited
# 268435456  (256 MB in bytes)
```

On a Linux host, you can see the cgroup file directly:

```bash
cat /sys/fs/cgroup/docker/<container-id>/memory.max
# 268435456
```

The kernel is enforcing that this process can't use more than 256 MB.
Try to exceed it and the kernel kills the process (OOM killed).

Set a CPU limit:

```bash
docker run -d --name cpulimited --cpus=0.5 nginx
```

This container can only use half a CPU core. Run a CPU-intensive process
inside and watch `docker stats` — it won't exceed 50% of one core:

```bash
docker stats cpulimited --no-stream
```

Cgroups are the reason containers can't starve each other — the kernel
enforces the limits regardless of what the process tries to do.
