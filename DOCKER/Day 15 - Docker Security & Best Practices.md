# Day 15 — Docker Security & Best Practices
## Security Scanning, Secrets Management, Container Hardening

---

## Learning Goals

By the end of this class, students should be able to:

- Explain why running containers as root is dangerous and how to prevent it.
- Apply read-only filesystems and drop Linux capabilities to limit blast radius.
- Understand the difference between Docker secrets and environment variables.
- Scan Docker images for CVEs using Trivy.
- Choose the right minimal base image (Alpine, distroless) for production.
- Harden an insecure Dockerfile by applying all security best practices.
- Produce a security checklist for any production Dockerfile.

---

# Part A — Theory

## 1. Running Containers as Non-Root

### Why root in a container is dangerous

By default, Docker containers run all processes as **root (UID 0)**. This feels safe because the container is isolated — but isolation is not perfect.

If an attacker exploits a vulnerability in your application and gains code execution inside the container, they are already root inside it. From there:

- If a volume is mounted, they can write to host directories.
- If the Docker socket is mounted (`/var/run/docker.sock`), they own the entire host.
- Kernel exploits and container escape techniques are far more likely to succeed as root.
- Sensitive files readable only by root (like mounted secrets) are immediately accessible.

Running as a non-root user does not eliminate all risk, but it applies **least-privilege** — the attacker gains an unprivileged user at most.

### Creating and switching to a non-root user

**Alpine Linux:**

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

**Debian/Ubuntu:**

```dockerfile
RUN groupadd --system appgroup && \
    useradd --system --gid appgroup --no-create-home appuser
USER appuser
```

**Using the built-in user in official Node images:**

The official `node` image already ships with a non-root user called `node` (UID 1000):

```dockerfile
FROM node:20-alpine
# No need to create a user — just switch to the built-in one
USER node
```

### Verifying at runtime

```bash
docker run --rm node:20-alpine whoami      # outputs: node
docker run --rm node:20-alpine id          # uid=1000(node) gid=1000(node)
```

### Enforcing non-root at runtime

You can also enforce non-root when running a container, independent of what the image specifies:

```bash
docker run --user 1000:1000 myimage
```

---

## 2. Read-Only Filesystems

### Why writable container filesystems are a risk

A container with a writable filesystem lets an attacker who gains execution:
- Write malware or backdoors to disk.
- Modify application binaries.
- Create setuid files.
- Exfiltrate data by writing to disk and then reading it from a mounted volume.

### Making the root filesystem read-only

```bash
docker run --read-only myimage
```

In Docker Compose:

```yaml
services:
  app:
    image: myimage
    read_only: true
```

### Handling directories that must be writable

Most applications need to write somewhere — temp files, PID files, logs. Allow writes only in specific, controlled directories using `tmpfs`:

```bash
docker run --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --tmpfs /var/run:rw,noexec,nosuid \
  myimage
```

In Docker Compose:

```yaml
services:
  app:
    image: myimage
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m
      - /var/run:rw,noexec,nosuid
```

`noexec` — cannot execute binaries from this mount.  
`nosuid` — setuid/setgid bits are ignored (prevents privilege escalation).  
`size=64m` — limits tmpfs to 64 MB (prevents disk-fill attacks).

---

## 3. Dropping Linux Capabilities

### What are Linux capabilities?

Traditionally, Linux had two privilege levels: root (can do everything) and non-root (limited). **Capabilities** break "root power" into ~40 fine-grained privileges.

Even when a container runs as root, Docker drops many capabilities by default. But it still grants more than most applications need.

Common capabilities and their meaning:

| Capability         | What it allows                                          |
|--------------------|---------------------------------------------------------|
| `NET_ADMIN`        | Configure network interfaces, firewall rules            |
| `SYS_ADMIN`        | Mount filesystems, change namespaces — very dangerous   |
| `SYS_PTRACE`       | Trace/debug other processes                             |
| `NET_RAW`          | Use raw/packet sockets (can sniff network traffic)      |
| `CHOWN`            | Change file ownership arbitrarily                       |
| `SETUID`/`SETGID`  | Switch to any UID/GID                                   |
| `DAC_OVERRIDE`     | Bypass file permission checks                           |

