# Virtual Machines

## What problems do VMs solve for us?

Before VMs, multiple applications shared one server and one OS. This caused:

- **Library conflicts** - apps needing different versions of the same library
- **Resource contention** - one app hogging CPU/memory, starving others
- **Security exposure** - one app's vulnerability exposing everything else on the machine

VMs solved all of this by giving each application its own fully isolated environment with its own OS.

## What does a VM consist of?

A VM consists of:

- **A hypervisor** - the software that creates and manages VMs
- **Virtualized hardware** - fake CPU, memory, disk, and network devices
- **Its own complete operating system** - a full kernel, system libraries, the entire stack
- **Dedicated resources** - its own allocated CPU cores, memory, and disk

Three VMs means three separate kernels running in memory, each with their own full OS stack.

## Is the VM problem just CPU?

No. CPU overhead is actually minimal - modern CPUs have dedicated virtualization instructions (Intel VT-x and AMD-V) that bring CPU overhead down to only 3-5%.

The real costs are:

- **Memory** - each VM runs a full OS kernel, consuming significant RAM
- **Boot time** - VMs take 30 seconds to over a minute to start (vs 2 seconds for containers)
- **Disk and network I/O** - the virtualization layers add overhead to every read/write operation

## What are the use cases of VMs?

- **Cloud infrastructure backbone** - VMs are still the foundation of cloud computing
- **Running untrusted code** - when you need strong isolation (e.g., AWS Lambda uses Firecracker microVMs to isolate different customers' functions)
- **Multi-tenant environments** - different customers on the same physical hardware need guaranteed separation
- **Running containers on non-Linux systems** - Docker Desktop on Mac/Windows uses a Linux VM behind the scenes to provide the kernel containers need
- **Security-critical workloads** - when container isolation isn't strong enough

## What are the pros and cons of VMs?

### Pros

- **Strong isolation** - each VM has its own kernel; an attacker who compromises one VM is stuck inside it
- **Small attack surface** - escaping requires a bug in the hypervisor, which is very hard to exploit
- **Full OS flexibility** - can run any operating system (Linux, Windows, etc.)
- **Low CPU overhead** - only 3-5% thanks to hardware virtualization (VT-x/AMD-V)
- **Battle-tested security** - proven track record for multi-tenant isolation

### Cons

- **High memory usage** - each VM runs a complete OS kernel and libraries
- **Slow boot time** - 30 seconds to over a minute to start
- **Disk and network I/O overhead** - virtualization layers add latency
- **Resource heavy** - three VMs means three full OS stacks consuming resources
- **Overkill for simple isolation** - if you just need to isolate one process, booting an entire OS is wasteful

## VM Architecture Layers

```
┌─────────────────────────────────────────────────────────────────┐
│                        VIRTUAL MACHINES                        │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │     App A        │  │     App B        │  │     App C        │ │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤ │
│  │   Libraries /    │  │   Libraries /    │  │   Libraries /    │ │
│  │   Dependencies   │  │   Dependencies   │  │   Dependencies   │ │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤ │
│  │   Guest OS       │  │   Guest OS       │  │   Guest OS       │ │
│  │   (Kernel)       │  │   (Kernel)       │  │   (Kernel)       │ │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤ │
│  │  Virtualized     │  │  Virtualized     │  │  Virtualized     │ │
│  │  Hardware        │  │  Hardware        │  │  Hardware        │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                     HYPERVISOR                                  │
│              (VMware / KVM / Hyper-V)                           │
├─────────────────────────────────────────────────────────────────┤
│                     HOST OS (Kernel)                            │
├─────────────────────────────────────────────────────────────────┤
│                  PHYSICAL HARDWARE                              │
│           (CPU with VT-x/AMD-V, RAM, Disk, NIC)                │
└─────────────────────────────────────────────────────────────────┘
```

Each VM carries its own full OS kernel and libraries — that's the source of
the memory and boot time overhead. The hypervisor sits between the VMs and
the real hardware, managing the virtualized resources for each VM.
