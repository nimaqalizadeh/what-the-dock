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

## Port conflicts explained

Processes that share the **same network namespace** share the same port
space. That's why two host processes can't both bind to port 80 — they're
in the same network "room":

```
Host processes (same network namespace):
  nginx      net:[4026531992]  ──┐  Same room = port 80 conflict!
  apache     net:[4026531992]  ──┘

Containers (different network namespaces):
  container1 net:[4026532568]  ──  own room = port 80 ✓
  container2 net:[4026532789]  ──  own room = port 80 ✓
```

That's also why you need the `-p` flag to bridge them. The host can't
see the container's port 80 because it's in a different network
namespace. `-p 8080:80` tells the kernel: "when something hits port
8080 in the host's network room, forward it to port 80 in the
container's network room."

## See it yourself

Run two containers both binding to port 80 — no conflict:

```bash
docker run -d --name web1 nginx
docker run -d --name web2 nginx
```

Both are running nginx on port 80 inside their own network namespace.
Verify they each have their own IP:

```bash
docker inspect --format '{{.NetworkSettings.IPAddress}}' web1
# 172.17.0.2
docker inspect --format '{{.NetworkSettings.IPAddress}}' web2
# 172.17.0.3
```

Different IPs, both listening on port 80 — because each lives in its own
network namespace. Now expose them to the host on different ports:

```bash
docker run -d -p 8080:80 --name web3 nginx
docker run -d -p 8081:80 --name web4 nginx
```

Port 8080 on the host forwards to port 80 in web3's network namespace.
Port 8081 on the host forwards to port 80 in web4's network namespace.
The `-p` flag bridges the gap between the host's network room and each
container's network room.