### Drop all capabilities, add back only what is needed

**Best practice: drop all capabilities, then explicitly allow only what your app requires.**

```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myimage
```

`NET_BIND_SERVICE` allows binding to ports below 1024 (like 80/443). If your app listens on port 3000+, you do not even need that.

In Docker Compose:

```yaml
services:
  app:
    image: myimage
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE    # only if listening on port < 1024
```

Most Node.js, Python, and Go applications need **zero** Linux capabilities. Drop them all:

```yaml
cap_drop:
  - ALL
```

### `--security-opt no-new-privileges`

Prevents processes inside the container from gaining new privileges via setuid binaries or file capabilities, even if they try:

```bash
docker run --security-opt no-new-privileges:true myimage
```

```yaml
services:
  app:
    security_opt:
      - no-new-privileges:true
```

---

## 4. Docker Secrets vs Environment Variables

### The problem with environment variables for secrets

Environment variables are the most common way to pass secrets (passwords, API keys, tokens) into containers. They are convenient but have real security drawbacks:

| Problem | Detail |
|---------|--------|
| Visible in `docker inspect` | Any user with Docker access can read all env vars |
| Visible in `/proc/1/environ` | A process inside the container can read them |
| Logged accidentally | Crash dumps, debug logs, and error handlers often print env vars |
| Stored in image history | `ARG` values baked into images are visible with `docker history` |
| Leaked in `docker-compose.yml` if committed | Devs frequently commit files with real secrets |

Check how exposed env vars are:

```bash
docker run -d --name test -e DB_PASSWORD=supersecret alpine sleep 60
docker inspect test | grep DB_PASSWORD        # visible in plain text
docker exec test cat /proc/1/environ          # visible from inside too
docker rm -f test
```

### Docker Secrets (Swarm mode)

Docker's native secret management is built into **Swarm mode**. Secrets are:
- Encrypted at rest and in transit.
- Mounted as files in `/run/secrets/` inside the container — not as environment variables.
- Never stored in image layers or visible in `docker inspect`.

```bash
echo "supersecret" | docker secret create db_password -
```

```yaml
# docker-compose.yml (Swarm mode)
services:
  app:
    image: myapp
    secrets:
      - db_password

secrets:
  db_password:
    external: true
```

Inside the container, the app reads the secret from a file:

```javascript
const fs = require('fs');
const dbPassword = fs.readFileSync('/run/secrets/db_password', 'utf8').trim();
```

### Secrets in Docker Compose (non-Swarm)

In standard Compose (not Swarm), you can simulate file-based secrets using bind mounts from a local secrets file:

```yaml
services:
  app:
    image: myapp
    volumes:
      - ./secrets/db_password.txt:/run/secrets/db_password:ro
```

The `:ro` flag mounts it read-only inside the container.

### Practical Hierarchy: From Worst to Best

```text
Worst  │  Hardcoded in Dockerfile or source code
       │  ENV in Dockerfile (baked into image)
       │  ARG in Dockerfile (visible in docker history)
       │  .env file committed to git
       │  Environment variables at runtime (-e flag)
       │  Secrets manager (AWS Secrets Manager, Vault) read at startup
Best   │  Docker Secrets (Swarm) or Kubernetes Secrets + file mount
```

### What to do in practice (without Swarm)

1. Never hardcode secrets in Dockerfiles or source code.
2. Never commit `.env` files — add to `.gitignore`.
3. Use `--password-stdin` for docker login, never `-p` on the command line.
4. In CI/CD, inject secrets as environment variables from the pipeline's secret store (GitHub Secrets, GitLab CI variables, etc.) — these are not stored in logs.
5. For production workloads: use AWS Secrets Manager, HashiCorp Vault, or GCP Secret Manager, and read the secret value at application startup.

---

## 5. Trivy — Image Vulnerability Scanning

### What is Trivy?

**Trivy** is an open-source security scanner by Aqua Security. It scans Docker images for known CVEs (Common Vulnerabilities and Exposures) in:

