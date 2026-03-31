# Image Layers and the Filesystem

Now that we know a Dockerfile produces an image — let's look at how that
image is structured internally.

Each Dockerfile instruction that modifies files creates a **layer**. These
layers stack up to form what the container sees as `/`.

```
  Dockerfile                          Container's /
  ┌──────────────────────────┐        ┌─────────────────────────┐
  │ FROM ubuntu:22.04        │───────>│ Layer 1: base OS        │
  │ COPY ./app /app          │───────>│ Layer 2: app files      │
  │ RUN npm install          │───────>│ Layer 3: dependencies   │
  │ CMD ["node", "app.js"]   │        │ (metadata only)         │
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

/bin/bash                            <-- Layer 1 (FROM ubuntu)
/etc/hosts                           <-- Layer 1 (FROM ubuntu)
/usr/bin/node                        <-- Layer 1 (FROM ubuntu)
/app/app.js                          <-- Layer 2 (COPY ./app)
/app/node_modules/express/           <-- Layer 3 (RUN npm install)
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
FROM ubuntu:22.04
COPY ./app /app
RUN npm install
CMD ["node", "app.js"]
```

### `FROM ubuntu:22.04`

This pulls the Ubuntu base image from Docker Hub. An Ubuntu image
contains everything that makes Ubuntu _Ubuntu_ — its filesystem
structure, its core binaries, its libraries:

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
  var/
    log/          ← log directory structure
```

Notice what's **not** here: there is no kernel. No `/boot/vmlinuz`, no
kernel modules, no kernel at all. The Ubuntu image only contains
**userspace** files — the binaries, config files, and libraries that
Ubuntu packages provide.

When `/bin/bash` inside the container calls `read()` or `write()`, that
syscall goes directly to the **host's kernel**. The host kernel handles
all system calls for every container. The Ubuntu image just provides the
programs and libraries that _make_ those syscalls — the kernel they talk
to is always the host's.

This is what it takes to have a working Ubuntu environment — not a
running OS, just its **filesystem**. With these files exposed through
OverlayFS, a process can run as if it's on an Ubuntu machine — but
the kernel doing all the real work is the host's.

The Ubuntu image itself is made of multiple layers (one for each step
in _Ubuntu's_ Dockerfile), but from our perspective it arrives as a
ready-to-use base. There's no single "Ubuntu image" — Docker Hub has
variants like `ubuntu:22.04` (standard) and `ubuntu:22.04-minimal`
(stripped down), each built from different Dockerfiles with different
packages and layers.

### `COPY ./app /app`

This takes the `./app` directory from your host machine and **copies**
it — a real, full copy, not a symlink — into a new layer directory:

```
/var/lib/docker/overlay2/<hash-B>/
  app/
    app.js          ← copied from your host's ./app/app.js
    package.json    ← copied from your host's ./app/package.json
```

After this step, OverlayFS stacks hash-B on top of hash-A. A process
looking at this image now sees both Ubuntu's files _and_ `/app/app.js`
as if they were on one filesystem.

The copy is self-contained. If you delete `./app` on your host, the
image still has these files. The image depends on nothing outside
itself.

### `RUN npm install`

This is where it gets interesting. `RUN` does **not** run on your host.
Docker:

1. Creates a **temporary container** from the layers built so far
   (hash-A + hash-B stacked together)
2. Starts a process inside that container that executes `npm install`
3. That process runs inside the image's filesystem — it reads
   `/app/package.json` from hash-B, downloads packages from the
   **npm registry** over the network
4. The downloaded `node_modules` are written to the container's
   writable layer
5. Docker saves that writable layer as a new **read-only layer**

```
/var/lib/docker/overlay2/<hash-C>/
  app/
    node_modules/
      express/      ← downloaded from npm, NOT from your host
      lodash/       ← downloaded from npm, NOT from your host
```

Your host's `node_modules` (if you even have one) is never involved.
The `RUN` instruction executes entirely within the image's world.

### `CMD ["node", "app.js"]`

This creates **no layer directory at all**. It stores a single piece
of metadata in the image's JSON config file:

```json
{
  "Cmd": ["node", "app.js"]
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
├── app/                 ├─ hash-C: node_modules
│   ├── app.js           ├─ hash-B: app code
│   ├── node_modules/    └─ hash-A: ubuntu base
│   └── package.json
├── bin/bash
└── etc/hosts
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

This is why Dockerfile order matters:

```dockerfile
# GOOD — rarely-changing layers first
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nodejs npm   # changes rarely
COPY package.json /app/package.json                    # changes sometimes
RUN npm install                                        # rebuilds only when
                                                       # package.json changes
COPY ./app /app                                        # changes often (last)

# BAD — frequently-changing layer early
FROM ubuntu:22.04
COPY . /app                    # any code change invalidates THIS layer
RUN apt-get update             # ...and forces this to rebuild too
RUN npm install                # ...and this
```

Put things that change **rarely** at the top and things that change
**often** at the bottom. That way, a code change only rebuilds the
last few layers instead of the entire image.

## Why Layers Instead of One Big Blob?

- **Caching** — if you only changed your app code, Docker reuses layers
  1 and 3 from cache and only rebuilds layer 2. This can save minutes
  on every build.
- **Sharing** — ten images based on `ubuntu:22.04` share that same base
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
share the same base (e.g., both `FROM ubuntu:22.04`), you'll see the
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
├── a3f2b1c9d4e7.../                ← Layer A (ubuntu base)
│   ├── diff/                        ← the actual files
│   │   ├── bin/bash
│   │   ├── etc/hosts
│   │   └── usr/lib/...
│   ├── link                         ← shortened ID for OverlayFS
│   ├── lower                        ← pointer to parent layers
│   └── work/                        ← OverlayFS internal scratch space
│
├── b7c1e4a8f2d3.../                ← Layer B (app code)
│   ├── diff/
│   │   └── app/
│   │       ├── app.js
│   │       └── package.json
│   ├── link
│   ├── lower                        ← points to Layer A
│   └── work/
│
├── e9d4f7b2c1a6.../                ← Layer C (node_modules)
│   ├── diff/
│   │   └── app/
│   │       └── node_modules/...
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
