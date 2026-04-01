# Image Layers and the Filesystem

Now that we know a Dockerfile produces an image — let's look at how that
image is structured internally.

Each Dockerfile instruction that modifies files creates a **layer**. These
layers stack up to form what the container sees as `/`.

```
  Dockerfile                          Container's /
  ┌──────────────────────────┐        ┌─────────────────────────┐
  │ FROM rust:1.78           │───────>│ Layer 1: base OS + Rust │
  │ COPY . /app              │───────>│ Layer 2: source code    │
  │ RUN cargo build --release│───────>│ Layer 3: compiled binary│
  │ CMD ["./target/release/  │        │ (metadata only)         │
  │       myapp"]            │        │                         │
  └──────────────────────────┘        └─────────────────────────┘
```

`CMD` doesn't create a real filesystem layer — it only stores metadata.
Only instructions that modify files (`FROM`, `COPY`, `RUN`, `ADD`) create
real layers.

## What is a Layer?

A layer is **not a filesystem itself** — it's a **diff** (a set of changes).

Think of it like git commits:

- A git commit doesn't contain the whole codebase — it only contains
  **what changed**
- But when you check out a branch, git stacks all commits together and
  you see the full codebase

Layers work the same way:

- Each layer only stores **files that were added, modified, or deleted**
  by that instruction
- When the container runs, Docker stacks all layers together and the
  container sees **one complete filesystem**

```
What the container sees at /:        Built from:

/bin/bash                            <-- Layer 1 (FROM rust)
/etc/hosts                           <-- Layer 1 (FROM rust)
/usr/local/cargo/bin/rustc           <-- Layer 1 (FROM rust)
/app/src/main.rs                     <-- Layer 2 (COPY . /app)
/app/target/release/myapp            <-- Layer 3 (RUN cargo build)
```

So:

- **Layer** = a diff, a set of changes ("these files were added/changed")
- **Filesystem** = what the container sees after all layers are merged

The filesystem is the **result** of stacking all the layers. The layers
are the **building blocks** that produce it.

## What Each Instruction Actually Does on Disk

Each layer is a **real directory** on the host under
`/var/lib/docker/overlay2/`. Let's trace what happens for each
instruction in this Dockerfile:

```dockerfile
FROM rust:1.78
COPY . /app
RUN cd /app && cargo build --release
CMD ["/app/target/release/myapp"]
```

### `FROM rust:1.78`

This pulls the Rust base image from Docker Hub. The `rust` image is
built on top of Debian and contains everything needed to compile Rust
code — the filesystem structure, core binaries, libraries, and the
Rust toolchain:

```
/var/lib/docker/overlay2/<hash-A>/
  bin/
    bash          ← the Bash shell binary
    ls            ← the ls command
    cat           ← the cat command
  etc/
    hosts         ← network hostname config
    apt/          ← package manager config
    passwd        ← user accounts
  usr/
    lib/          ← shared libraries (.so files)
    bin/          ← more executables (grep, find, ...)
    local/
      cargo/
        bin/
          rustc   ← the Rust compiler
          cargo   ← the Rust package manager
          rustup  ← the Rust toolchain manager
      rustup/     ← toolchain files
  var/
    log/          ← log directory structure
```

Notice what's **not** here: there is no kernel. No `/boot/vmlinuz`, no
kernel modules, no kernel at all. The Rust image only contains
**userspace** files — the binaries, config files, and libraries that
the base OS and Rust toolchain provide.

When `/bin/bash` inside the container calls `read()` or `write()`, that
syscall goes directly to the **host's kernel**. The host kernel handles
all system calls for every container. The image just provides the
programs and libraries that _make_ those syscalls — the kernel they talk
to is always the host's.

This is what it takes to have a working Rust build environment — not a
running OS, just its **filesystem**. With these files exposed through
OverlayFS, a process can run as if it's on a Debian machine with Rust
installed — but the kernel doing all the real work is the host's.

The Rust image itself is made of multiple layers (one for each step
in its Dockerfile), but from our perspective it arrives as a
ready-to-use base. There's no single "Rust image" — Docker Hub has
variants like `rust:1.78` (full Debian), `rust:1.78-slim` (smaller
Debian), and `rust:1.78-alpine` (Alpine-based), each built from
different Dockerfiles with different packages and layers.

### `COPY . /app`

