# Day 12 — Dockerfile & Image Building
## Writing Production-Quality Dockerfiles, Multi-Stage Builds, Best Practices

---

## Learning Goals

By the end of this class, students should be able to:

- Write a complete Dockerfile using all core instructions.
- Explain how Docker layer caching works and how to optimize for it.
- Build multi-stage Dockerfiles that separate build-time and runtime concerns.
- Use `.dockerignore` to exclude unnecessary files from the build context.
- Run containers as a non-root user for security.
- Reduce a bloated image from 800 MB to under 150 MB.

---

# Part A — Theory

## 1. What is a Dockerfile?

A **Dockerfile** is a plain text file with instructions that tell Docker how to build an image — step by step.

Each instruction in a Dockerfile creates one **layer** in the final image.

Workflow:

```text
Dockerfile  ──► docker build ──► Image ──► docker run ──► Container
```

---

## 2. Dockerfile Instructions

### `FROM` — Base Image

Every Dockerfile starts with `FROM`. It sets the base image your image builds on top of.

```dockerfile
FROM ubuntu:24.04
FROM node:20-alpine
FROM python:3.12-slim
```

**Best practice:** Use specific version tags, never `latest` in production. `latest` can silently break builds when the upstream image updates.

```dockerfile
# Bad
FROM node:latest

# Good
FROM node:20.11-alpine3.19
```

---

### `RUN` — Execute Commands During Build

`RUN` executes a shell command inside the image during the build process. The result is saved as a new layer.

```dockerfile
RUN apt-get update && apt-get install -y curl
```

**Best practice — combine related commands with `&&`:**

```dockerfile
# Bad — creates 3 separate layers, wastes space
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# Good — single layer, smaller image
RUN apt-get update \
    && apt-get install -y curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

> Deleting cache in the same `RUN` step is important. If you delete it in a later step, it still exists in the earlier layer and adds to the total image size.

---

### `COPY` — Copy Files into the Image

`COPY` copies files or directories from the build context (your project folder) into the image.

```dockerfile
COPY package.json .
COPY src/ ./src/
COPY . .
```

Syntax:

```text
COPY <source-on-host> <destination-in-image>
```

`COPY` vs `ADD`:

| Command | Use case |
|---------|----------|
| `COPY`  | Copy local files — prefer this almost always |
| `ADD`   | Can also extract `.tar` archives and fetch URLs — avoid unless needed |

**Best practice:** Use `COPY` over `ADD` unless you specifically need `ADD`'s extra features.

---

### `WORKDIR` — Set Working Directory

`WORKDIR` sets the current working directory for all following instructions (`RUN`, `COPY`, `CMD`, etc.).

```dockerfile
WORKDIR /app
```

If the directory does not exist, Docker creates it automatically.

**Best practice:** Always use `WORKDIR` instead of `RUN cd /app`. Using `cd` in a `RUN` step only affects that single layer; `WORKDIR` persists for all following instructions.

```dockerfile
# Bad
RUN cd /app && node server.js

# Good
WORKDIR /app
CMD ["node", "server.js"]
```

---

### `ENV` — Set Environment Variables

`ENV` sets environment variables that are available both during the build and at container runtime.

```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
ENV APP_HOME=/app
```

Multiple variables can be set in one instruction:

```dockerfile
ENV NODE_ENV=production \
    PORT=3000 \
    LOG_LEVEL=info
```

Access in your application code:

```javascript
const port = process.env.PORT || 3000;
```

Override at runtime:

```bash
docker run -e PORT=8080 myapp
```

---

### `EXPOSE` — Document a Port

`EXPOSE` documents which port the application inside the container listens on.

```dockerfile
EXPOSE 3000
```

> **Important:** `EXPOSE` does **not** publish the port to the host. It is documentation only. You still need `-p 3000:3000` in `docker run` to access it from outside.

---

### `CMD` — Default Command

`CMD` sets the default command that runs when a container starts.

```dockerfile
CMD ["node", "server.js"]
```

There can only be **one** `CMD` in a Dockerfile. If you specify multiple, only the last one takes effect.

`CMD` can be overridden at `docker run`:

```bash
docker run myapp node other-script.js
```

**Exec form vs shell form:**

```dockerfile
# Shell form — runs as /bin/sh -c "node server.js"
CMD node server.js

