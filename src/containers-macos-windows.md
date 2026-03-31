# Containers on macOS and Windows — The Hidden VM

Containers need **Linux kernel features** — namespaces and cgroups. These
are Linux-specific:

```
  Container ────── ✗ ──────  Darwin Kernel (macOS)
                             doesn't have namespaces or cgroups

  Container ────── ✗ ──────  NT Kernel (Windows)
                             doesn't have namespaces or cgroups

  Container ────── ✓ ──────  Linux Kernel
                             namespaces + cgroups built in
```

So when you install Docker Desktop on a Mac or Windows machine, it quietly
runs a **lightweight Linux VM** in the background. Your containers run
inside that VM, not on your Mac/Windows directly.

```
  ┌─────────────────────────────────────────┐
  │  Your Mac / Windows                     │
  │                                         │
  │  ┌───────────────────────────────────┐  │
  │  │  Lightweight Linux VM             │  │
  │  │  (created by Docker Desktop)      │  │
  │  │                                   │  │
  │  │  ┌───────────┐  ┌───────────┐     │  │
  │  │  │ Container │  │ Container │     │  │
  │  │  └───────────┘  └───────────┘     │  │
  │  │                                   │  │
  │  │  Linux Kernel (namespaces+cgroups)│  │
  │  └───────────────────────────────────┘  │
  │                                         │
  │  Darwin / NT Kernel (no container       │
  │  features)                              │
  └─────────────────────────────────────────┘
```

This is also why:

- **Docker feels slower on Mac/Windows** — files are synced across the VM
  boundary (~3x slower than native Linux)
- **Networking has quirks** — traffic routes through the VM
- **On Linux, no VM is needed** — containers talk straight to the kernel,
  which is why Docker feels faster there

Docker itself isn't a VM, but on non-Linux systems it **needs one** to
provide the Linux kernel that containers require.
