# The Shared Kernel Problem — CVE-2019-5736

The shared kernel isn't just a design trade-off — it's a **proven attack
surface**. In 2019, CVE-2019-5736 hit **runc**, the low-level runtime that
both Docker and Kubernetes use:

- A container process could **overwrite the host's runc binary**
- This gave the attacker **root access on the host machine**
- It affected Docker, Kubernetes, and anything else using runc

It was patched, but it proved that container breakouts are real, not
theoretical.

## Why Untrusted Code Should Run on VMs, Not Containers

This is the key insight: **containers isolate with namespaces (a kernel
feature), VMs isolate with a hypervisor (a hardware boundary)**.

```
  Container breakout:                VM breakout:

  Attacker ──> kernel vulnerability  Attacker ──> hypervisor vulnerability
               (large attack surface)              (tiny attack surface)
               Shared with ALL                     Each VM has its own
               containers on the host              kernel — blast radius
                                                   is one VM only
```

If someone finds a kernel bug, **every container on that host** is
potentially compromised. With VMs, each one has its own kernel — a
compromised VM is stuck inside itself.

This is why companies running untrusted code don't rely on containers alone:

- **AWS Lambda** — uses **Firecracker** (lightweight microVMs) to isolate
  each customer's functions. Different customers always get separate VMs.
- **Google Cloud** — uses **gVisor**, an application kernel in user space
  that reimplements Linux syscalls so they never reach the real host kernel.
- **Fly.io** — also uses Firecracker.

**The pattern:** when you can't trust the code, you add something stronger
than namespaces and cgroups. For most workloads (your own code, trusted
images), container isolation is fine. But for multi-tenant environments
running code from different customers, VMs provide the hard boundary
containers can't guarantee.

In production, you typically use **both**: VMs for the infrastructure
boundary, containers for application packaging. Complementary tools for
different layers.
