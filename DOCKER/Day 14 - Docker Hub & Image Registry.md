# Day 14 — Docker Hub & Image Registry
## Tagging Conventions, Pushing to Docker Hub, Private Registries

---

## Learning Goals

By the end of this class, students should be able to:

- Explain the full image naming format: `registry/username/repo:tag`.
- Apply semantic tagging strategies used in real CI/CD pipelines.
- Create a Docker Hub account and push images using access tokens.
- Understand the difference between public and private registries.
- Authenticate to Docker Hub, ECR, and GCR.
- Tag an image with a version and git commit SHA.
- Write a shell script that automates image tagging and pushing.

---

# Part A — Theory

## 1. Image Naming Format

Every Docker image has a fully qualified name made of four parts:

```text
registry/username/repository:tag

│         │         │          │
│         │         │          └── Version label (default: latest)
│         │         └───────────── Repository name (the app)
│         └─────────────────────── Namespace / Docker Hub username
└───────────────────────────────── Registry hostname (default: docker.io)
```

### Examples

| Image reference                                           | Registry  | Namespace | Repo  | Tag         |
| --------------------------------------------------------- | --------- | --------- | ----- | ----------- |
| `nginx`                                                   | docker.io | library   | nginx | latest      |
| `nginx:1.25-alpine`                                       | docker.io | library   | nginx | 1.25-alpine |
| `yourname/myapp:v1.0.0`                                   | docker.io | yourname  | myapp | v1.0.0      |
| `ghcr.io/yourname/myapp:main`                             | ghcr.io   | yourname  | myapp | main        |
| `123456789.dkr.ecr.ap-south-1.amazonaws.com/myapp:latest` | AWS ECR   | —         | myapp | latest      |

When no registry is specified, Docker defaults to `docker.io` (Docker Hub).  
When no tag is specified, Docker defaults to `latest`.

Official images (like `nginx`, `node`, `mongo`) are published under the special `library` namespace and can be referenced without a username.

---

## 2. Semantic Tagging for CI/CD

Tags are not just version numbers — they are a communication contract between the team that builds images and the systems that deploy them.

### `:latest`

`latest` always points to the most recently pushed image on that repository.

```bash
docker push yourname/myapp:latest
```

**Use in:** Local development, quick demos.

**Avoid in production deployments** because:
- "Latest" is ambiguous — different machines may pull different versions if the tag is updated.
- There is no way to roll back to a previous `latest`.
- It makes deployments non-reproducible.

---

### Semantic Versioning — `:v1.0.0`

Semantic versioning (`MAJOR.MINOR.PATCH`) makes releases explicit and rollback straightforward.

```text
v1.0.0   ← initial release
v1.0.1   ← patch: bug fix
v1.1.0   ← minor: new feature, backward compatible
v2.0.0   ← major: breaking change
```

```bash
docker tag myapp:latest yourname/myapp:v1.2.3
docker push yourname/myapp:v1.2.3
```

A common convention is to push multiple tags for the same image so consumers can choose their pinning strategy:

```bash
docker push yourname/myapp:v1.2.3    # exact version — most specific
docker push yourname/myapp:v1.2      # minor version — gets patches
docker push yourname/myapp:v1        # major version — gets all minor+patch
docker push yourname/myapp:latest    # always the newest
```

---

### Git Commit SHA — `:git-abc1234`

Every git commit has a unique 40-character SHA. Using a short SHA (7 characters) as a Docker tag gives you a direct, traceable link from a running container back to the exact code that built it.

```bash
GIT_SHA=$(git rev-parse --short HEAD)   # e.g. a3f9c2d
docker tag myapp:latest yourname/myapp:${GIT_SHA}
docker push yourname/myapp:${GIT_SHA}
```

**This is the standard in CI/CD pipelines** — every push to a branch creates an immutable image tagged with the commit SHA. You always know what code is running.

---

### Branch and Environment Tags

