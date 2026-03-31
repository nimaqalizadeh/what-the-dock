# Seeing It With Your Own Eyes

Everything we discussed — containers as processes, namespaces, cgroups —
you can observe it all yourself on a Linux machine (or inside Docker
Desktop's VM).

## 1. A container is just a process on your host

First, pull the image (this downloads the image layers to your machine):

```bash
docker pull nginx
```

Then start a container in the background:

```bash
docker run -d --name mycontainer nginx
```

Now check your host's process list:

```bash
ps aux | grep nginx
```

There it is. Your "container" sitting in the normal process table with a
regular PID. Not inside a virtual machine, not in a special sandbox — a
process.

## 2. See the namespace links

Find the container's PID on the host:

```bash
docker inspect --format '{{.State.Pid}}' mycontainer
# e.g. output: 48372
```

Now look at that PID's namespace links:

```bash
ls -la /proc/48372/ns/
```

Output looks like:

```
mnt   ->  mnt:[4026532465]     filesystem isolation
pid   ->  pid:[4026532470]
net   ->  net:[4026532537]
user  ->  user:[4026531837]
uts   ->  uts:[4026532468]
ipc   ->  ipc:[4026532469]
```

Each line is a symlink pointing to a **namespace identifier** (the number
in brackets). Here's what each one controls:

Let's break this down piece by piece:

`/proc/` is a special directory in Linux. It's not a real folder with
real files — it's a window into what the kernel is doing right now.
Every running process gets a folder inside `/proc/` named after its PID.
So `/proc/48372/` contains everything the kernel knows about process
48372 (our container).

`/proc/48372/ns/` is the subfolder that shows which **namespaces** this
process belongs to. The `ns` stands for "namespace."

Each entry is a symlink (a shortcut) that points to a namespace ID:

- `mnt -> mnt:[4026532465]`
  **Mount namespace.** Controls which files and folders the process can
  see. The number `4026532465` is like a room number — it tells you
  which "version" of the filesystem this process lives in. The container
  is in room `4026532465`, the host is in a different room, so they see
  different files.

- `net -> net:[4026532568]`
  **Network namespace.** Controls the network. The container gets its
  own IP address, its own ports. It's in network room `4026532568`,
  separate from the host's network room.

- `pid -> pid:[4026532466]`
  **PID namespace.** Controls the process list. The container is in PID
  room `4026532466`, so it only sees processes in that same room. That's
  why it thinks it's PID 1 — it can't see the thousands of processes in
  the host's PID room.

- `user -> user:[4026531837]`
  **User namespace.** Controls user identities. Root (UID 0) inside the
  container gets remapped to a regular user (UID 1000) on the host.

- `uts -> uts:[4026532468]`
  **UTS namespace.** Controls the **hostname**. This is why a container
  can have its own hostname (like `a3f2b1c9d4e7`) instead of showing
  the host's hostname. UTS stands for "Unix Time-Sharing" (a legacy
  name).

- `ipc -> ipc:[4026532469]`
  **IPC namespace.** Controls **inter-process communication** — shared
  memory, message queues, semaphores. Processes in the container can
  only communicate via IPC with other processes in the same IPC
  namespace, not with host processes.

Think of it like hotel rooms. Every guest (process) is assigned to
specific rooms (namespace IDs). Guests in the same room can see each
other. Guests in different rooms are invisible to each other. The kernel
is the hotel — it manages who is in which room.

The numbers are just kernel-internal IDs. What matters is: if two
processes have the **same number** for a namespace, they share that view.
If the numbers are **different**, they're isolated from each other.

This is literally what makes a process a "container" — not Docker, not
a runtime, but these namespace links that the kernel attached to a
regular Linux process.

## 3. Compare with a host process

```bash
ls -la /proc/1/ns/
```

The namespace IDs are different. That's the isolation — same kernel, but
the container process lives in different namespaces than host processes.

## Every process has namespace assignments

**Every single process** on a Linux system has all the namespace
assignments. There's no process without them. Even your regular host
processes (your shell, text editor, browser) have namespace entries in
`/proc/PID/ns/`. They just all share the **same default namespaces** —
the host's "rooms."

```
Process         PID ns          Mount ns        Network ns       User ns
─────────       ──────          ────────        ──────────       ───────
bash            [4026531836]    [4026531840]    [4026531992]     [4026531837]
vim             [4026531836]    [4026531840]    [4026531992]     [4026531837]
chrome          [4026531836]    [4026531840]    [4026531992]     [4026531837]
                     ↑               ↑               ↑               ↑
              all the same     all the same     all the same     all the same
              = they can see   = they see the   = they share     = same user
                each other       same files       ports            mappings

container1      [4026532466]    [4026532465]    [4026532568]     [4026531837]
container2      [4026532470]    [4026532471]    [4026532789]     [4026531837]
                     ↑               ↑               ↑               ↑
               different from  different from  different from    might share
               host & each     host & each     host & each      the same one
               other            other            other
```

