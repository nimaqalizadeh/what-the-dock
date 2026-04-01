# Dockerfile Instructions

The previous chapter introduced Dockerfiles with a quick overview. Here
we go through every common instruction in detail — what it does, what
layer or metadata it produces, and the pitfalls to watch for.

## Two Kinds of Instructions

Every Dockerfile instruction falls into one of two categories:

**Build-time instructions** — execute when you run `docker build`.
They modify the filesystem and create **layers**. Once the image is
built, these instructions are done — they never run again.

```
FROM, RUN, COPY, ADD, WORKDIR
         │
         └── run during: docker build
             produce: image layers (files on disk)
             done after: build finishes
```

**Runtime metadata** — do **not** execute during build. They store
configuration in the image's JSON config file. Docker reads this
metadata later, when you run `docker run` to create a container.

```
CMD, ENTRYPOINT, ENV, EXPOSE, ARG
         │
         └── saved during: docker build (as JSON metadata)
             used during:  docker run (when creating a container)
             produce: no layers, just config
```

This distinction matters: if something should happen once and be baked
into the image (install packages, copy code), it's a build-time
instruction. If something configures how the container should behave
when it starts (what command to run, what ports to expose), it's
runtime metadata.

## `FROM` — The Base Image

```dockerfile
FROM ubuntu:22.04
```

Every Dockerfile starts with `FROM`. It sets the **base image** — the
starting filesystem that all subsequent instructions build on top of.

- `FROM ubuntu:22.04` — Ubuntu with its core packages and libraries
- `FROM rust:1.78` — Debian with the Rust toolchain pre-installed
- `FROM python:3.12-slim` — Debian slim with Python pre-installed
- `FROM scratch` — literally empty, no OS at all (for static binaries)

`FROM` creates the first layer(s). Everything your image inherits —
binaries, libraries, config files — comes from this choice.

### See it yourself

Compare what different base images give you:

```bash
# Ubuntu — has bash, apt, common utils
docker run --rm ubuntu:22.04 bash -c "which bash && which apt"
# /usr/bin/bash
# /usr/bin/apt

# Alpine — has sh, apk, very little else
docker run --rm alpine:3.19 sh -c "which sh && which apk"
# /bin/sh
# /sbin/apk

# Scratch — nothing at all (this will fail, no shell to run)
# docker run --rm scratch sh
# → exec: "sh": executable file not found
```

## `RUN` — Execute During Build

```dockerfile
RUN apt-get update && apt-get install -y curl
```

`RUN` executes a command **inside the image being built**. Docker creates
a temporary container from the layers so far, runs the command, and saves
the resulting filesystem changes as a new **read-only layer**.

This is where you:
- Install packages (`apt-get install`, `apk add`, `yum install`)
- Download dependencies (`cargo build`, `pip install`)
- Compile code (`cargo build --release`, `go build`, `make`)
- Create directories, set permissions, generate config files

**Each `RUN` creates a layer.** Files created in one `RUN` and deleted in
a later `RUN` still exist in the earlier layer (layers are immutable).
That's why you combine commands:

```dockerfile
# BAD — 3 layers, cleanup doesn't shrink earlier layers
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# GOOD — 1 layer, temp files never persist
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

### Shell form vs Exec form

```dockerfile
# Shell form — runs through /bin/sh -c
RUN apt-get update

# Exec form — runs directly, no shell
RUN ["apt-get", "update"]
```

Shell form lets you use shell features (pipes, `&&`, variable expansion).
Exec form runs the command directly without a shell. For `RUN`, shell
form is more common because you usually need those shell features.

### See it yourself

Build an image and watch the `RUN` output:

```bash
cat > /tmp/Dockerfile <<'EOF'
FROM alpine:3.19
RUN echo "I run at build time" && date > /built-at.txt
CMD ["cat", "/built-at.txt"]
EOF