```bash
docker push yourname/myapp:main          # latest build from main branch
docker push yourname/myapp:develop       # latest build from develop branch
docker push yourname/myapp:staging       # image deployed to staging
docker push yourname/myapp:production    # image deployed to production
```

---

### Recommended Tagging Strategy (Real CI/CD)

For each build, push all of these:

```bash
yourname/myapp:v1.2.3           # exact — used for prod deployments and rollbacks
yourname/myapp:v1.2             # minor track
yourname/myapp:v1               # major track
yourname/myapp:git-a3f9c2d      # traceability to exact commit
yourname/myapp:main             # branch track
yourname/myapp:latest           # convenience
```

---

## 3. Docker Hub

**Docker Hub** (`hub.docker.com`) is Docker's default public registry.

### Key Concepts

| Term            | Meaning                                                      |
| --------------- | ------------------------------------------------------------ |
| Repository      | A collection of image tags under one name (`yourname/myapp`) |
| Tag             | A specific version of an image within a repository           |
| Public repo     | Anyone can pull without authentication                       |
| Private repo    | Requires authentication to pull (free tier: 1 private repo)  |
| Official image  | Verified images maintained by Docker or the software vendor  |
| Docker Verified | Published by verified organizations (e.g. `bitnami/`)        |

### Access Tokens (Preferred Over Password)

Docker Hub supports **Personal Access Tokens** (PATs) — these are used instead of your account password for `docker login`. This is the correct approach for:

- Automated CI/CD pipelines
- Shell scripts
- Team environments where the real password must not be shared

Access token benefits:
- Can be scoped: `Read-only`, `Read & Write`, or `Read, Write & Delete`
- Can be revoked individually without changing your account password
- Can be given a name so you know which pipeline or machine is using them

---

## 4. Private Registries

When you cannot or should not use a public registry, you use a **private registry**. Common options:

### Docker Hub Private Repository

The simplest option — just create a private repository on Docker Hub. Images require authentication to pull.

Free tier allows **1 private repository**. Paid tiers allow unlimited.

### Amazon Elastic Container Registry (ECR)

AWS's private container registry, tightly integrated with ECS, EKS, and CodePipeline.

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:latest
```

Authentication uses AWS IAM — no separate password:

```bash
aws ecr get-login-password --region ap-south-1 \
  | docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

### Google Container Registry / Artifact Registry (GCR)

Google Cloud's registry, integrated with GKE and Cloud Build.

```text
asia-south1-docker.pkg.dev/my-project/my-repo/myapp:latest
```

Authentication uses Google Cloud credentials:

```bash
gcloud auth configure-docker asia-south1-docker.pkg.dev
```

### GitHub Container Registry (GHCR)

GitHub's registry, integrated with GitHub Actions.

```text
ghcr.io/username/myapp:latest
```

```bash
echo $GITHUB_TOKEN | docker login ghcr.io -u username --password-stdin
```

### Self-Hosted Registry

Run Docker's open-source registry image on your own server:

```bash
docker run -d -p 5000:5000 --name registry registry:2
docker tag myapp localhost:5000/myapp:v1.0.0
docker push localhost:5000/myapp:v1.0.0
```

---

## 5. `docker login`, `push`, `pull` Authentication

### Login

```bash
# Interactive login (prompts for username and password)
docker login

# Login to Docker Hub with username flag
docker login -u yourusername

# Login to a private registry
docker login registry.example.com

# Non-interactive login using access token (for scripts)
echo "your-access-token" | docker login -u yourusername --password-stdin
```

`--password-stdin` reads the password from standard input rather than the command line. This prevents the token from appearing in shell history or process lists.

### Push

```bash
docker push yourname/myapp:v1.0.0
docker push yourname/myapp:latest
```

Before pushing, the image must be tagged with the full registry path:

```bash
docker tag localimage:latest yourname/myapp:v1.0.0
docker push yourname/myapp:v1.0.0
```

### Pull

