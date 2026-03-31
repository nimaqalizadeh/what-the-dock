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
FROM node:20-alpine          →  Pull base image (its layers become your starting point)
WORKDIR /app                 →  Set metadata (no new layer)
COPY package.json .          →  Snapshot: new layer with just package.json
RUN npm install              →  Start temp container, run npm install, snapshot result as new layer
COPY . .                     →  Snapshot: new layer with all your source code
CMD ["node", "server.js"]    →  Store as metadata (no new layer, runs later at container start)
```

The `.` at the end is the **build context** — the directory Docker can pull files from during `COPY` and `ADD` instructions. Everything in this directory gets scanned, which is why `.dockerignore` matters for build speed.

## Image Layers and Build Performance

A Docker image is not one big blob — it's a **stack of layers**. A blob (Binary Large Object) means a single, undivided chunk of data with no internal structure. Instead of being stored as one monolithic file, an image is split into layers — which is what enables caching, sharing, and efficient transfers. Each Dockerfile instruction that modifies files (`FROM`, `COPY`, `RUN`, `ADD`) creates a new layer. Other instructions like `CMD` and `ENV` only add metadata.

Each layer is a **diff** — it only stores files that were added, modified, or deleted by that instruction. Think of it like git commits: a commit doesn't contain the whole codebase, just what changed. Docker stacks all layers together so the container sees one complete filesystem.

**Where layers live on disk:**

```
/var/lib/docker/
├── overlay2/                          ← actual layer contents live here
│   ├── abc123.../diff/                ← Layer 1 (FROM): /bin, /etc, /usr...
│   ├── def456.../diff/                ← Layer 2 (COPY): /app/package.json
│   ├── ghi789.../diff/                ← Layer 3 (RUN):  /app/node_modules/...
│   ├── jkl012.../diff/                ← Layer 4 (COPY): /app/server.js, /app/src/...
│   └── merged/                        ← union of all layers — what the container sees as /
├── image/
│   └── overlay2/
│       ├── imagedb/                   ← image configs (JSON: layer order, CMD, ENV, EXPOSE)
│       └── layerdb/                   ← layer metadata (maps layer IDs to content SHA-256 hashes)
└── containers/                        ← per-container: writable layer, logs, config
```

Each layer's `diff/` directory contains **only the files that instruction added or changed** — not a full copy of the filesystem. Docker uses **OverlayFS** (a union filesystem) to stack all these `diff/` directories on top of each other, so the container sees one merged `/` as its root filesystem. The `merged/` directory is that combined view.

### How Layer Caching Works

Docker caches every layer. When you rebuild, Docker checks each instruction top to bottom — if nothing changed for that step **and nothing changed in any step before it**, Docker reuses the cached layer and skips the work entirely.

**The moment one layer is invalidated, every layer after it is also invalidated.**

This is why **instruction order in your Dockerfile matters enormously** for build speed.

### Bad order — slow builds

```
Dockerfile instruction              Layer on disk                              What it contains
───────────────────────────────────────────────────────────────────────────────────────────────────────
FROM node:20-alpine              →  overlay2/abc123.../diff/                →  /bin, /etc, /usr, /usr/bin/node...
                                    Pulls Alpine Linux (~5MB) with Node.js 20 pre-installed.
                                    Without this, you'd install the OS and Node.js manually.

WORKDIR /app                     →  (no layer — metadata only)              →  Sets working directory to /app
                                    Creates /app if it doesn't exist. All subsequent commands run from /app.
                                    Without this, files scatter across / (root):
                                      With WORKDIR /app:      Without WORKDIR:
                                      /app/package.json       /package.json
                                      /app/server.js          /server.js
                                      /app/node_modules/      /node_modules/    (mixed with OS files)

COPY . .                         →  overlay2/def456.../diff/                →  /app/server.js, /app/src/...
                                    Copies EVERYTHING from host build context (first .) to container's
                                    WORKDIR (second . = /app). Code changes every time → layer invalidated.

RUN npm install                  →  overlay2/ghi789.../diff/                →  /app/node_modules/...
                                    Forced to reinstall every time, even if dependencies didn't change.

CMD ["node", "server.js"]        →  (no layer — metadata only)              →  Stored in image JSON config
                                    Not run during build. Executed when container starts as PID 1.
```

Cache result with bad order:
```
Layer 1: FROM node:20-alpine     ✓ cached
Layer 2: COPY . .                 ✗ INVALIDATED (you changed a line of code)
Layer 3: RUN npm install          ✗ INVALIDATED (previous layer changed → cache broken)
```

You changed one line of code and Docker has to reinstall **all** your dependencies. On a large project, `npm install` can take minutes.

### Good order — fast builds

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json .         # ← copy dependency file first (rarely changes)
RUN npm install             # ← only reruns when dependencies actually change
COPY . .                    # ← your code (changes often, but npm install is already cached)
CMD ["node", "server.js"]
```

```
Layer 1: FROM node:20-alpine     ✓ cached
Layer 2: WORKDIR /app             ✓ cached
Layer 3: COPY package.json .      ✓ cached (dependencies didn't change)
Layer 4: RUN npm install          ✓ cached (previous layer unchanged → skip!)
Layer 5: COPY . .                 ✗ INVALIDATED (code changed)
```

Same result, but Docker skips the expensive `npm install` step. **Identical output, dramatically different build time.**

### The Rule

Put things that change **least frequently** at the top, and things that change **most frequently** at the bottom:

```
  ┌─────────────────────────────────┐
  │  FROM base image                │  ← almost never changes
  │  Install system dependencies    │  ← rarely changes
  │  Copy dependency manifests      │  ← changes when you add/remove packages
  │  Install app dependencies       │  ← expensive, cached when manifest unchanged
  │  Copy source code               │  ← changes constantly
  │  CMD / ENTRYPOINT               │  ← metadata, rarely changes
  └─────────────────────────────────┘
       Top = stable, Bottom = volatile
```

### Layer Sharing — Disk and Network Efficiency

Layers are identified by their **SHA256 hash**. If two images share the same base, those layers are stored only once on disk and transferred only once over the network.

```
Image A (Node API):                 Image B (Node Worker):
┌─────────────────────┐             ┌─────────────────────┐
│ Layer 5: API code   │             │ Layer 5: Worker code │
│ Layer 4: npm install│             │ Layer 4: npm install │
│ Layer 3: package.json│            │ Layer 3: package.json│
├─────────────────────┤             ├─────────────────────┤
│ Layer 2: WORKDIR    │  ── same ── │ Layer 2: WORKDIR    │
│ Layer 1: node:20    │  ── same ── │ Layer 1: node:20    │
└─────────────────────┘             └─────────────────────┘
                    Shared on disk — stored once
```

This means:
- **Disk space** — 10 images from `node:20-alpine` don't store 10 copies of the base. One copy, shared.
- **Pulling images** — `docker pull` only downloads layers you don't already have. If you already have `node:20-alpine` locally, pulling a new image built on it only downloads the unique layers.
- **Pushing images** — same in reverse. The registry already has the shared layers; only your new layers are uploaded.

### .dockerignore — Keeping the Build Context Clean

When you run `docker build .`, Docker scans the entire directory as the **build context**. If your project has a 2 GB `data/` folder, build artifacts, or `node_modules/`, they all get scanned — and broad `COPY . .` instructions will pull them into the image.

A `.dockerignore` file excludes files from the build context entirely:

```
node_modules
.git
*.log
data/
dist/
.env
```

This speeds up builds (less to scan and copy) and keeps images small (no junk included).

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
