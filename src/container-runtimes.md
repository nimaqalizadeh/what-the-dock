# Container Runtimes — Docker and Its Alternatives

Docker isn't the only tool that can run containers. They all use the
**same underlying kernel primitives** (namespaces + cgroups). The
difference is in what they wrap around them:

- **Docker** — the most popular. Full tooling: CLI, image building,
  networking, Docker Compose. Uses containerd under the hood.
- **containerd** — not really a competitor, more a **lower-level
  component**. Docker itself uses containerd to manage containers.
  Kubernetes also uses containerd directly, skipping Docker entirely.
- **Podman** — Docker-compatible CLI (you can alias `docker=podman`)
  but **daemonless** — no background process needed. Also runs
  containers rootless by default (user namespaces), which is more secure.
- **LXC (Linux Containers)** — the original. Closer to lightweight VMs —
  runs a full init system inside. Docker was originally built on LXC
  before replacing it with its own runtime (libcontainer, now runc).

```
            Docker    Podman    containerd    LXC
              │         │          │           │
              └────┬────┘──────────┘───────────┘
                   │
         Namespaces + Cgroups (Linux kernel)
```

The difference is the **tooling on top** — image management, networking,
orchestration, security defaults. Under the hood, they all ask the kernel
for the same isolation.
