# Image Commands and Layers

## `docker build -t myapp .`

What you see: Docker reads your Dockerfile and produces an image.

What's actually happening:

1. Docker sends the **build context** (the `.` directory) to the Docker daemon
2. For each instruction in the Dockerfile, Docker:
   - Checks if a **cached layer** already exists for this step
   - If not, creates a temporary container, runs the instruction inside it, then snapshots the container's filesystem as a new layer
   - Removes the temporary container
3. The final stack of layers is tagged as your image (`myapp`)

```
Dockerfile instruction          What Docker does
─────────────────────────────────────────────────────────
FROM rust:1.78               →  Pull base image (its layers become your starting point)
WORKDIR /app                 →  Set metadata (no new layer)
COPY Cargo.toml Cargo.lock . →  Snapshot: new layer with dependency manifests
RUN cargo build --release    →  Start temp container, compile code, snapshot result as new layer
COPY . .                     →  Snapshot: new layer with all your source code
CMD ["./target/release/myapp"] → Store as metadata (no new layer, runs later at container start)
```

The `.` at the end is the **build context** — the directory Docker can pull files from during `COPY` and `ADD` instructions. Everything in this directory gets scanned, which is why `.dockerignore` matters for build speed.

## Layer Caching and Build Performance

In the [Image Layers](./image-layers.md) chapter we saw that each
Dockerfile instruction creates a layer — a directory of files under
`/var/lib/docker/overlay2/`. Here we focus on how to use that knowledge
to make builds fast.

### How Layer Caching Works

Docker caches every layer. When you rebuild, Docker checks each instruction top to bottom — if nothing changed for that step **and nothing changed in any step before it**, Docker reuses the cached layer and skips the work entirely.

**The moment one layer is invalidated, every layer after it is also invalidated.**

This is why **instruction order in your Dockerfile matters enormously** for build speed.

### Bad order — slow builds

```dockerfile
FROM rust:1.78
WORKDIR /app
COPY . .
RUN cargo build --release
CMD ["./target/release/myapp"]
```

| Instruction                      | Layer?        | What it does                                                                      |
| -------------------------------- | ------------- | --------------------------------------------------------------------------------- |
| `FROM rust:1.78`                 | Yes           | Pulls Debian with Rust toolchain. Becomes the base layer.                         |
| `WORKDIR /app`                   | No (metadata) | Sets working directory. Without it, files scatter across `/` mixed with OS files. |
| `COPY . .`                       | Yes           | Copies **everything** from host into `/app`. Changes every time you edit code.    |
| `RUN cargo build --release`      | Yes           | Compiles the project. Forced to redo everything, even if only code changed.       |
| `CMD ["./target/release/myapp"]` | No (metadata) | Stored in image config. Runs when container starts, not during build.             |

Cache result with bad order:

```
Layer 1: FROM rust:1.78           ✓ cached
Layer 2: COPY . .                 ✗ INVALIDATED (you changed a line of code)
Layer 3: RUN cargo build          ✗ INVALIDATED (previous layer changed → cache broken)
```

You changed one line of code and Docker has to recompile **everything** from scratch — downloading all crates and compiling all dependencies. On a Rust project, `cargo build` can take many minutes.

### Good order — fast builds

```dockerfile
FROM rust:1.78
WORKDIR /app
COPY Cargo.toml Cargo.lock .             # ← copy dependency manifests first (rarely change)
RUN mkdir src && echo "fn main(){}" > src/main.rs \
    && cargo build --release             # ← pre-build dependencies only
COPY . .                                 # ← your code (changes often, but deps are cached)
RUN cargo build --release                # ← only recompiles your code, deps already built
CMD ["./target/release/myapp"]
```

```
Layer 1: FROM rust:1.78           ✓ cached
Layer 2: WORKDIR /app              ✓ cached
Layer 3: COPY Cargo.toml ...       ✓ cached (dependencies didn't change)
Layer 4: RUN cargo build           ✓ cached (previous layer unchanged → skip!)
Layer 5: COPY . .                  ✗ INVALIDATED (code changed)
Layer 6: RUN cargo build           ✗ recompiles only your code (deps in target/ from layer 4)
```

Same result, but Docker skips the expensive dependency compilation. **Identical output, dramatically different build time.**

### The Rule

Put things that change **least frequently** at the top, and things that change **most frequently** at the bottom:

```
  ┌─────────────────────────────────┐
  │  FROM base image                │  ← almost never changes
  │  Install system dependencies    │  ← rarely changes
  │  Copy dependency manifests      │  ← changes when you add/remove crates
  │  Build dependencies             │  ← expensive, cached when manifest unchanged
  │  Copy source code               │  ← changes constantly
  │  Build application              │  ← only recompiles your code
  │  CMD / ENTRYPOINT               │  ← metadata, rarely changes
  └─────────────────────────────────┘
       Top = stable, Bottom = volatile
```

### Layer Sharing — Disk and Network Efficiency

Layers are identified by their **SHA256 hash**. If two images share the same base, those layers are stored only once on disk and transferred only once over the network.

```
Image A (Rust API):                 Image B (Rust Worker):
┌─────────────────────┐             ┌─────────────────────┐
│ Layer 5: API binary │             │ Layer 5: Worker bin │
│ Layer 4: cargo build│             │ Layer 4: cargo build│
│ Layer 3: Cargo.toml │             │ Layer 3: Cargo.toml │
├─────────────────────┤             ├─────────────────────┤
│ Layer 2: WORKDIR    │  ── same ── │ Layer 2: WORKDIR    │
│ Layer 1: rust:1.78  │  ── same ── │ Layer 1: rust:1.78  │
└─────────────────────┘             └─────────────────────┘
                    Shared on disk — stored once
```

This means:

- **Disk space** — 10 images from `rust:1.78` don't store 10 copies of the base. One copy, shared.
- **Pulling images** — `docker pull` only downloads layers you don't already have. If you already have `rust:1.78` locally, pulling a new image built on it only downloads the unique layers.
- **Pushing images** — same in reverse. The registry already has the shared layers; only your new layers are uploaded.

### .dockerignore — Keeping the Build Context Clean

When you run `docker build .`, Docker scans the entire directory as the **build context**. If your project has a large `data/` folder, build artifacts, or a `target/` directory, they all get scanned — and broad `COPY . .` instructions will pull them into the image.

A `.dockerignore` file excludes files from the build context entirely:

```
target
.git
*.log
data/
.env
```

This speeds up builds (less to scan and copy) and keeps images small (no junk included). Excluding `target/` is critical for Rust projects — it can be gigabytes in size and should be built fresh inside the image.

## `docker pull nginx`

What you see: Docker downloads an image.

What's actually happening:

1. Docker contacts the registry (Docker Hub by default)
2. Downloads the image **manifest** — a list of layer digests (SHA256 hashes)
3. For each layer, checks if it already exists locally
4. Only downloads layers you don't have — this is why pulling a second image with the same base is fast

```
$ docker pull nginx
Already exists:    a2abf6c4d29d    ←  shared base layer, already on disk
Already exists:    b1e7b4d3c5f8    ←  another shared layer
Downloading:       c9e3a2d1f4b7    ←  only this layer is new
```

## `docker images`

Lists all images stored locally. Each image is just a stack of read-only filesystem layers sitting on disk. They take up space but aren't running anything.

## `docker rmi myapp`

Removes an image. Docker deletes the layers — but only the layers not shared with other images. Shared base layers stick around until no image references them.
