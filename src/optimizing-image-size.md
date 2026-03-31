# Optimizing Image Size

Now that we know images are stacks of layers, and each layer is a
directory of files on disk — image size is just the total size of all
those directories. Smaller images mean faster pulls, faster deploys,
and less disk usage.

## Choose a Smaller Base Image

The base image (`FROM`) is usually the biggest layer. Different bases
have very different sizes:

```
Base image              Size
──────────              ────
ubuntu:22.04            ~77 MB
node:20                 ~1 GB
node:20-slim            ~200 MB
node:20-alpine          ~130 MB
alpine:3.19             ~5 MB
distroless/static       ~2 MB
```

`alpine` is a minimal Linux distribution — it has a package manager
(`apk`) and a shell, but very little else. `distroless` images go
further: they contain only your app and its runtime dependencies. No
shell, no package manager, no `ls`, no `cat`. If an attacker breaks
into the container, there are almost no tools to exploit.

The trade-off: smaller bases have fewer built-in tools, which can make
debugging harder. During development you might use `node:20`; for
production you switch to `node:20-alpine` or a multi-stage build.

## Multi-Stage Builds

A common problem: you need build tools (compilers, dev dependencies) to
**build** your app, but you don't need them to **run** it. Without
multi-stage builds, all those tools end up in the final image.

Multi-stage builds solve this by using multiple `FROM` instructions.
Each one starts a new stage. Only the final stage becomes the image:

```dockerfile
# Stage 1: build (thrown away)
FROM node:20 AS build
COPY . /app
WORKDIR /app
RUN npm install && npm run build

# Stage 2: production (this becomes the image)
FROM node:20-alpine
COPY --from=build /app/dist /app
CMD ["node", "/app/index.js"]
```

What happens:

1. Stage 1 uses the full `node:20` image (~1 GB) with all build tools.
   It installs dependencies, compiles the code, produces `/app/dist`.
2. Stage 2 starts fresh from `node:20-alpine` (~130 MB). It copies
   **only** the built output from stage 1. No `node_modules`, no source
   code, no build tools.

The final image is small and contains only what's needed to run.

### See it yourself

```bash
mkdir /tmp/multistage && cd /tmp/multistage

cat > app.js <<'EOF'
console.log("hello from optimized image");
EOF

cat > Dockerfile.single <<'EOF'
FROM node:20
COPY app.js /app/app.js
CMD ["node", "/app/app.js"]
EOF

cat > Dockerfile.multi <<'EOF'
FROM node:20 AS build
COPY app.js /app/app.js

FROM node:20-alpine
COPY --from=build /app/app.js /app/app.js
CMD ["node", "/app/app.js"]
EOF

docker build -t single-stage -f Dockerfile.single .
docker build -t multi-stage -f Dockerfile.multi .

docker images | grep stage
# multi-stage     ~130 MB
# single-stage    ~1 GB
```

Same app, dramatically different image size.

## Combine RUN Commands

Each `RUN` instruction creates a layer. If you install packages and
then clean up in **separate** `RUN` instructions, the cleanup doesn't
help — the files are already frozen in the previous layer:

```dockerfile
# BAD — cleanup is in a separate layer, original layer still has the files
RUN apt-get update
RUN apt-get install -y build-essential
RUN rm -rf /var/lib/apt/lists/*
```

Combine them so temporary files never persist in any layer:

```dockerfile
# GOOD — one layer, temp files are cleaned before the layer is saved
RUN apt-get update \
    && apt-get install -y build-essential \
    && rm -rf /var/lib/apt/lists/*
```

## Use .dockerignore

`COPY . /app` copies **everything** in the build context into the
image. Without a `.dockerignore` file, that includes `node_modules`,
`.git`, test files, IDE configs — files that bloat the image and
invalidate layer caches unnecessarily.

Create a `.dockerignore` at the project root:

```
node_modules
.git
.env
*.md
test/
```

This works like `.gitignore` — listed files are excluded from `COPY`
and `ADD` instructions. The image only gets what it actually needs.