- OS packages (Alpine, Debian, Ubuntu, RHEL)
- Language dependencies (npm, pip, gem, go.sum, cargo)
- Misconfigurations in Dockerfiles and Kubernetes manifests

### Install Trivy

```bash
# Ubuntu / Debian
sudo apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
  sudo gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
  https://aquasecurity.github.io/trivy-repo/deb generic main" | \
  sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install -y trivy
```

Or run Trivy itself as a Docker container (no install needed):

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $HOME/.trivy-cache:/root/.cache \
  aquasec/trivy:latest image node:20
```

### Scan an image

```bash
# Basic scan — all severities
trivy image node:20

# Filter to only CRITICAL and HIGH vulnerabilities
trivy image --severity CRITICAL,HIGH node:20

# Scan your own built image
trivy image myapp:latest

# Output as JSON (for CI/CD pipelines)
trivy image --format json --output results.json myapp:latest

# Scan and fail (exit code 1) if CRITICAL CVEs are found
trivy image --exit-code 1 --severity CRITICAL myapp:latest
```

### Understanding Trivy output

```text
node:20 (debian 12.4)
====================
Total: 87 (UNKNOWN: 0, LOW: 47, MEDIUM: 25, HIGH: 12, CRITICAL: 3)

┌──────────────────────┬────────────────┬──────────┬───────────────────┬──────────────┬────────────────────────────────────────┐
│       Library        │ Vulnerability  │ Severity │ Installed Version │ Fixed Version│               Title                    │
├──────────────────────┼────────────────┼──────────┼───────────────────┼──────────────┼────────────────────────────────────────┤
│ openssl              │ CVE-2024-0727  │ HIGH     │ 3.0.11            │ 3.0.12       │ OpenSSL: denial of service via null ptr │
│ libexpat1            │ CVE-2023-52425 │ HIGH     │ 2.5.0-1           │ 2.6.0        │ expat: parsing crash                   │
└──────────────────────┴────────────────┴──────────┴───────────────────┴──────────────┴────────────────────────────────────────┘
```

Key columns:
- **Library** — the vulnerable package
- **Severity** — CRITICAL / HIGH / MEDIUM / LOW
- **Installed Version** — what is in the image now
- **Fixed Version** — upgrade to this version to fix it
- **Title** — brief CVE description

### Why Alpine has fewer CVEs

```bash
trivy image node:20          # ~80-150 vulnerabilities (Debian base)
trivy image node:20-alpine   # ~5-20 vulnerabilities (Alpine base)
```

Alpine uses `musl libc` and `busybox` instead of the large GNU toolchain. Fewer packages = smaller attack surface = fewer CVEs.

### Scan for misconfigurations in a Dockerfile

```bash
trivy config .
# or
trivy config Dockerfile
```

This detects issues like running as root, adding unnecessary capabilities, and missing health checks.

---

## 6. Minimal Base Images

The fewer packages an image contains, the fewer potential vulnerabilities exist in it. Choose the smallest base image that works for your application.

### Image Size and Attack Surface Comparison

```text
node:20          → ~1.1 GB, Debian, full toolchain, many packages
node:20-slim     → ~250 MB, Debian minimal, no build tools
node:20-alpine   → ~130 MB, Alpine, musl libc, very few packages
gcr.io/distroless/nodejs20-debian12 → ~160 MB, no shell, no package manager
```

### Alpine

`alpine` is based on Alpine Linux — a security-oriented, minimal distribution.

Pros:
- Very small (~5 MB base OS)
- Few packages = few CVEs
- `apk` package manager if you need to install something

Cons:
- Uses `musl libc` instead of `glibc` — rare compatibility issues with some native modules
- No shell by default in distroless variant (debugging harder)

```dockerfile
FROM node:20-alpine
```

### Distroless

**Distroless** images (from Google) contain only the application runtime and its dependencies — no shell (`/bin/sh`), no package manager, no utilities.

```dockerfile
# Multi-stage: build with Node, run with distroless
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY src/ ./src/

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=builder /app /app
USER nonroot
CMD ["src/server.js"]
```

Distroless pros:
- No shell = attacker cannot run shell commands even with code execution
- No package manager = cannot install tools after compromise
- Minimal CVE surface

Distroless cons:
- Cannot `docker exec mycontainer sh` for debugging (no shell)
- Use the `:debug` variant for troubleshooting: `gcr.io/distroless/nodejs20-debian12:debug`

### Choosing a base image

| Scenario                            | Recommended base          |
|-------------------------------------|---------------------------|
| Development / debugging             | `node:20` or `node:20-slim` |
| Production (standard security)      | `node:20-alpine`          |
| Production (high security)          | `gcr.io/distroless/nodejs20-debian12` |
| Compiled Go binary                  | `scratch` (zero-byte base) |

---

# Part B — Practical Lab

## Lab Goal

1. Scan a Node.js Docker image with Trivy and understand the output.
2. Take an intentionally insecure Dockerfile and harden it step by step.

---

## Step 1 — Install Trivy

```bash
sudo apt-get install -y wget apt-transport-https gnupg lsb-release

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
  sudo gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
  https://aquasecurity.github.io/trivy-repo/deb generic main" | \
  sudo tee /etc/apt/sources.list.d/trivy.list