docker build -t run-test -f /tmp/Dockerfile /tmp
```

You'll see "I run at build time" in the build output. Now run it
multiple times:

```bash
docker run --rm run-test
docker run --rm run-test
```

Same timestamp every time — `RUN` executed once at build, not at runtime.

## `COPY` and `ADD` — Bring Files Into the Image

```dockerfile
COPY ./src /app/src
COPY Cargo.toml /app/Cargo.toml
```

`COPY` takes files from your **build context** (the directory you passed
to `docker build`) and copies them into the image as a new layer. It's
a real, full copy — not a symlink, not a mount.

```dockerfile
ADD https://example.com/file.tar.gz /app/
ADD archive.tar.gz /app/
```

`ADD` does the same as `COPY` but with two extra features:
- It can download files from **URLs**
- It **auto-extracts** compressed archives (`.tar.gz`, `.tar.bz2`, etc.)

**Prefer `COPY` over `ADD`** unless you specifically need URL download or
auto-extraction. `COPY` is explicit — it does exactly one thing. `ADD`
has implicit behavior that can surprise you.

### See it yourself

```bash
mkdir /tmp/copytest && echo "hello" > /tmp/copytest/data.txt

cat > /tmp/copytest/Dockerfile <<'EOF'
FROM alpine:3.19
COPY data.txt /app/data.txt
CMD ["cat", "/app/data.txt"]
EOF

docker build -t copy-test /tmp/copytest
docker run --rm copy-test
# hello
```

Now delete the file on your host and run the container again:

```bash
rm /tmp/copytest/data.txt
docker run --rm copy-test
# hello  ← still there, it was copied into the image
```

## `WORKDIR` — Set the Working Directory

```dockerfile
WORKDIR /app
```

Sets the working directory for all subsequent instructions (`RUN`,
`COPY`, `CMD`, `ENTRYPOINT`). If the directory doesn't exist, Docker
creates it.

```dockerfile
WORKDIR /app
COPY . .                          # copies into /app/
RUN cargo build --release         # runs in /app/
CMD ["./target/release/myapp"]    # starts in /app/
```

Without `WORKDIR`, everything runs from `/` (the root). Using `WORKDIR`
is like `cd` but persistent — it applies to every instruction that
follows, and to the container at runtime.

**Don't use `RUN cd /app`** — the `cd` only lasts for that one `RUN`
instruction. The next instruction starts back at `/`. Use `WORKDIR`
instead.

## `CMD` — Default Runtime Command

```dockerfile
CMD ["./target/release/myapp"]
```

`CMD` sets the **default command** that runs when a container starts.
It does **not** execute during build — it's stored as metadata in the
image config.

```dockerfile
# Exec form (preferred) — runs directly, no shell wrapping
CMD ["./target/release/myapp"]

# Shell form — runs through /bin/sh -c
CMD ./target/release/myapp
```

**Exec form is preferred** because:
- The process runs as PID 1 directly and receives signals properly
  (important for graceful shutdown)
- Shell form wraps your command in `sh -c`, so `sh` is PID 1 and your
  process is a child — it won't receive `SIGTERM` from `docker stop`

`CMD` can be **overridden** at runtime. Anything you put after the image
name in `docker run` **replaces** `CMD` entirely:

```
docker run <image> [optional override]
                    │
                    └── if provided, this REPLACES the CMD
                        if omitted, Docker uses the CMD from the Dockerfile
```

Say your Dockerfile has `CMD ["./target/release/myapp"]`:

```bash
docker run myapp                # runs: ./target/release/myapp (CMD from Dockerfile)
docker run myapp bash           # runs: bash          (CMD ignored)
docker run myapp ls /app        # runs: ls /app       (CMD ignored)
docker run myapp cat /etc/hosts # runs: cat /etc/hosts(CMD ignored)
```

This is useful for debugging — you can jump into a container with
`bash` to look around, without changing the Dockerfile.

`CMD` can technically appear anywhere in the Dockerfile — it doesn't
have to be last. But it almost always is, by convention. Since `CMD`
doesn't create a layer or affect the build, putting it in the middle
would just be confusing. And if you write multiple `CMD` instructions,
only the **last one** wins — earlier ones are silently ignored:

```dockerfile
CMD ["echo", "first"]
RUN echo "installing something"
CMD ["echo", "second"]
# only "second" runs — the first CMD is overwritten
```

### See it yourself

```bash
cat > /tmp/Dockerfile <<'EOF'
FROM alpine:3.19
CMD ["echo", "default command"]
EOF