```bash
# Pull from Docker Hub (public — no login needed)
docker pull yourname/myapp:v1.0.0

# Pull from a private registry (must be logged in)
docker pull yourname/private-app:v1.0.0

# Pull from ECR
docker pull 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:latest
```

### Logout

```bash
docker logout
docker logout registry.example.com
```

Credentials are stored in `~/.docker/config.json`. Logout removes them.

---

## 6. How Docker Stores Credentials

```bash
cat ~/.docker/config.json
```

```json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "base64encodedusername:token"
    }
  }
}
```

> **Security warning:** On Linux without a credential store configured, credentials are stored as base64 in this file. Base64 is **not encryption** — it can be decoded trivially. In production pipelines, use a credential store (AWS Secrets Manager, GitHub Secrets, etc.) and avoid storing credentials on disk.

---

# Part B — Practical Lab

## Lab Setup

You need:
- A Docker Hub account — create one free at `hub.docker.com`
- A Docker Hub Personal Access Token
- The Node.js Express app from Day 12 (or create a fresh one below)
- Git initialized in the project

---

## Step 1 — Create a Docker Hub Access Token

1. Go to `hub.docker.com` and log in.
2. Click your avatar → **Account Settings** → **Security** → **New Access Token**.
3. Name it `devops-lab` and set scope to **Read, Write & Delete**.
4. Copy the token — it is only shown once.

Store it safely (you will use it in the lab and in the assignment script).

---

## Step 2 — Login from the Terminal

```bash
# Replace YOURTOKEN with your actual access token
echo "YOURTOKEN" | docker login -u YOURUSERNAME --password-stdin
```

Expected output:

```text
Login Succeeded
```

Verify login was saved:

```bash
cat ~/.docker/config.json
```

---

## Step 3 — Prepare the Project

If you do not have the Day 12 app, create a minimal one:

```bash
mkdir ~/registry-lab && cd ~/registry-lab
git init
```

Create `src/server.js`:

```javascript
const http = require('http');
const os = require('os');

const PORT = process.env.PORT || 3000;
const VERSION = process.env.APP_VERSION || 'unknown';
const GIT_SHA = process.env.GIT_SHA || 'unknown';

http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({
    message: 'Registry Lab App',
    version: VERSION,
    git_sha: GIT_SHA,
    hostname: os.hostname()
  }));
}).listen(PORT, () => {
  console.log(`Running version=${VERSION} sha=${GIT_SHA} on port ${PORT}`);
});
```

Create `Dockerfile`:

```dockerfile
FROM node:20-alpine

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY src/ ./src/

RUN chown -R appuser:appgroup /app

USER appuser

ARG APP_VERSION=dev
ARG GIT_SHA=unknown
ENV APP_VERSION=${APP_VERSION}
ENV GIT_SHA=${GIT_SHA}

EXPOSE 3000

CMD ["node", "src/server.js"]
```

Create `.dockerignore`:

```text
.git/
.env
node_modules/
```

Make an initial git commit:

```bash
git add .
git commit -m "Initial commit"
```

---

## Step 4 — Build the Image with Build Args

```bash
GIT_SHA=$(git rev-parse --short HEAD)
APP_VERSION="v1.0.0"
DOCKERHUB_USERNAME="YOURUSERNAME"   # replace with your Docker Hub username

docker build \
  --build-arg APP_VERSION=${APP_VERSION} \
  --build-arg GIT_SHA=${GIT_SHA} \
  -t ${DOCKERHUB_USERNAME}/registry-lab:local .
```

Test it locally:

```bash
docker run --rm -p 3000:3000 ${DOCKERHUB_USERNAME}/registry-lab:local
curl http://localhost:3000
```

The response should show your `version` and `git_sha` values baked in.

---

## Step 5 — Tag the Image for Docker Hub

Apply all tags for this release:

```bash
DOCKERHUB_USERNAME="YOURUSERNAME"
APP_VERSION="v1.0.0"
GIT_SHA=$(git rev-parse --short HEAD)
IMAGE="${DOCKERHUB_USERNAME}/registry-lab"

# Tag with all strategies
docker tag ${IMAGE}:local ${IMAGE}:${APP_VERSION}
docker tag ${IMAGE}:local ${IMAGE}:git-${GIT_SHA}
docker tag ${IMAGE}:local ${IMAGE}:latest

# Verify all tags exist locally
docker images ${IMAGE}
```

Expected output — all tags pointing to the same IMAGE ID:

```text
REPOSITORY                  TAG           IMAGE ID       CREATED         SIZE
yourname/registry-lab       v1.0.0        a3f9c2d1b042   2 minutes ago   130MB
yourname/registry-lab       git-a3f9c2d   a3f9c2d1b042   2 minutes ago   130MB
yourname/registry-lab       latest        a3f9c2d1b042   2 minutes ago   130MB
yourname/registry-lab       local         a3f9c2d1b042   2 minutes ago   130MB
```

All four lines share the same IMAGE ID — they are the same image with different labels.

---

## Step 6 — Push to Docker Hub

```bash
IMAGE="${DOCKERHUB_USERNAME}/registry-lab"

docker push ${IMAGE}:v1.0.0
docker push ${IMAGE}:git-${GIT_SHA}
docker push ${IMAGE}:latest
```

Watch the layer upload progress. Notice that because all tags point to the same image, the layers are only uploaded once — subsequent pushes of the same layers say `Layer already exists`.

---

## Step 7 — Verify on Docker Hub

1. Go to `hub.docker.com/r/YOURUSERNAME/registry-lab`
2. Click the **Tags** tab
3. Verify all three tags appear: `v1.0.0`, `git-a3f9c2d`, `latest`

---

## Step 8 — Pull and Run from Docker Hub

Delete local copies to simulate pulling on a different machine:

```bash
docker rmi ${IMAGE}:v1.0.0 ${IMAGE}:git-${GIT_SHA} ${IMAGE}:latest ${IMAGE}:local
docker images ${IMAGE}   # should show nothing
```

Pull from Docker Hub:

```bash
docker pull ${DOCKERHUB_USERNAME}/registry-lab:v1.0.0
docker run --rm -p 3000:3000 ${DOCKERHUB_USERNAME}/registry-lab:v1.0.0
curl http://localhost:3000
```

---

## Step 9 — Make a Change and Release v1.0.1

Simulate a patch release:

```bash
# Edit src/server.js — change the message to "Registry Lab App v2"
sed -i "s/Registry Lab App/Registry Lab App - updated/" src/server.js

git add .
git commit -m "Update message"

GIT_SHA=$(git rev-parse --short HEAD)
APP_VERSION="v1.0.1"

docker build \
  --build-arg APP_VERSION=${APP_VERSION} \
  --build-arg GIT_SHA=${GIT_SHA} \
  -t ${DOCKERHUB_USERNAME}/registry-lab:local .

docker tag ${DOCKERHUB_USERNAME}/registry-lab:local ${DOCKERHUB_USERNAME}/registry-lab:v1.0.1
docker tag ${DOCKERHUB_USERNAME}/registry-lab:local ${DOCKERHUB_USERNAME}/registry-lab:git-${GIT_SHA}
docker tag ${DOCKERHUB_USERNAME}/registry-lab:local ${DOCKERHUB_USERNAME}/registry-lab:latest

docker push ${DOCKERHUB_USERNAME}/registry-lab:v1.0.1
docker push ${DOCKERHUB_USERNAME}/registry-lab:git-${GIT_SHA}
docker push ${DOCKERHUB_USERNAME}/registry-lab:latest
```

Now verify both versions are on Docker Hub:
- `v1.0.0` — old image, unchanged
- `v1.0.1` — new image
- `latest` — now points to v1.0.1

Pull the old version to demonstrate rollback:

```bash
docker run --rm ${DOCKERHUB_USERNAME}/registry-lab:v1.0.0
# Shows original message

docker run --rm ${DOCKERHUB_USERNAME}/registry-lab:v1.0.1
# Shows updated message
```

