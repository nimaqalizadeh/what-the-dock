# From Dockerfile to Container

We've seen that a container is just a process wrapped in namespaces and
cgroups. But where does that process's **filesystem** come from? How does
the container know what files to see, what runtime to use, what command
to run?

The answer is a three-step pipeline:

```
  Dockerfile  ──build──>  Image  ──run──>  Container

  (recipe)               (snapshot)        (running process)
```

## What is a Dockerfile?

A Dockerfile is a **text file with build instructions** — a recipe that
tells Docker how to assemble an image, step by step.

```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nodejs npm
COPY ./app /app
WORKDIR /app
RUN npm install
CMD ["node", "app.js"]
```

Each line is an instruction:

- **`FROM`** — start from an existing base image (here, Ubuntu 22.04).
  Every Dockerfile starts with `FROM`. You're not building from scratch —
  you're building on top of something.
- **`RUN`** — execute a command inside the image during build time. This
  is where you install dependencies, compile code, or set up the
  environment. The result is saved as a new layer.
- **`COPY`** — copy files from your local machine into the image.
- **`WORKDIR`** — set the working directory for subsequent instructions
  (like `cd` but persistent).
- **`CMD`** — the default command to run when a container starts from
  this image. This doesn't execute during build — it's stored as
  metadata for runtime.

The Dockerfile is **not a script that runs when the container starts**.
It runs once, at **build time**, to produce an image. After that, the
Dockerfile's job is done.

### See it yourself

Create a simple Dockerfile and build it:

```bash
mkdir /tmp/myapp && cd /tmp/myapp

cat > Dockerfile <<'EOF'
FROM ubuntu:22.04
RUN echo "built at $(date)" > /build-time.txt
CMD ["cat", "/build-time.txt"]
EOF

docker build -t myapp .
```

The output shows Docker executing each instruction in order. Now run it:

```bash
docker run --rm myapp
# built at Mon Mar 31 12:00:00 UTC 2026
```

Run it again a minute later — same timestamp. The `RUN` command executed
once during build. The image captured that moment. Every container from
this image sees the same file.

## What is a Docker Image?

An image is the **output of the build** — a read-only, self-contained
snapshot that has everything the application needs to run:

- The base OS filesystem (from `FROM`)
- Installed packages and dependencies (from `RUN`)
- Your application code (from `COPY`)
- Metadata like the default command (from `CMD`) and environment
  variables (from `ENV`)

An image is **immutable** — once built, it never changes. You don't edit
an image. If you need to change something, you update the Dockerfile and
rebuild.

An image is **not one big file**. It's a **manifest** — a blueprint that
tells the Docker engine:

1. **Which layers to stack**, and in what order (pointers to the layer
   directories under `/var/lib/docker/overlay2/`)
2. **How to configure the container** — the default command (`CMD`),
   environment variables (`ENV`), working directory (`WORKDIR`), exposed
   ports, and other metadata

```
  Image = manifest
  ┌──────────────────────────────────────┐
  │  Layers (in order):                  │
  │    1. sha256:a3f2...  → overlay2/... │  ← Ubuntu base files
  │    2. sha256:b7c1...  → overlay2/... │  ← app code
  │    3. sha256:e9d4...  → overlay2/... │  ← node_modules
  │                                      │
  │  Config:                             │
  │    CMD: ["node", "app.js"]           │
  │    WORKDIR: /app                     │
  │    ENV: NODE_ENV=production          │
  └──────────────────────────────────────┘
```

When you run `docker run myapp`, Docker reads this manifest, stacks the
layer directories with OverlayFS into one filesystem view, wraps the
process in namespaces and cgroups, and starts the command from `CMD`.
The image is the recipe — the container is the result of following it.

### See it yourself

List your images:

```bash
docker images
```

Inspect what's inside one:

```bash
docker inspect myapp
```