docker build -t cmd-test -f /tmp/Dockerfile /tmp

docker run --rm cmd-test
# default command

docker run --rm cmd-test echo "overridden"
# overridden
```

## `ENTRYPOINT` — The Fixed Command

```dockerfile
ENTRYPOINT ["./target/release/myapp"]
CMD ["--port", "8080"]
```

`ENTRYPOINT` sets a command that **always runs** — unlike `CMD`, it
doesn't get replaced when you pass arguments to `docker run`. `CMD`
then provides **default arguments** to the entrypoint.

`ENTRYPOINT` *can* be overridden, but you have to be explicit about it:

```bash
docker run --entrypoint /bin/sh myapp
```

This replaces the entrypoint entirely. Without `--entrypoint`, the
entrypoint always runs — that's the key difference from `CMD`.

```
  ENTRYPOINT = the verb (always runs)
  CMD        = the default noun (can be overridden)
```

With the Dockerfile above:

```bash
docker run myapp                    # runs: ./target/release/myapp --port 8080
docker run myapp --port 3000        # runs: ./target/release/myapp --port 3000 (CMD overridden)
```

This pattern is useful when your image is a **tool** — the entrypoint
is the tool itself, and the user provides arguments:

```dockerfile
# curl image — always runs curl, user provides the URL
ENTRYPOINT ["curl", "-s"]
CMD ["https://example.com"]
```

```bash
docker run mycurl                        # curl -s https://example.com
docker run mycurl https://google.com     # curl -s https://google.com
```

## `ENV` — Environment Variables

```dockerfile
ENV RUST_LOG=info
ENV DB_HOST=localhost DB_PORT=5432
```

Sets environment variables that are available both during **build**
(in subsequent `RUN` instructions) and at **runtime** (inside the
container).

```dockerfile
ENV APP_HOME=/app
WORKDIR $APP_HOME      # uses the variable
COPY . $APP_HOME       # uses the variable
```

Environment variables can be **overridden** at runtime:

```bash
docker run -e RUST_LOG=debug myapp
```

### See it yourself

```bash
cat > /tmp/Dockerfile <<'EOF'
FROM alpine:3.19
ENV GREETING="hello from the image"
CMD ["sh", "-c", "echo $GREETING"]
EOF

docker build -t env-test -f /tmp/Dockerfile /tmp

docker run --rm env-test
# hello from the image

docker run --rm -e GREETING="overridden" env-test
# overridden
```

## `EXPOSE` — Document a Port

```dockerfile
EXPOSE 3000
```

`EXPOSE` does **not** actually publish a port. It's documentation — it
tells anyone reading the Dockerfile (and tools like `docker inspect`)
that the app listens on port 3000. You still need `-p` at runtime to
actually map the port:

```bash
docker run -p 3000:3000 myapp
```

Think of `EXPOSE` as a comment that tooling can read.

## `ARG` — Build-Time Variables

```dockerfile
ARG RUST_VERSION=1.78
FROM rust:${RUST_VERSION}
```

`ARG` defines a variable that's only available **during build** — not
at runtime. Use it to parameterize your Dockerfile:

```bash
docker build --build-arg RUST_VERSION=1.77 -t myapp .
docker build --build-arg RUST_VERSION=1.78 -t myapp .
```

Key difference from `ENV`:
- `ARG` — available during build only, gone at runtime
- `ENV` — available during build and at runtime

## Quick Reference

```
Instruction  Creates layer?  When it runs     Purpose
───────────  ──────────────  ──────────────   ──────────────────────
FROM         yes             build            set base image
RUN          yes             build            execute a command
COPY         yes             build            copy files from host
ADD          yes             build            copy + extract/download
WORKDIR      no (metadata)   build            set working directory
CMD          no (metadata)   runtime          default command
ENTRYPOINT   no (metadata)   runtime          fixed command
ENV          no (metadata)   build + runtime  set env variables
EXPOSE       no (metadata)   -                document a port
ARG          no (metadata)   build only       build-time variable
```
