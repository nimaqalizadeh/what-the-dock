# Understanding Docker Commands

Now that we understand what containers actually are (isolated processes, not VMs), let's look at what each Docker command is really doing under the hood.

Every Docker command maps to something we covered earlier — namespaces, cgroups, layers, or networking. There's no magic, just Linux kernel features wrapped in a CLI.

## Summary: What Each Command Actually Does

```
Command                     Kernel / system action
──────────────────────────────────────────────────────────────────
docker build             →  Execute Dockerfile steps, snapshot layers
docker run               →  Create namespaces + cgroups + writable layer, start process
docker exec              →  setns() into existing container's namespaces
docker stop              →  Send SIGTERM/SIGKILL to PID 1
docker rm                →  Delete writable layer
docker run -p 8080:3000  →  Add iptables rule forwarding host:8080 → container:3000
docker run -v data:/app  →  Mount host directory into container's mount namespace
docker network create    →  Create a virtual bridge + enable DNS resolution
docker compose up        →  Run all of the above based on a YAML file
```

The following chapters break down each category in detail.
