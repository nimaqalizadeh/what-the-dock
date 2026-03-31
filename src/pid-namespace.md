# PID Namespace

This is the core trick that makes a container feel like its own machine.
There is only **one process**, but the kernel presents **two different views** of it:

```
        ── PID Namespace ──────────────────────────────────────────

  ┌──────────────────────┐ │ ┌──────────────────────────────────┐
  │   Container's view   │ │ │          Host's view             │
  │                      │ │ │                                  │
  │   ┌──────────────┐   │ │ │   ┌─────────┐    ┌─────────┐     │
  │   │    PID 1     │   │ │ │   │  PID 1  │    │   ...   │     │
  │   └──────────────┘   │ │ │   └─────────┘    └─────────┘     │
  │                      │ │ │         ┌─────────────┐          │
  │  "I just booted!"    │ │ │         │  PID 48371  │          │
  │                      │ │ │         └─────────────┘          │
  │                      │ │ │   ┌──────────────────────┐       │
  │                      │ │ │   │      PID 48372       │       │
  │                      │ │ │   └──────────────────────┘       │
  └──────────────────────┘ │ └──────────────────────────────────┘
                           │
              ╰─ ─ ─ ─ same process, two views ─ ─ ─ ─╯
```

- **Left (Container's view):** The process sees itself as **PID 1** — as if
  it just booted a fresh machine. It has no idea thousands of other processes
  exist. Its entire process tree starts from itself.

- **Right (Host's view):** The host sees the full picture — all its normal
  processes (PID 1, PID 48371, etc.) plus the container's process sitting
  right there as **PID 48372**. It's just another entry in the process table.

The process didn't move. It didn't enter a box. The kernel simply gave it a
**separate PID namespace** so it counts from 1 in its own isolated view while
the host still tracks it by its real PID. This is what namespaces are — the
kernel deciding what each process is allowed to see.

## What does being PID 1 actually mean?

PID 1 does **not** mean the process thinks it's the only process. It means
it thinks it's the **first** process — the root of its own process tree.

On a real Linux machine, PID 1 is the **init process** (like `systemd`) —
the very first process that starts when the OS boots, and every other process
is a descendant of it. So when a container's main process sees itself as
PID 1, it behaves as if a fresh OS just booted and it's the starting point.

The container **can** have multiple processes. If PID 1 spawns children,
they appear as PID 2, PID 3, etc. — all within the container's namespace.
But it can't see the host's thousands of other processes. PID 1 means
"I am the root of this world" — not "I am alone."

## Consequences of being PID 1 in a namespace

- **It acts as the init process** — it's responsible for reaping zombie
  child processes. If it doesn't handle that, orphaned children pile up.
- **If PID 1 dies, the entire container stops** — just like on a real
  Linux machine where the init process dying brings the system down.
  Kill PID 1, the container is gone.
- **It can't see or interact with host processes** — it has no way to
  signal, inspect, or even know about processes outside its namespace.
  From its perspective, they don't exist.
- **Process IDs are independent** — two containers can both have a PID 5
  inside them without conflict, just like two containers can both bind
  to port 80 via network namespaces.

The isolation isn't about being alone — it's about having a **limited,
independent view**. Each namespace (PID, mount, network, user) removes one
more piece of visibility into the real host. Stack them all together and
the process _behaves_ as if it has the machine to itself — even though
it's just one process among thousands on the host.

## See it yourself

From the host, get the container's real PID:

```bash
docker run -d --name pidtest nginx
docker inspect --format '{{.State.Pid}}' pidtest
# 48372
```

Now ask the container what PID it thinks it is:

```bash
docker exec pidtest cat /proc/1/cmdline
# nginx: master process ...
```

The container's main process sees itself as **PID 1**. The host sees it as
**PID 48372**. Same process, two views — that's the PID namespace in action.

You can also see the full process list from inside the container:

```bash
docker exec pidtest ps aux
```

Only nginx processes appear. The host's thousands of other processes are
completely invisible — they exist in a different PID namespace.