A "container" isn't a special type of process. It's a regular process
that got assigned to **different namespaces** than the default ones.
That's it. The kernel treats all processes the same way — every one has
a room assignment for every namespace type.

## Same namespace ID = same view (port conflict example)

Processes that share the **same namespace ID** are in the same "room" —
they can see each other in that dimension. This is how `docker exec`
works: it starts a new process with the **same namespace IDs** as the
existing container process, so they share the same filesystem, network,
and process tree.

This also explains port conflicts. All host processes share the same
network namespace ID, so they share the same port space:

```
Host processes (same network namespace):
  nginx      net:[4026531992]  ──┐  Same room = port 80 conflict!
  apache     net:[4026531992]  ──┘

Containers (different network namespaces):
  container1 net:[4026532568]  ──  own room = port 80 ✓
  container2 net:[4026532789]  ──  own room = port 80 ✓
```

- Two host processes **can't** both bind to port 80 — same network room
- Two containers **can** both bind to port 80 — different network rooms

That's also why you need the `-p` flag to bridge them. The host can't
see the container's port 80 because it's in a different network
namespace. `-p 8080:80` tells the kernel: "when something hits port
8080 in the host's network room, forward it to port 80 in the
container's network room."

## 4. See PID namespace in action

From the host, get the container's PID:

```bash
docker inspect --format '{{.State.Pid}}' mycontainer
# 48372
```

Now ask the container what PID it thinks it is:

```bash
docker exec mycontainer cat /proc/1/cmdline
# nginx: master process ...
```

The container's main process sees itself as **PID 1**. The host sees it as
**PID 48372**. Same process, two views.

## 5. See mount namespace in action

What the container sees as its root filesystem:

```bash
docker exec mycontainer ls /
# bin  dev  etc  home  lib  media  mnt  opt  proc  root  ...
```

Where those files actually live on the host:

```bash
docker inspect --format '{{.GraphDriver.Data.MergedDir}}' mycontainer
# /var/lib/docker/overlay2/abc123.../merged
ls /var/lib/docker/overlay2/abc123.../merged/
# bin  dev  etc  home  lib  media  mnt  opt  proc  root  ...
```

Same files. The container sees them at `/`. The host sees them at a
nested overlay path. The mount namespace remaps the view.

## 6. See cgroups in action

Check the memory limit for a container:

```bash
docker run -d --name limited --memory=256m nginx
cat /sys/fs/cgroup/docker/<container-id>/memory.max
# 268435456  (256 MB in bytes)
```

The kernel is enforcing that this process can't use more than 256 MB.
Try to exceed it and the kernel kills the process (OOM killed).

## 7. Create a container without Docker — What's Under the Hood

One Linux command. No Docker, no runtime, no images:

```bash
$ sudo unshare --pid --fork --mount-proc bash
# echo $$
1
```

The `bash` process thinks it's **PID 1**. The prompt changed from `$`
to `#` (running as root). And `echo $$` (which prints the current
process's PID) returns `1`.

Let's break down each flag:

- `--pid` — create a new PID namespace (the process gets its own
  process tree)
- `--fork` — fork a new child process into that namespace
- `--mount-proc` — mount a new `/proc` so commands like `ps` and
  `echo $$` reflect the new PID namespace instead of the host's

Now run `ps` inside:

```
  PID TTY          TIME CMD
    1 pts/0    00:00:00 bash
    2 pts/0    00:00:00 ps
```

Two processes. The host's thousands of other processes are invisible.
This process thinks it's PID 1 on a fresh machine.

**This is what a container is at its core.** Docker just adds layers
on top:

- Image management (layers, registries, pull/push)
- Networking (bridge networks, DNS, port mapping)
- Volume management
- A nice CLI and Docker Compose

But underneath all of that, it's `unshare` + `setns()` — kernel
namespace manipulation on regular Linux processes.

## 8. See what docker exec really does

Start a container, then exec into it:

```bash
docker run -d --name mybox alpine sleep 3600
docker exec -it mybox sh
```

What actually happened? Docker called the `setns()` syscall — it started
a new process (`sh`) and put it in the **same namespaces** as the
existing container process (`sleep 3600`). Same filesystem view, same
network, same PID tree, same cgroup limits.

You can verify — from the host, both processes share the same namespace
IDs:

```bash
SLEEP_PID=$(docker inspect --format '{{.State.Pid}}' mybox)
SH_PID=$(ps aux | grep "sh" | grep -v grep | awk '{print $2}')

ls -la /proc/$SLEEP_PID/ns/pid
ls -la /proc/$SH_PID/ns/pid
# Same namespace ID
```

There's no box being entered. No boundary being crossed. Just a new
process joining the same namespace assignments.
