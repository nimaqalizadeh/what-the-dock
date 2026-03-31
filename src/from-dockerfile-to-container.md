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
  │  Layer 3: app code │      │  Writable layer     │  ← changes go here
  │  Layer 2: packages │      │  Layer 3: app code  │
  │  Layer 1: base OS  │      │  Layer 2: packages  │
  └────────────────────┘      │  Layer 1: base OS   │
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
