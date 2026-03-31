# Image Layers and the Filesystem

## How a Dockerfile Becomes the Container's Filesystem

Each Dockerfile instruction that modifies files creates a **layer**. These
layers stack up to form what the container sees as `/`.

```
  Dockerfile                          Container's /
  ┌──────────────────────────┐        ┌─────────────────────────┐
  │ FROM ubuntu:22.04        │───────>│ Layer 1: base OS        │
  │ COPY ./app /app          │───────>│ Layer 2: app files      │
  │ RUN npm install          │───────>│ Layer 3: dependencies   │
  │ CMD ["node", "app.js"]   │───────>│ Layer 4: config         │
  └──────────────────────────┘        └─────────────────────────┘
```

Note: `CMD` doesn't create a real filesystem layer — it only stores metadata.
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