# Exec form — runs directly, no shell wrapper (preferred)
CMD ["node", "server.js"]
```

**Best practice:** Always use exec form (JSON array). Shell form wraps the process in a shell, which means signals like `SIGTERM` (from `docker stop`) may not reach your application correctly.

---

### `ENTRYPOINT` — Fixed Starting Command

`ENTRYPOINT` sets a command that always runs. Unlike `CMD`, it cannot be overridden by arguments in `docker run`.

```dockerfile
ENTRYPOINT ["node", "server.js"]
```

`ENTRYPOINT` + `CMD` together:

```dockerfile
ENTRYPOINT ["node"]
CMD ["server.js"]
```

`CMD` acts as the **default argument** to `ENTRYPOINT`. You can override `CMD` without changing `ENTRYPOINT`:

```bash
docker run myapp other-script.js   # runs: node other-script.js
```

**When to use which:**

| Instruction   | Use when                                                   |
|---------------|------------------------------------------------------------|
| `CMD`         | The container has a sensible default but it can be changed |
| `ENTRYPOINT`  | The container always runs a specific executable            |
| Both together | Fixed executable, flexible arguments                       |

---

## 3. Layer Caching

Docker caches each layer. When you rebuild an image, Docker reuses cached layers for instructions that have not changed — and only rebuilds from the first changed instruction downward.

This makes rebuilds fast — **but only if you order instructions correctly**.

### Cache Invalidation

A layer is invalidated (cache is broken) when:
- The instruction itself changes.
- Any file copied by `COPY` or `ADD` changes.
- Any previous layer is invalidated.

### Optimization: Copy Dependencies Before Source Code

Bad order — every code change rebuilds npm install:

```dockerfile
COPY . .                     # copies everything including source code
RUN npm install              # cache broken every time any file changes
```

Good order — npm install only re-runs when package.json changes:

```dockerfile
COPY package.json package-lock.json ./   # only copy dependency files first
RUN npm install                          # cached unless package.json changes
COPY . .                                 # copy source code last
```

Visual comparison:

```text
Bad order:                         Good order:
┌───────────────────────┐         ┌───────────────────────┐
│ COPY . .              │ ←dirty  │ COPY package*.json    │ ← rarely changes
├───────────────────────┤         ├───────────────────────┤
│ RUN npm install       │ ←rebuilt│ RUN npm install       │ ← cached most of the time
├───────────────────────┤         ├───────────────────────┤
│ ...                   │         │ COPY . .              │ ← changes often, but cheap
└───────────────────────┘         └───────────────────────┘
```

**Rule of thumb:** Put instructions that change infrequently near the top, and instructions for frequently-changing files near the bottom.

---

## 4. Multi-Stage Builds

### The Problem

To build a Node.js or Go application, you need build tools (compilers, dev dependencies, build scripts). But you do **not** need these tools in the final running image. Shipping them wastes disk space and increases the attack surface.

A naive Node.js image often reaches 800 MB–1 GB because it includes:
- Full node image
- All `devDependencies`
- npm cache
- Build tools like TypeScript compiler

### The Solution: Multi-Stage Builds

A multi-stage Dockerfile uses **multiple `FROM` statements**. Each `FROM` starts a new build stage with its own filesystem.

You can selectively `COPY` files from one stage into another, leaving behind everything you don't need.

```dockerfile
# ── Stage 1: Build ──────────────────────────────────────────
FROM node:20 AS builder

WORKDIR /app
COPY package*.json ./
RUN npm install                  # installs devDependencies too
COPY . .
RUN npm run build                # compiles TypeScript, bundles, etc.

# ── Stage 2: Production ─────────────────────────────────────
FROM node:20-alpine AS production

WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev       # production dependencies only
COPY --from=builder /app/dist ./dist   # copy compiled output only

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

What gets left behind in the `builder` stage (never reaches the final image):
- TypeScript compiler
- All `devDependencies`
- Source `.ts` files
- npm cache
- Build scripts