This is why explicit version tags exist — `latest` alone cannot give you rollback.

---

# Assignment 14 — Automate Image Tagging with a Shell Script

## Goal

Write a shell script `build-and-push.sh` that:

1. Reads `DOCKERHUB_USERNAME` and `APP_VERSION` from environment variables (or `.env`).
2. Gets the current git commit SHA automatically.
3. Builds the Docker image with `APP_VERSION` and `GIT_SHA` baked in as build args.
4. Tags the image with `:v<version>`, `:git-<sha>`, and `:latest`.
5. Pushes all three tags to Docker Hub.
6. Prints a clear summary of what was built and pushed.
7. Exits with an error if any step fails.

---

## Solution — `build-and-push.sh`

```bash
#!/usr/bin/env bash
# =============================================================================
# build-and-push.sh
# Builds a Docker image, applies version + git SHA tags, pushes to Docker Hub.
#
# Usage:
#   APP_VERSION=v1.2.0 DOCKERHUB_USERNAME=yourname ./build-and-push.sh
#
# Required environment variables:
#   DOCKERHUB_USERNAME  — your Docker Hub username
#   APP_VERSION         — semantic version, e.g. v1.2.0
#
# Optional:
#   IMAGE_NAME          — repo name (default: registry-lab)
#   DOCKERFILE          — path to Dockerfile (default: ./Dockerfile)
#   BUILD_CONTEXT       — build context path (default: .)
# =============================================================================

set -euo pipefail
# set -e  → exit immediately if any command fails
# set -u  → treat unset variables as errors
# set -o pipefail → catch failures inside pipelines (e.g. cmd1 | cmd2)

# ── Color output ─────────────────────────────────────────────────────────────
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
BOLD='\033[1m'
RESET='\033[0m'

info()    { echo -e "${BLUE}[INFO]${RESET}  $*"; }
success() { echo -e "${GREEN}[OK]${RESET}    $*"; }
warn()    { echo -e "${YELLOW}[WARN]${RESET}  $*"; }
error()   { echo -e "${RED}[ERROR]${RESET} $*" >&2; exit 1; }

# ── Load .env if it exists ────────────────────────────────────────────────────
if [[ -f .env ]]; then
  info "Loading .env file"
  set -a
  # shellcheck source=/dev/null
  source .env
  set +a
fi

# ── Validate required variables ───────────────────────────────────────────────
[[ -z "${DOCKERHUB_USERNAME:-}" ]] && error "DOCKERHUB_USERNAME is not set."
[[ -z "${APP_VERSION:-}" ]]        && error "APP_VERSION is not set. Example: APP_VERSION=v1.0.0"

# Validate version format: must start with 'v' followed by semver
if ! echo "${APP_VERSION}" | grep -qE '^v[0-9]+\.[0-9]+\.[0-9]+$'; then
  error "APP_VERSION '${APP_VERSION}' is not valid semver. Use format: v1.2.3"
fi

# ── Configuration ─────────────────────────────────────────────────────────────
IMAGE_NAME="${IMAGE_NAME:three-tier-crud-app-backend}"
DOCKERFILE="${DOCKERFILE:-backend/Dockerfile}"
BUILD_CONTEXT="${BUILD_CONTEXT:-.}"
FULL_IMAGE="${DOCKERHUB_USERNAME}/${IMAGE_NAME}"

# ── Get git SHA ───────────────────────────────────────────────────────────────
if ! git rev-parse --git-dir > /dev/null 2>&1; then
  error "Not inside a git repository. git SHA cannot be determined."
fi

GIT_SHA=$(git rev-parse --short HEAD)
GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
GIT_DIRTY=""
if ! git diff --quiet || ! git diff --cached --quiet; then
  GIT_DIRTY="-dirty"
  warn "Working directory has uncommitted changes. SHA will be marked as ${GIT_SHA}${GIT_DIRTY}"
fi

# ── Print build summary ───────────────────────────────────────────────────────
echo ""
echo -e "${BOLD}══════════════════════════════════════════════${RESET}"
echo -e "${BOLD}  Docker Build & Push                         ${RESET}"
echo -e "${BOLD}══════════════════════════════════════════════${RESET}"
echo -e "  Image      : ${FULL_IMAGE}"
echo -e "  Version    : ${APP_VERSION}"
echo -e "  Git SHA    : ${GIT_SHA}${GIT_DIRTY}"
echo -e "  Branch     : ${GIT_BRANCH}"
echo -e "  Dockerfile : ${DOCKERFILE}"
echo -e "${BOLD}══════════════════════════════════════════════${RESET}"
echo ""

# ── Build ─────────────────────────────────────────────────────────────────────
info "Building image..."

docker build \
  --file "${DOCKERFILE}" \
  --build-arg APP_VERSION="${APP_VERSION}" \
  --build-arg GIT_SHA="${GIT_SHA}${GIT_DIRTY}" \
  --tag "${FULL_IMAGE}:local" \
  "${BUILD_CONTEXT}"

success "Build complete"

# ── Tag ───────────────────────────────────────────────────────────────────────
info "Applying tags..."

TAG_VERSION="${FULL_IMAGE}:${APP_VERSION}"
TAG_SHA="${FULL_IMAGE}:git-${GIT_SHA}${GIT_DIRTY}"
TAG_LATEST="${FULL_IMAGE}:latest"

docker tag "${FULL_IMAGE}:local" "${TAG_VERSION}"
docker tag "${FULL_IMAGE}:local" "${TAG_SHA}"
docker tag "${FULL_IMAGE}:local" "${TAG_LATEST}"

echo "  → ${TAG_VERSION}"
echo "  → ${TAG_SHA}"
echo "  → ${TAG_LATEST}"
success "Tags applied"

# ── Push ─────────────────────────────────────────────────────────────────────
info "Pushing to Docker Hub..."

docker push "${TAG_VERSION}"
docker push "${TAG_SHA}"
docker push "${TAG_LATEST}"

success "All tags pushed"

# ── Final summary ─────────────────────────────────────────────────────────────
echo ""
echo -e "${BOLD}══════════════════════════════════════════════${RESET}"
echo -e "${GREEN}${BOLD}  Build & Push Complete ✓${RESET}"
echo -e "${BOLD}══════════════════════════════════════════════${RESET}"
echo -e "  Pushed tags:"
echo -e "    docker pull ${TAG_VERSION}"
echo -e "    docker pull ${TAG_SHA}"
echo -e "    docker pull ${TAG_LATEST}"
echo -e ""
echo -e "  Docker Hub:"
echo -e "    https://hub.docker.com/r/${DOCKERHUB_USERNAME}/${IMAGE_NAME}/tags"
echo -e "${BOLD}══════════════════════════════════════════════${RESET}"
echo ""
```

