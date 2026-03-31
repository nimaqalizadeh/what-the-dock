# Network Namespace

Just like PID and mount namespaces, network namespaces give each container
its own **isolated network stack** — its own IP address and its own ports.

```
            ── Network Namespace ──────────────────────────

  ┌──────────────────────┐       ┌──────────────────────┐
  │     Container 1      │       │     Container 2      │
  │                      │       │                      │
  │    IP: 172.17.0.2    │       │    IP: 172.17.0.3    │
  │    Port: :80         │       │    Port: :80         │
  └──────────────────────┘       └──────────────────────┘

            Both bind to :80 — no conflict!
```

Without network namespaces, two processes can't both listen on port 80 on
the same machine — the second one would fail with "port already in use."
But because each container has its own network namespace, each one gets
its own IP address and its own port space. They don't know about each
other's network at all.

It's the same pattern across all namespaces:

- **PID namespace** — two containers can both have PID 1, no conflict
- **Mount namespace** — two containers can both have different files at
  `/`, no conflict
- **Network namespace** — two containers can both bind to port 80, no
  conflict

Same trick every time: the kernel gives each container its own isolated
view of that resource.