Result: the final image contains only the Alpine Node.js runtime + production dependencies + compiled output.

---

## 5. `.dockerignore`

`.dockerignore` works exactly like `.gitignore`. It tells Docker which files and directories to **exclude** from the build context before sending it to the daemon.

Why it matters: when you run `docker build .`, Docker sends your entire project directory to the daemon. Without `.dockerignore`, this includes `node_modules` (potentially hundreds of MB), `.git`, logs, secrets, and local configs.

Create a `.dockerignore` file in your project root:

```text
# Dependencies (will be installed fresh inside the container)
node_modules/
npm-debug.log
yarn-error.log

# Build output (copied from builder stage, not from host)
dist/
build/
.next/

# Version control
.git/
.gitignore

# Environment files (never put secrets in images)
.env
.env.local
.env.*.local

# Editor and OS files
.DS_Store
.vscode/
*.swp
*.swo
Thumbs.db

# Test and documentation
coverage/
*.test.js
*.spec.js
__tests__/
docs/

# Docker files themselves
Dockerfile*
.dockerignore
```

Benefits:
- Faster builds (less data sent to daemon)
- Smaller images
- No accidental secrets leaked into images

---

## 6. Non-Root User for Security

By default, processes inside Docker containers run as **root** (UID 0). This is a security risk: if an attacker exploits your application, they gain root access inside the container, which can be leveraged to escape to the host in misconfigured setups.

**Best practice:** Always create and switch to a non-root user.

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Create a non-root user and group
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY package*.json ./
RUN npm install --omit=dev
COPY . .

# Give ownership of /app to the non-root user
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

EXPOSE 3000
CMD ["node", "server.js"]
```

`USER` instruction switches the active user for all following `RUN`, `CMD`, and `ENTRYPOINT` instructions.

> Note: Some official images (like `node`) already include a built-in non-root user named `node`. You can use it directly with `USER node`.

---

# Part B — Practical Lab

## Lab Goal

Write a Dockerfile for a Node.js Express application, then improve it with multi-stage builds, caching optimization, `.dockerignore`, and a non-root user.

---

## Step 1 — Create the Express App

```bash
mkdir ~/express-docker && cd ~/express-docker
```

Initialize the project:

```bash
npm init -y
npm install express
```

Create `src/server.js`:

```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Dockerized Express!',
    environment: process.env.NODE_ENV || 'development',
    hostname: require('os').hostname(),
  });
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT} in ${process.env.NODE_ENV} mode`);
});
```

Test it locally first:

```bash
node src/server.js
curl http://localhost:3000
```

Stop the server with `Ctrl+C`.

---

## Step 2 — Write a Basic Dockerfile (Version 1)

Create `Dockerfile.v1`:

```dockerfile
FROM node:20

WORKDIR /app

COPY . .

RUN npm install

EXPOSE 3000

CMD ["node", "src/server.js"]
```

Build and check the size:

```bash
docker build -f Dockerfile.v1 -t express-v1 .
docker images express-v1
```

Note the image size — likely 1 GB+. This is the baseline to improve.

---

## Step 3 — Optimize Layer Caching (Version 2)

Create `Dockerfile.v2`:

```dockerfile
FROM node:20

WORKDIR /app

# Copy dependency files first — cached unless package.json changes
COPY package.json package-lock.json ./

RUN npm install

# Copy source code last — changes here don't bust npm install cache
COPY src/ ./src/

EXPOSE 3000

ENV NODE_ENV=production

CMD ["node", "src/server.js"]
```

Build:

```bash
docker build -f Dockerfile.v2 -t express-v2 .
```

Now make a small change to `src/server.js` (change the message text) and rebuild:

```bash
docker build -f Dockerfile.v2 -t express-v2 .
```

Observe: the `npm install` step says `CACHED`. Only the `COPY src/` step re-runs.

---

## Step 4 — Create `.dockerignore`

Create `.dockerignore` in the project root:

```text
node_modules/
npm-debug.log
.git/
.gitignore
.env
.env.*
Dockerfile*
.dockerignore
coverage/
*.test.js
```