---

## Usage

Make the script executable:

```bash
chmod +x build-and-push.sh
```

Run with environment variables inline:

```bash
DOCKERHUB_USERNAME=yourusername APP_VERSION=v1.0.0 ./build-and-push.sh
```

Or set them in your `.env` file:

```bash
# .env
DOCKERHUB_USERNAME=yourusername
APP_VERSION=v1.0.0
IMAGE_NAME=registry-lab
```

Then run:

```bash
./build-and-push.sh
```

Override just one variable:

```bash
APP_VERSION=v1.1.0 ./build-and-push.sh
```

---

## Sample Output

```text
══════════════════════════════════════════════
  Docker Build & Push
══════════════════════════════════════════════
  Image      : yourname/registry-lab
  Version    : v1.0.0
  Git SHA    : a3f9c2d
  Branch     : main
  Dockerfile : ./Dockerfile
══════════════════════════════════════════════

[INFO]  Building image...
[OK]    Build complete
[INFO]  Applying tags...
  → yourname/registry-lab:v1.0.0
  → yourname/registry-lab:git-a3f9c2d
  → yourname/registry-lab:latest
[OK]    Tags applied
[INFO]  Pushing to Docker Hub...
[OK]    All tags pushed

══════════════════════════════════════════════
  Build & Push Complete ✓
══════════════════════════════════════════════
  Pushed tags:
    docker pull yourname/registry-lab:v1.0.0
    docker pull yourname/registry-lab:git-a3f9c2d
    docker pull yourname/registry-lab:latest

  Docker Hub:
    https://hub.docker.com/r/yourname/registry-lab/tags
══════════════════════════════════════════════
```