sudo apt-get update && sudo apt-get install -y trivy
trivy --version
```

---

## Step 2 — Scan the Official Node.js Image

```bash
# Scan the full Debian-based node image (many CVEs expected)
trivy image --severity CRITICAL,HIGH node:20
```

Note the total count of CRITICAL and HIGH vulnerabilities.

Now scan the Alpine variant:

```bash
trivy image --severity CRITICAL,HIGH node:20-alpine
```

Compare the difference in vulnerability count. This alone makes a strong case for Alpine.

---

## Step 3 — The Insecure Dockerfile

Create a project directory:

```bash
mkdir ~/security-lab && cd ~/security-lab
```

Create `src/server.js`:

```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({ message: 'Security Lab', hostname: require('os').hostname() });
});

app.listen(PORT, () => console.log(`Listening on ${PORT}`));
```

Create `package.json`:

```json
{
  "name": "security-lab",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

Create the intentionally **insecure** Dockerfile — `Dockerfile.insecure`:

```dockerfile
# INSECURE — DO NOT USE IN PRODUCTION
# This Dockerfile contains many deliberate security problems for learning

FROM node:20

# Problem 1: Running as root (default — no USER instruction)

# Problem 2: No .dockerignore — copies everything including secrets
COPY . .

# Problem 3: Installing devDependencies in production image
# Problem 4: Not pinning npm install
RUN npm install

# Problem 5: Hardcoded secret in ENV
ENV DB_PASSWORD=supersecret123
ENV API_KEY=sk-prod-abc123xyz

# Problem 6: Running as root is never explicitly fixed
EXPOSE 3000
CMD ["node", "src/server.js"]
```

Build it and scan it:

```bash
docker build -f Dockerfile.insecure -t security-lab:insecure .
trivy image --severity CRITICAL,HIGH security-lab:insecure
```

Also scan for Dockerfile misconfigurations:

```bash
trivy config Dockerfile.insecure
```

Trivy will flag:
- Running as root user
- No `USER` instruction
- Pinning issues

---

## Step 4 — Create `.dockerignore`

```bash
cat > .dockerignore << 'EOF'
node_modules/
npm-debug.log
.git/
.gitignore
.env
.env.*
*.secret
secrets/
Dockerfile*
.dockerignore
coverage/
*.test.js
__tests__/
docs/
README.md
EOF
```

---

## Step 5 — Build the Hardened Dockerfile

Create `Dockerfile` (the secure version):

```dockerfile
# ── Stage 1: Install dependencies ────────────────────────────
FROM node:20-alpine AS deps

WORKDIR /app

# Copy only dependency manifests first — layer cache optimization
COPY package.json package-lock.json ./

# Install production dependencies only, clean npm cache in the same layer
RUN npm ci --omit=dev \
    && npm cache clean --force


# ── Stage 2: Production image ─────────────────────────────────
FROM node:20-alpine AS production

# Use the built-in non-root user from the official node image
# UID 1000 / GID 1000
RUN mkdir -p /app && chown -R node:node /app

WORKDIR /app

# Copy production node_modules from deps stage
COPY --from=deps --chown=node:node /app/node_modules ./node_modules

# Copy application source — chown to node user
COPY --chown=node:node src/ ./src/
COPY --chown=node:node package.json ./

# Switch to non-root user
USER node

# Declare expected environment variables without values
# Secrets are injected at runtime — never hardcoded
ENV NODE_ENV=production
ENV PORT=3000

EXPOSE 3000

# Use exec form (JSON array) — signals reach the process correctly
CMD ["node", "src/server.js"]
```

Build the hardened image:

```bash
docker build -t security-lab:hardened .
```

---

## Step 6 — Compare Image Sizes

```bash
docker images security-lab
```

Expected:

```text
REPOSITORY     TAG        IMAGE ID       SIZE
security-lab   hardened   ...            ~130 MB
security-lab   insecure   ...            ~1.1 GB
```

---

## Step 7 — Scan the Hardened Image

```bash
trivy image --severity CRITICAL,HIGH security-lab:hardened
```

Compare the CVE count with `security-lab:insecure`. The Alpine-based hardened image should have dramatically fewer vulnerabilities.

Scan for misconfigurations:

```bash
trivy config Dockerfile
```

Expect zero or minimal findings.

---

## Step 8 — Run with Runtime Hardening Flags

Combine image hardening with runtime flags for maximum security:

```bash
docker run -d \
  --name hardened-app \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=32m \
  --cap-drop ALL \
  --security-opt no-new-privileges:true \
  --user 1000:1000 \
  -p 3000:3000 \
  security-lab:hardened
```

Test it works:

```bash
curl http://localhost:3000
```

Verify it runs as non-root:

```bash
docker exec hardened-app whoami
# node

docker exec hardened-app id
# uid=1000(node) gid=1000(node) groups=1000(node)
```

Verify the root filesystem is read-only:

```bash
docker exec hardened-app touch /evil-file
# touch: /evil-file: Read-only file system

docker exec hardened-app touch /tmp/allowed-file
# succeeds — /tmp is tmpfs, writable
```

Cleanup:

```bash
docker stop hardened-app && docker rm hardened-app
```

---

# Assignment 15 — Security-Hardened Dockerfile Checklist

## Goal

Produce a reusable security checklist that can be applied to any Dockerfile in the final project. The checklist covers both Dockerfile best practices and runtime hardening.

---

## The Checklist — `dockerfile-security-checklist.md`

```markdown
# Dockerfile Security Checklist
## For Production Node.js (and General) Applications

Use this checklist before pushing any image to a registry or deploying to production.
Each item includes the rationale and the fix.

---

## 1. Base Image

- [ ] **Use a minimal base image**
  - Prefer `node:20-alpine` over `node:20` (Debian)
  - Consider `gcr.io/distroless/nodejs20-debian12` for maximum hardening
  - Never use `:latest` — pin to a specific version tag
  - _Why: fewer packages = fewer CVEs = smaller attack surface_

- [ ] **Pin the exact image digest or tag**
  ```dockerfile
  # Good
  FROM node:20.11-alpine3.19
  # Even better (immutable)
  FROM node:20.11-alpine3.19@sha256:abc123...
  ```

---

## 2. Non-Root User

- [ ] **Never run as root**
  - Use `USER node` (built-in in official node image) or create a dedicated user
  ```dockerfile
  RUN addgroup -S appgroup && adduser -S appuser -G appgroup
  USER appuser
  ```
  - _Why: limits blast radius if the application is compromised_

- [ ] **Set correct file ownership with `--chown`**
  ```dockerfile
  COPY --chown=node:node src/ ./src/
  ```

---

## 3. Secrets Management

- [ ] **No secrets hardcoded in the Dockerfile**
  - No passwords, API keys, or tokens in `ENV` or `ARG` instructions
  - `ARG` values are visible in `docker history` — never use for secrets

- [ ] **No `.env` files with real credentials committed to git**
  - Add `.env` and `*.secret` to `.gitignore` and `.dockerignore`

- [ ] **Secrets are injected at runtime, not baked into the image**
  - Use `-e` flags, CI/CD secret stores, or Docker Secrets
  - Prefer file-based secrets (`/run/secrets/`) over environment variables

- [ ] **No credentials in `docker build` arguments**
  ```dockerfile
  # BAD — visible in docker history
  ARG DB_PASSWORD
  ENV DB_PASSWORD=${DB_PASSWORD}
  ```

---

## 4. Filesystem and Dependencies

- [ ] **`.dockerignore` file is present and covers:**
  ```
  node_modules/
  .git/
  .env
  .env.*
  *.secret
  secrets/
  Dockerfile*
  .dockerignore
  coverage/
  *.test.js
  ```

- [ ] **Only production dependencies in the final image**
  ```dockerfile
  RUN npm ci --omit=dev && npm cache clean --force
  ```

- [ ] **Use multi-stage builds to exclude build tools**
  - Builder stage: install devDependencies, compile, build
  - Production stage: copy only compiled output and prod dependencies

- [ ] **Clean package manager caches in the same `RUN` layer**
  ```dockerfile
  RUN apt-get update && apt-get install -y curl \
      && rm -rf /var/lib/apt/lists/*
  ```

---

## 5. Image Instructions

- [ ] **Use exec form for `CMD` and `ENTRYPOINT`**
  ```dockerfile
  CMD ["node", "src/server.js"]       # Good — signals work correctly
  CMD node src/server.js              # Bad — wrapped in /bin/sh -c
  ```

- [ ] **Set `WORKDIR` explicitly — never use `cd` in RUN**
  ```dockerfile
  WORKDIR /app
  ```

- [ ] **`EXPOSE` the correct port (documentation)**
  ```dockerfile
  EXPOSE 3000
  ```

- [ ] **Use `COPY` not `ADD`** (unless extracting archives)

---

## 6. Layer Caching and Build Efficiency

- [ ] **Copy dependency files before source code**
  ```dockerfile
  COPY package.json package-lock.json ./
  RUN npm ci --omit=dev
  COPY src/ ./src/          # cache not busted unless package.json changes
  ```

- [ ] **Combine related `RUN` commands to minimize layers**

---

## 7. Vulnerability Scanning

- [ ] **Scan with Trivy before pushing**
  ```bash
  trivy image --severity CRITICAL,HIGH myapp:latest
  ```
  - No CRITICAL vulnerabilities allowed in production images
  - HIGH vulnerabilities documented with accepted risk or fixed

- [ ] **Scan Dockerfile for misconfigurations**
  ```bash
  trivy config Dockerfile
  ```

- [ ] **Regularly rebuild images to pick up OS-level security patches**
  - At minimum: weekly automated rebuild in CI

---

## 8. Runtime Hardening (docker run / docker-compose.yml)

- [ ] **Drop all Linux capabilities**
  ```yaml
  cap_drop:
    - ALL
  cap_add:
    - NET_BIND_SERVICE    # only if port < 1024
  ```

- [ ] **Set read-only root filesystem**
  ```yaml
  read_only: true
  tmpfs:
    - /tmp:rw,noexec,nosuid,size=64m
    - /var/run:rw,noexec,nosuid
  ```

- [ ] **Prevent privilege escalation**
  ```yaml
  security_opt:
    - no-new-privileges:true
  ```

- [ ] **Explicit user at runtime**
  ```yaml
  user: "1000:1000"
  ```

- [ ] **Set memory and CPU limits**
  ```yaml
  deploy:
    resources:
      limits:
        cpus: '0.50'
        memory: 256M
  ```

- [ ] **Use a named user-defined network — not the default bridge**
  ```yaml
  networks:
    - app-network
  ```

- [ ] **`restart: unless-stopped` for production services**

---

## 9. Complete Example — Hardened `docker-compose.yml` Service

```yaml
version: "3.9"

services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile
    image: yourname/myapp:v1.0.0
    container_name: myapp
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      PORT: 3000
    env_file:
      - .env                       # contains non-secret config only
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m
      - /var/run:rw,noexec,nosuid
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    user: "1000:1000"
    networks:
      - app-network
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 256M

networks:
  app-network:
    driver: bridge
```

---

## 10. Complete Example — Hardened `Dockerfile`

```dockerfile
# ── Stage 1: Dependencies ──────────────────────────────────────────────
FROM node:20-alpine AS deps

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci --omit=dev \
    && npm cache clean --force


# ── Stage 2: Production ────────────────────────────────────────────────
FROM node:20-alpine AS production

# Ensure /app is owned by the node user
RUN mkdir -p /app && chown -R node:node /app

WORKDIR /app

# Copy production dependencies from deps stage
COPY --from=deps --chown=node:node /app/node_modules ./node_modules

# Copy application source
COPY --chown=node:node src/ ./src/
COPY --chown=node:node package.json ./

# Switch to non-root user (UID 1000, built into official node image)
USER node

ENV NODE_ENV=production
ENV PORT=3000

EXPOSE 3000

CMD ["node", "src/server.js"]
```

---

## Scoring

Score your Dockerfile before submitting the final project:

| Category               | Items | Points each | Max |
|------------------------|-------|-------------|-----|
| Base image             | 2     | 5           | 10  |
| Non-root user          | 2     | 10          | 20  |
| Secrets management     | 4     | 5           | 20  |
| Filesystem/deps        | 4     | 5           | 20  |
| Trivy scan clean       | 2     | 5           | 10  |
| Runtime hardening      | 7     | 5           | 35  |
| **Total**              |       |             | **115** |

Minimum passing score for the final project: **80/115**
```

---

## Running the Trivy Scan as Part of Assignment

After completing the hardened Dockerfile, run the full scan suite and save results:

```bash
# Scan image
trivy image --severity CRITICAL,HIGH \
  --format table \
  security-lab:hardened 2>&1 | tee trivy-image-report.txt

# Scan Dockerfile for misconfigurations
trivy config --format table \
  Dockerfile 2>&1 | tee trivy-config-report.txt

echo "=== SCAN COMPLETE ==="
echo "Image report:  trivy-image-report.txt"
echo "Config report: trivy-config-report.txt"
```

Submit both report files alongside the checklist and Dockerfile.

---

## Assignment Deliverables

| File                               | Description                                      |
|------------------------------------|--------------------------------------------------|
| `Dockerfile`                       | Hardened production Dockerfile                   |
| `docker-compose.yml`               | Hardened Compose file with all security options  |
| `.dockerignore`                    | Comprehensive ignore file                        |
| `dockerfile-security-checklist.md` | Completed checklist with all items checked off   |
| `trivy-image-report.txt`           | Trivy scan output for the hardened image         |
| `trivy-config-report.txt`          | Trivy config scan output for the Dockerfile      |

---

# Quick Reference

## Security Hardening Cheat Sheet

### Dockerfile

```dockerfile
FROM node:20-alpine                          # minimal base
USER node                                    # non-root user
COPY --chown=node:node src/ ./src/           # correct file ownership
RUN npm ci --omit=dev && npm cache clean --force  # prod deps only
CMD ["node", "src/server.js"]                # exec form
```

### docker run

```bash
docker run \
  --read-only \                              # read-only root fs
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \  # writable scratch space
  --cap-drop ALL \                           # drop all capabilities
  --security-opt no-new-privileges:true \    # no privilege escalation
  --user 1000:1000 \                         # enforce non-root
  myapp
```

### Trivy

```bash
trivy image --severity CRITICAL,HIGH IMAGE   # scan image
trivy config Dockerfile                      # scan Dockerfile
trivy image --exit-code 1 --severity CRITICAL IMAGE  # fail CI on CRITICAL
```

### What Never Goes in a Dockerfile

```text
✗  ENV DB_PASSWORD=secret
✗  ARG API_KEY
✗  COPY .env .
✗  RUN curl https://example.com/secret-setup.sh | bash
✗  FROM myimage:latest  (unpinned)
✗  USER root  (or no USER instruction)
```