Rebuild and notice the build context size printed in the first line is now much smaller:

```bash
docker build -f Dockerfile.v2 -t express-v2 .
```

---

## Step 5 — Multi-Stage Build with Non-Root User (Version 3 — Production)

Create `Dockerfile`:

```dockerfile
# ── Stage 1: Dependencies ────────────────────────────────────
FROM node:20-alpine AS deps

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci --omit=dev \
    && npm cache clean --force


# ── Stage 2: Production Image ────────────────────────────────
FROM node:20-alpine AS production

# Create non-root user
RUN addgroup -S appgroup \
    && adduser -S appuser -G appgroup

WORKDIR /app

# Copy only production node_modules from the deps stage
COPY --from=deps /app/node_modules ./node_modules

# Copy application source
COPY src/ ./src/
COPY package.json ./

# Set ownership
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

ENV NODE_ENV=production
ENV PORT=3000

EXPOSE 3000

CMD ["node", "src/server.js"]
```

Build:

```bash
docker build -t express-v3 .
docker images express-v3
```

Compare all three versions:

```bash
docker images | grep express
```

Expected results:

```text
express-v3   latest   ...   ~130 MB   ← production-ready
express-v2   latest   ...   ~1.1 GB
express-v1   latest   ...   ~1.1 GB
```

---

## Step 6 — Run and Test the Production Image

```bash
docker run -d --name express-app -p 3000:3000 express-v3
curl http://localhost:3000
curl http://localhost:3000/health
```

Verify the process runs as a non-root user:

```bash
docker exec express-app whoami
```

Expected:

```text
appuser
```

Check it cannot write to root-owned paths:

```bash
docker exec express-app touch /etc/test
```

Expected:

```text
touch: /etc/test: Permission denied
```

This is the security benefit of running as a non-root user.

---

## Step 7 — Inspect the Final Image

View layers and their sizes:

```bash
docker image history express-v3
```

Inspect full metadata:

```bash
docker inspect express-v3 | grep -A5 "Env\|User\|ExposedPorts"
```

Verify the non-root user is baked into the image config:

```bash
docker inspect express-v3 --format '{{.Config.User}}'
```

Expected:

```text
appuser
```

---

## Cleanup

```bash
docker stop express-app
docker rm express-app
docker rmi express-v1 express-v2 express-v3
```

---

# Assignment 12 — Optimize a Dockerfile from 800 MB to Under 150 MB

## The Problem

You are given this bloated Dockerfile for a Node.js application. It produces an image over 800 MB.

### Bloated `Dockerfile` (given — do not run as-is):

```dockerfile
FROM node:20

WORKDIR /app

COPY . .

RUN npm install

RUN npm run build

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

### Bloated `package.json` (given):

```json
{
  "name": "assignment-app",
  "version": "1.0.0",
  "scripts": {
    "build": "tsc"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "typescript": "^5.3.3",
    "@types/express": "^4.17.21",
    "@types/node": "^20.11.0"
  }
}
```

### `tsconfig.json` (given):

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true
  }
}
```

### `src/server.ts` (given):

```typescript
import express, { Request, Response } from 'express';

const app = express();
const PORT = process.env.PORT || 3000;

app.get('/', (req: Request, res: Response) => {
  res.json({ message: 'Optimized TypeScript + Docker!', hostname: require('os').hostname() });
});

app.listen(PORT, () => {
  console.log(`Listening on port ${PORT}`);
});
```

---

## Your Task

Rewrite the Dockerfile to bring the final image under **150 MB** using:

1. Multi-stage build (builder stage + production stage)
2. Alpine base image
3. Only production dependencies in the final image (`--omit=dev`)
4. Proper layer caching (copy `package.json` before source)
5. A non-root user
6. A `.dockerignore` file

---

## Solution

### `.dockerignore`

```text
node_modules/
dist/
.git/
.env
*.md
Dockerfile*
.dockerignore
coverage/
```

### `Dockerfile`