---

## Error Handling in the Script

| Scenario                         | What happens                                       |
|----------------------------------|----------------------------------------------------|
| `DOCKERHUB_USERNAME` not set     | Script exits with `[ERROR] DOCKERHUB_USERNAME is not set.` |
| `APP_VERSION` missing or wrong format | Exits with format validation error            |
| Not inside a git repo            | Exits: `git SHA cannot be determined`              |
| Uncommitted changes              | Warning printed, SHA gets `-dirty` suffix          |
| `docker build` fails             | `set -e` causes immediate exit                     |
| `docker push` fails (not logged in) | `set -e` exits; fix: run `docker login` first  |

---

## Assignment Checklist

- [x] Script file named `build-and-push.sh`, executable (`chmod +x`)
- [x] `set -euo pipefail` at the top
- [ ] `DOCKERHUB_USERNAME` and `APP_VERSION` validated — exits with clear error if missing
- [ ] Semver format validated with regex
- [ ] Git SHA captured with `git rev-parse --short HEAD`
- [ ] Script exits with error if not run inside a git repository
- [ ] Image built with `--build-arg APP_VERSION` and `--build-arg GIT_SHA`
- [ ] Three tags applied: `:v<version>`, `:git-<sha>`, `:latest`
- [ ] All three tags pushed
- [ ] `curl http://localhost:3000` on the running image shows correct `version` and `git_sha`
- [ ] Docker Hub tags page shows all three tags after push
- [ ] Script loads `.env` if present
- [ ] Clean summary printed at end showing exact `docker pull` commands

---

# Quick Reference

## Image Naming

```text
[registry/][username/]image[:tag]

nginx                                         → docker.io/library/nginx:latest
yourname/myapp:v1.0.0                         → docker.io/yourname/myapp:v1.0.0
ghcr.io/yourname/myapp:main                   → GitHub Container Registry
123.dkr.ecr.region.amazonaws.com/myapp:latest → AWS ECR
```

## Tag Commands

```bash
docker tag SOURCE_IMAGE TARGET_IMAGE        # create a new tag
docker images REPO                          # list all tags for a repo
docker rmi IMAGE:TAG                        # remove a local tag
```

## Registry Commands

```bash
docker login                                     # login to Docker Hub
docker login -u USER --password-stdin            # non-interactive login
docker logout                                    # logout
docker push USERNAME/REPO:TAG                    # push image
docker pull USERNAME/REPO:TAG                    # pull image
```

## Tagging Strategy Summary

| Tag             | Meaning                        | Use for              |
|-----------------|--------------------------------|----------------------|
| `:latest`       | Most recent push               | Local dev only       |
| `:v1.2.3`       | Exact semantic version         | Production deploys   |
| `:v1.2`         | Minor version track            | Auto-patch updates   |
| `:git-a3f9c2d`  | Exact commit                   | Traceability, CI/CD  |
| `:main`         | Latest build from main branch  | Staging              |
| `:staging`      | Promoted to staging            | QA environments      |

## Shell Script Best Practices

```bash
set -euo pipefail              # exit on error, unset var, pipe failure
[[ -z "${VAR:-}" ]]            # check if variable is unset or empty
git rev-parse --short HEAD     # get 7-char git SHA
source .env                    # load .env file variables
set -a; source .env; set +a    # export all variables from .env
echo "text" | docker login \
  -u USER --password-stdin     # secure non-interactive login
```
