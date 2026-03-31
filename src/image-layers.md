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
Change only the `COPY` line's content and rebuild: Docker reuses the
cached `FROM` and `RUN` layers and only rebuilds from the changed layer
onward.