This shows the image's metadata: the command it will run, environment
variables, the layers that make it up. The image is a complete,
self-contained package sitting on disk, waiting to be run.

## What is a Container (again)?

A container is what happens when you **run** an image. Docker takes the
image's filesystem, adds a thin **writable layer** on top, sets up the
namespaces and cgroups we discussed in the previous chapter, and starts
the process defined by `CMD`.

```
  Image (read-only)           Container (running)
  ┌────────────────────┐      ┌────────────────────┐
  │  Layer 3: app code │      │  Writable layer    │  ← changes go here
  │  Layer 2: packages │      │  Layer 3: app code │
  │  Layer 1: base OS  │      │  Layer 2: packages │
  └────────────────────┘      │  Layer 1: base OS  │
                              └────────────────────┘
                              + PID namespace
                              + Mount namespace
                              + Network namespace
                              + Cgroups
```

Any files the container creates or modifies go into the writable layer.
The image layers below remain untouched. When the container is removed,
the writable layer is deleted — the image stays clean for the next
container.

### Multiple containers, one image — no copying

When you run 10 containers from the same image, Docker does **not** copy
the layers 10 times. The image layers sit on disk **once**, read-only
and shared. Docker only creates one new writable layer per container:

```
Image layers (on disk ONCE, shared, read-only):
  /var/lib/docker/overlay2/hash-A/    ← ubuntu base
  /var/lib/docker/overlay2/hash-B/    ← app code
  /var/lib/docker/overlay2/hash-C/    ← node_modules

Container 1:                          Container 2:
┌──────────────────────┐              ┌──────────────────────┐
│ writable-layer-001   │ ← new       │ writable-layer-002   │ ← new
│ (empty at start)     │              │ (empty at start)     │
├──────────────────────┤              ├──────────────────────┤
│ hash-C               │ ← shared    │ hash-C               │ ← same dir
│ hash-B               │ ← shared    │ hash-B               │ ← same dir
│ hash-A               │ ← shared    │ hash-A               │ ← same dir
└──────────────────────┘              └──────────────────────┘
```

Both containers see `/bin/bash` — but it's the **same file on disk**,
served through OverlayFS. When container 1 writes a new file, it goes
into `writable-layer-001`. Container 2 never sees it — it has its own
`writable-layer-002`.

What if a container **modifies** an existing file, say `/etc/hosts`?
OverlayFS does a **copy-on-write**: it copies that one file from the
read-only layer into the container's writable layer, then applies the
change there. The original in `hash-A` stays untouched. Every other
container still sees the original.

This is why `docker run` takes seconds — Docker doesn't build or copy
anything. It tells OverlayFS "stack these existing directories, add a
fresh writable layer on top," sets up namespaces and cgroups, and starts
the process. Almost nothing to create.

### See it yourself

Run a container, write a file, then check:

```bash
docker run -d --name testbox ubuntu:22.04 sleep 3600

# Create a file inside the container
docker exec testbox bash -c "echo 'hello' > /tmp/myfile.txt"
docker exec testbox cat /tmp/myfile.txt
# hello

# Remove the container
docker rm -f testbox

# Start a new one from the same image
docker run --rm ubuntu:22.04 cat /tmp/myfile.txt
# cat: /tmp/myfile.txt: No such file or directory
```

The file lived in the container's writable layer. The image was never
modified. New container, clean slate.

## The Full Pipeline

```
  You write          Docker builds        Docker runs
  ──────────         ────────────         ───────────

  Dockerfile    ──>  Image           ──>  Container
  (instructions)     (read-only           (process with
                      snapshot with         writable layer,
                      all layers)           namespaces,
                                            cgroups)

  Runs once          Stored on disk       Lives in memory
  at build time      (immutable)          (temporary)
```

This is why:

- Changing code means **rebuilding the image** (not modifying a running
  container)
- Multiple containers from the same image are identical at start
- Removing a container loses its changes — the image is the source of
  truth
