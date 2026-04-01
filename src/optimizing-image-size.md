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
rust:1.78               ~1.5 GB
rust:1.78-slim          ~800 MB
rust:1.78-alpine        ~900 MB
ubuntu:22.04            ~77 MB
alpine:3.19             ~5 MB
distroless/static       ~2 MB
```

`alpine` is a minimal Linux distribution — it has a package manager
(`apk`) and a shell, but very little else. `distroless` images go
further: they contain only your app and its runtime dependencies. No
shell, no package manager, no `ls`, no `cat`. If an attacker breaks
into the container, there are almost no tools to exploit.

The trade-off: smaller bases have fewer built-in tools, which can make
debugging harder. But for Rust this matters less — once you compile a
release binary, you often don't need the Rust toolchain at all in the
final image. That's where multi-stage builds shine.

## Multi-Stage Builds

A common problem: you need the full Rust toolchain (compiler, cargo,
standard library) to **build** your app, but you don't need any of it
to **run** the compiled binary. Without multi-stage builds, the entire
~1.5 GB Rust image ends up in production.

Multi-stage builds solve this by using multiple `FROM` instructions.
Each one starts a new stage. Only the final stage becomes the image:

```dockerfile
# Stage 1: build (thrown away)
FROM rust:1.78 AS build
COPY . /app
WORKDIR /app
RUN cargo build --release

# Stage 2: production (this becomes the image)
FROM debian:bookworm-slim
COPY --from=build /app/target/release/myapp /usr/local/bin/myapp
CMD ["myapp"]
```

What happens:

1. Stage 1 uses the full `rust:1.78` image (~1.5 GB) with the entire
   Rust toolchain. It compiles your code into a release binary.
2. Stage 2 starts fresh from `debian:bookworm-slim` (~80 MB). It copies
   **only** the compiled binary from stage 1. No Rust compiler, no
   source code, no `target/` directory, no cargo registry cache.

The final image is small and contains only what's needed to run.

For even smaller images, if your Rust binary is statically linked
(using `musl`), you can go all the way down to `scratch`:

```dockerfile
FROM rust:1.78 AS build
RUN rustup target add x86_64-unknown-linux-musl
COPY . /app
WORKDIR /app
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM scratch
COPY --from=build /app/target/x86_64-unknown-linux-musl/release/myapp /myapp
CMD ["/myapp"]
```

This produces an image that contains **nothing but your binary** — a
few megabytes total.

### See it yourself

```bash
mkdir /tmp/multistage && cd /tmp/multistage

cat > main.rs <<'EOF'
fn main() {
    println!("hello from optimized image");
}
EOF

cat > Dockerfile.single <<'EOF'
FROM rust:1.78
COPY main.rs /app/main.rs
WORKDIR /app
RUN rustc main.rs -o myapp
CMD ["./myapp"]
EOF

cat > Dockerfile.multi <<'EOF'
FROM rust:1.78 AS build
COPY main.rs /app/main.rs
WORKDIR /app
RUN rustc main.rs -o myapp

FROM debian:bookworm-slim
COPY --from=build /app/myapp /usr/local/bin/myapp
CMD ["myapp"]
EOF

docker build -t single-stage -f Dockerfile.single .
docker build -t multi-stage -f Dockerfile.multi .

docker images | grep stage
# multi-stage     ~80 MB
# single-stage    ~1.5 GB
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
image. Without a `.dockerignore` file, that includes `target/`,
`.git`, test fixtures, IDE configs — files that bloat the image and
invalidate layer caches unnecessarily.

Create a `.dockerignore` at the project root:

```
target
.git
.env
*.md
```

This works like `.gitignore` — listed files are excluded from `COPY`
and `ADD` instructions. The image only gets what it actually needs.
Excluding `target/` is especially important for Rust projects — the
build directory can be gigabytes in size and should be rebuilt inside
the image anyway.
