# What Containers Really Are

## What a Container IS

A container is a **regular Linux process** running on the host operating system — the same kind of process as your shell or text editor. If you run a container and check `ps aux` on the host, you'll find it sitting right there in the normal process table.

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
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │     App A        │  │     App B        │  │     App C        │ │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤ │
│  │   Libraries /    │  │   Libraries /    │  │   Libraries /    │ │
│  │   Dependencies   │  │   Dependencies   │  │   Dependencies   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│         │                     │                     │           │
│         └─────── Namespaces + Cgroups ──────────────┘           │
│                   (isolation wrappers)                           │
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

## What a Container is NOT

- **Not a virtual machine** - it does NOT run a separate operating system. There is no guest kernel. All containers share the host's kernel.
- **Not a small VM** - calling it a "lightweight VM" is misleading. VMs virtualize hardware; containers virtualize the OS view.
- **Not a special sandbox or box** - when you `docker exec` into a container, you're not "entering" anything. The runtime simply calls `setns()` (a Linux syscall) to start a new process with the same namespace assignments as the existing container process.
- **Not its own machine** - it just _looks_ like one because the kernel lies to it about what exists.