This takes your project directory from your host machine and **copies**
it — a real, full copy, not a symlink — into a new layer directory:

```
/var/lib/docker/overlay2/<hash-B>/
  app/
    src/
      main.rs       ← copied from your host's ./src/main.rs
    Cargo.toml      ← copied from your host's ./Cargo.toml
    Cargo.lock      ← copied from your host's ./Cargo.lock
```

After this step, OverlayFS stacks hash-B on top of hash-A. A process
looking at this image now sees both the Rust toolchain files _and_
`/app/src/main.rs` as if they were on one filesystem.

The copy is self-contained. If you delete your project on your host, the
image still has these files. The image depends on nothing outside
itself.

### `RUN cd /app && cargo build --release`

This is where it gets interesting. `RUN` does **not** run on your host.
Docker:

1. Creates a **temporary container** from the layers built so far
   (hash-A + hash-B stacked together)
2. Starts a process inside that container that executes `cargo build`
3. That process runs inside the image's filesystem — it reads
   `/app/Cargo.toml` from hash-B, downloads crate dependencies from
   **crates.io** over the network, and compiles everything
4. The compiled binary and downloaded dependencies are written to the
   container's writable layer
5. Docker saves that writable layer as a new **read-only layer**

```
/var/lib/docker/overlay2/<hash-C>/
  app/
    target/
      release/
        myapp       ← compiled binary, built inside the image
        deps/       ← compiled dependencies from crates.io
```

Your host's `target/` directory (if you even have one) is never involved.
The `RUN` instruction executes entirely within the image's world.

### `CMD ["/app/target/release/myapp"]`

This creates **no layer directory at all**. It stores a single piece
of metadata in the image's JSON config file:

```json
{
  "Cmd": ["/app/target/release/myapp"]
}
```

When you later run `docker run myapp`, Docker reads this config to
know what process to start. You can see it yourself with
`docker inspect myapp`.

## How Layers Become One Filesystem — OverlayFS

When a container runs, Docker doesn't merge these directories into one.
It uses **OverlayFS** — a kernel filesystem that presents multiple
directories as one unified view:

```
Container sees:          Actually stacked from:

/                        ┌─ Writable layer (container changes)
├── app/                 ├─ hash-C: compiled binary + deps
│   ├── src/main.rs      ├─ hash-B: source code
│   ├── Cargo.toml       └─ hash-A: rust toolchain + OS base
│   └── target/release/
│       └── myapp
├── bin/bash
└── usr/local/cargo/bin/rustc
```

The kernel handles this stacking — the container sees one flat
filesystem, but each layer remains a separate directory on disk. That's
what makes sharing and caching possible.

## Layer Hashing and Cache Invalidation

Each layer is identified by a **SHA256 hash** of its contents. Docker
uses this hash to decide whether it can reuse a cached layer or needs
to rebuild.

- If the inputs to an instruction haven't changed, the hash is the
  same → **cache hit**, layer is reused
- If something changes (different files for `COPY`, different command
  for `RUN`), the hash changes → **cache miss**, layer must rebuild
- **Cascading invalidation:** if layer N is invalidated, all layers
  after it (N+1, N+2, ...) must also rebuild — because they were
  built on top of layer N's state

This cascading rule is why Dockerfile instruction order matters
enormously for build speed. For practical examples of good vs bad
ordering, see the [Image Commands](./image-commands.md) chapter.

## Why Layers Instead of One Big Blob?

- **Caching** — if you only changed your source code, Docker reuses the
  base and dependency layers from cache and only rebuilds the final
  compile step. This can save minutes on every build.
- **Sharing** — ten images based on `rust:1.78` share that same base
  layer on disk, stored only once.
- **Fast transfers** — when pushing/pulling images, only changed layers
  are transferred.

## See it yourself

Inspect the layers of any image:

```bash
docker history nginx
```

Output shows each layer, its size, and the command that created it. You
can see exactly which Dockerfile instruction produced which layer.

To see layers in more detail:

```bash
docker inspect nginx | jq '.[0].RootFS.Layers'
```

Each entry is a SHA256 hash of a layer. If you inspect two images that
share the same base (e.g., both `FROM rust:1.78`), you'll see the
same layer hashes at the bottom — proving they share layers on disk.

Watch caching in action by building an image twice:

```bash
mkdir /tmp/cachetest && cd /tmp/cachetest

cat > Dockerfile <<'EOF'
FROM ubuntu:22.04
RUN apt-get update
COPY . /app
EOF

docker build -t test1 .
docker build -t test2 .
```

The second build is nearly instant — every layer comes from cache.

Verify that both images share the exact same layers — not copies, the
same ones on disk:

```bash
docker inspect test1 --format '{{json .RootFS.Layers}}' | jq .
docker inspect test2 --format '{{json .RootFS.Layers}}' | jq .
```

Both output the same list of SHA256 hashes. Same hashes = same layer
directories under `/var/lib/docker/overlay2/`.

Now change only the `COPY` line's content and rebuild: Docker reuses the
cached `FROM` and `RUN` layers and only rebuilds from the changed layer
onward.

## Where Layers Live on Disk

Docker doesn't use a traditional database. It uses the **filesystem
itself** as its storage — everything lives under `/var/lib/docker/` in
a predictable directory structure.

### The layer content: `/var/lib/docker/overlay2/`

Each layer is a directory here. The directory name is a hash derived
from the layer's content:

```
/var/lib/docker/overlay2/
├── a3f2b1c9d4e7.../                ← Layer A (rust base)
│   ├── diff/                        ← the actual files
│   │   ├── bin/bash
│   │   ├── etc/hosts
│   │   └── usr/local/cargo/bin/rustc
│   ├── link                         ← shortened ID for OverlayFS
│   ├── lower                        ← pointer to parent layers
│   └── work/                        ← OverlayFS internal scratch space
│
├── b7c1e4a8f2d3.../                ← Layer B (source code)
│   ├── diff/
│   │   └── app/
│   │       ├── src/main.rs
│   │       ├── Cargo.toml
│   │       └── Cargo.lock
│   ├── link
│   ├── lower                        ← points to Layer A
│   └── work/
│
├── e9d4f7b2c1a6.../                ← Layer C (compiled binary)
│   ├── diff/
│   │   └── app/
│   │       └── target/release/myapp
│   ├── link
│   ├── lower                        ← points to Layer A + B
│   └── work/
│
└── l/                               ← shortened symlinks
    ├── ABC123 -> ../a3f2b1c9d4e7.../diff
    ├── DEF456 -> ../b7c1e4a8f2d3.../diff
    └── GHI789 -> ../e9d4f7b2c1a6.../diff
```

Inside each layer directory:

- **`diff/`** — the actual files this layer added or changed. This is
  the content that OverlayFS exposes to the container.
- **`link`** — a short identifier for this layer (OverlayFS has a limit
  on mount option length, so it uses these short IDs instead of the
  full hash).
- **`lower`** — tells OverlayFS which layers sit below this one. It's a
  chain: Layer C's `lower` points to B and A, so OverlayFS knows the
  stacking order.
- **`work/`** — scratch space OverlayFS uses internally for atomic
  operations like copy-on-write.

### The image metadata: `/var/lib/docker/image/overlay2/`

This is where Docker stores the JSON files that map hashes to layers
and describe each image:

```
/var/lib/docker/image/overlay2/
├── imagedb/
│   └── content/sha256/
│       └── <image-hash>             ← JSON config for each image
│                                      (CMD, ENV, layer list, etc.)
├── layerdb/
│   └── sha256/
│       └── <layer-hash>/
│           ├── cache-id             ← maps this hash to the directory
│           │                          name in /var/lib/docker/overlay2/
│           ├── diff                 ← this layer's content hash
│           ├── parent               ← hash of the layer below
│           └── size                 ← layer size in bytes
└── repositories.json                ← maps image names/tags to hashes
                                       (e.g., "myapp:latest" → sha256:...)
```

The flow when Docker runs an image:

1. You say `docker run myapp`
2. Docker reads `repositories.json` to find the hash for `myapp:latest`
3. It reads the image config from `imagedb/` to get the ordered list of
   layer hashes and the `CMD` to run
4. For each layer hash, it reads `layerdb/` to find the `cache-id` —
   which maps to the actual directory in `overlay2/`
5. It tells OverlayFS to stack those directories, adds a writable layer,
   sets up namespaces and cgroups, and starts the process

Everything is flat files on disk — JSON configs pointing to directories
pointing to other directories. No database engine, no daemon state. This
is why you can break Docker by manually deleting things in
`/var/lib/docker/` — there's no database to recover from, just files
referencing other files.
