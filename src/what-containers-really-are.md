# What Containers Really Are

## What a Container IS

A container is a **regular Linux process** running on the host operating system — the same kind of process as your shell or text editor. If you run a container and check `ps aux` on the host, you'll find it sitting right there in the normal process table.

### See it yourself

Pull an image and start a container:

```bash
docker pull nginx
docker run -d --name mycontainer nginx
```

Now check your host's process list:

```bash
ps aux | grep nginx
```

There it is. Your "container" sitting in the normal process table with a regular PID. Not inside a virtual machine, not in a special sandbox — a process.

**Images vs Containers:** An image is a blueprint; a container is a running instance of that blueprint. This is the same relationship as:

- **Class vs Object** — the image defines the structure, the container is an instantiation of it
- **Compiled binary vs Running process** — the image sits on disk, the container is that image actively executing

You can create many containers from the same image, just like creating many objects from one class. Each container is independent with its own writable layer, process space, and network identity, but they all start from the same base.

**What's inside an image?** An image is a **read-only, self-contained file on disk** — a complete package that bundles everything the application needs to run:

- **App Code** — your source code
- **Runtime** — the language runtime (Node, Python, Go, etc.)
- **System Libraries** — OS-level dependencies your app relies on
- **Environment** — environment variables (database URLs, API keys, feature flags)
- **Config Files** — configuration your app reads at startup

Because the image contains all of this, it doesn't depend on anything being installed on the host machine. The same image runs identically everywhere — that's what eliminates the "it works on my machine" problem.

**The "it works on my machine" problem:** A developer writes code that runs perfectly on their laptop, but it breaks on someone else's machine or in production. The cause is environment differences — different OS version, different runtime version (e.g., Node 18 vs Node 20), missing system libraries, different environment variables, or different config. Developer A says "it works on my machine," and Developer B can't reproduce it because their setup is different. Docker solves this because the **image is the machine** — everyone runs the same image with the same runtime, libraries, and config baked in. There's nothing left to differ between environments.

Images are **immutable** — they never change after they're built. Any modifications happen in the container's writable layer, not the image itself. When you remove or replace the container, those changes are gone. If you want to make a change that's **consistent and permanent**, change the Dockerfile and rebuild the image. The image is the source of truth — not the container.

## Container Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CONTAINERS                              │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │     App A       │  │     App B       │  │     App C       │  │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤  │
│  │   Libraries /   │  │   Libraries /   │  │   Libraries /   │  │
│  │   Dependencies  │  │   Dependencies  │  │   Dependencies  │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│         │                     │                     │           │
│         └─────── Namespaces + Cgroups ──────────────┘           │
│                   (isolation wrappers)                          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│              CONTAINER RUNTIME (Docker / Podman / ContainerD)   │
├─────────────────────────────────────────────────────────────────┤
│                   HOST OS (Single Shared Kernel)                │
├─────────────────────────────────────────────────────────────────┤
│                     PHYSICAL HARDWARE                           │
└─────────────────────────────────────────────────────────────────┘
```

Key difference from VMs: there is **no guest OS layer**. All containers share
one kernel. That's why they start in seconds and use far less memory — but
it also means a kernel vulnerability can potentially affect all containers.

What makes it a "container" is that the kernel wraps this process in **isolation mechanisms**:

- **Namespaces** - control what the process can **see**:
  - **PID namespace** - the process sees itself as PID 1 (on the host it's PID 48372)
  - **Mount namespace** - the process sees a completely different file system than the host
  - **Network namespace** - the process gets its own IP and ports
  - **User namespace** - root inside the container maps to an unprivileged user on the host

### See the namespace links

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
in brackets). `/proc/` is a special directory in Linux — it's not a real
folder with real files, it's a window into what the kernel is doing right
now. Every running process gets a folder inside `/proc/` named after its
PID. `/proc/48372/ns/` shows which **namespaces** this process belongs to.

### Compare with a host process

```bash
ls -la /proc/1/ns/
```

The namespace IDs are different. That's the isolation — same kernel, but
the container process lives in different namespaces than host processes.

### Every process has namespace assignments

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

This is literally what makes a process a "container" — not Docker, not
a runtime, but these namespace links that the kernel attached to a
regular Linux process.

## What a Container is NOT

- **Not a virtual machine** - it does NOT run a separate operating system. There is no guest kernel. All containers share the host's kernel.
- **Not a small VM** - calling it a "lightweight VM" is misleading. VMs virtualize hardware; containers virtualize the OS view.
- **Not a special sandbox or box** - when you `docker exec` into a container, you're not "entering" anything. The runtime simply calls `setns()` (a Linux syscall) to start a new process with the same namespace assignments as the existing container process.
- **Not its own machine** - it just _looks_ like one because the kernel lies to it about what exists.

### See what `docker exec` really does

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

### Create a container without Docker

One Linux command. No Docker, no runtime, no images:

```bash
$ sudo unshare --pid --fork --mount-proc bash
# echo $$
1
```

The `bash` process thinks it's **PID 1**. The prompt changed from `$`
to `#` (running as root). And `echo $$` (which prints the current
process's PID) returns `1`.

Each flag:

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
