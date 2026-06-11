# syntax=docker/dockerfile:1
#
# OpenCode Server Deployment
# ---------------------------
# Stage 1 (builder): Alpine + Bun → builds opencode standalone binary
#   The binary embeds the web UI and links musl statically, so it runs
#   on any Linux regardless of the host glibc version (CentOS 7 compatible).
# Stage 2 (runtime): Alpine slim + git/rg → runs `opencode web`
#
# Build:  docker build -f deploy/Dockerfile -t opencode-server .
# Run:    see deploy/docker-compose.yml

# ── Stage 1: Build ──────────────────────────────────────────────────────────
FROM oven/bun:1.3-alpine AS builder

# Build tools needed for native modules (node-pty, @parcel/watcher, etc.)
RUN apk add --no-cache python3 build-base git

WORKDIR /app

# Copy everything (root .dockerignore excludes node_modules/dist for us)
COPY . .

# Build opencode binary.
# `--single`  → only build for current platform (linux-x64 in Docker)
# NOTE: we do NOT pass --skip-install. The build script runs its own
# `bun install --os="*" --cpu="*"` to fetch cross-platform native binaries
# (including @opentui/core-linux-x64-musl). Skipping it would leave the
# embedded binary without the correct musl native modules.
RUN bun install --ignore-scripts && \
    bun run --cwd packages/opencode build --single

# ── Stage 2: Runtime ───────────────────────────────────────────────────────
FROM oven/bun:1.3-alpine

RUN apk add --no-cache \
    git        \
    ripgrep    \
    openssh-client \
    ca-certificates

# Bring the compiled binary over from the builder
COPY --from=builder /app/packages/opencode/dist/opencode-linux-x64/bin/opencode /usr/local/bin/opencode
RUN chmod +x /usr/local/bin/opencode

# Default workspace — /home and /www are mounted via docker-compose volumes.
# WORKDIR here is just the container's starting directory; opencode's web UI
# lets you pick which project to work on at runtime.
WORKDIR /home

# 4096 is the default port when --port is not set; we pass it explicitly.
EXPOSE 4096

# Run headless server with embedded web UI.
# Auth: set OPENCODE_SERVER_PASSWORD in docker-compose or .env.
ENTRYPOINT ["opencode"]
CMD ["web", "--hostname", "0.0.0.0", "--port", "4096"]