```dockerfile
# ── Stage 1: Build ───────────────────────────────────────────
FROM node:20-alpine AS builder

WORKDIR /app

# Install all dependencies including devDependencies for build
COPY package.json package-lock.json tsconfig.json ./

RUN npm ci

# Copy source and compile TypeScript → JavaScript
COPY src/ ./src/

RUN npm run build


# ── Stage 2: Production ──────────────────────────────────────
FROM node:20-alpine AS production

# Create non-root user
RUN addgroup -S appgroup \
    && adduser -S appuser -G appgroup

WORKDIR /app

# Copy package files and install only production dependencies
COPY package.json package-lock.json ./

RUN npm ci --omit=dev \
    && npm cache clean --force

# Copy compiled output from builder — no TypeScript, no devDeps
COPY --from=builder /app/dist ./dist

# Set ownership
RUN chown -R appuser:appgroup /app

USER appuser

ENV NODE_ENV=production
ENV PORT=3000

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

### Build and Verify

```bash
# Build the optimized image
docker build -t assignment-app .

# Check the size — should be under 150 MB
docker images assignment-app

# Run it
docker run -d --name assign-app -p 3000:3000 assignment-app
curl http://localhost:3000

# Confirm non-root user
docker exec assign-app whoami

# Cleanup
docker stop assign-app && docker rm assign-app
```

---

## Why Each Change Reduced Size

| Change                            | Size impact                                              |
|-----------------------------------|----------------------------------------------------------|
| `node:20` → `node:20-alpine`      | ~900 MB → ~130 MB base image (Alpine uses musl + busybox)|
| Multi-stage build                 | TypeScript compiler and devDeps never enter final image  |
| `npm ci --omit=dev`               | Removes `typescript`, `@types/*` from final image        |
| `npm cache clean --force`         | Removes npm's internal download cache from the layer     |
| `.dockerignore` excludes `node_modules/` | Host node_modules not copied into build context   |
| Non-root user                     | No size change, but required for production security     |

---

## Assignment Checklist

- [ ] Final image is under 150 MB (`docker images` confirms)
- [ ] Multi-stage build with separate `builder` and `production` stages
- [ ] Alpine base image used in both stages
- [ ] TypeScript compiled in builder stage, only `dist/` copied to production
- [ ] `npm ci --omit=dev` used in production stage
- [ ] Non-root user created and set with `USER`
- [ ] `.dockerignore` present and excludes `node_modules/`, `dist/`, `.git/`
- [ ] Container runs and responds on port 3000
- [ ] `docker exec assign-app whoami` returns a non-root username

---

# Quick Reference

## Dockerfile Instructions Summary

| Instruction   | Purpose                                        | Example                              |
|---------------|------------------------------------------------|--------------------------------------|
| `FROM`        | Set base image                                 | `FROM node:20-alpine`                |
| `WORKDIR`     | Set working directory                          | `WORKDIR /app`                       |
| `COPY`        | Copy files from host into image                | `COPY package.json ./`               |
| `RUN`         | Run command during build                       | `RUN npm install`                    |
| `ENV`         | Set environment variable                       | `ENV NODE_ENV=production`            |
| `EXPOSE`      | Document listening port                        | `EXPOSE 3000`                        |
| `USER`        | Switch to non-root user                        | `USER appuser`                       |
| `CMD`         | Default run command (overridable)              | `CMD ["node", "server.js"]`          |
| `ENTRYPOINT`  | Fixed run command                              | `ENTRYPOINT ["node"]`                |

## Build Commands

| Command                                    | Description                          |
|--------------------------------------------|--------------------------------------|
| `docker build -t name .`                   | Build image from Dockerfile in `.`   |
| `docker build -f Dockerfile.v2 -t name .` | Build from a specific Dockerfile     |
| `docker build --no-cache -t name .`        | Build without using cache            |
| `docker image history name`                | Show image layers and sizes          |
| `docker inspect name`                      | Full image metadata                  |

## Layer Caching Rules

1. Copy `package.json` before application source.
2. Put rarely-changing instructions near the top.
3. Put frequently-changing files (`COPY . .`) near the bottom.
4. Combine related `RUN` commands into one step with `&&`.
5. Clean caches in the same `RUN` step that created them.
