# Day 11 — Docker Introduction
## Container Theory, Docker Architecture, Images vs Containers

---

## Learning Goals

By the end of this class, students should be able to:

- Explain the difference between containers and virtual machines.
- Describe how Linux namespaces and cgroups power containers.
- Understand Docker's daemon, client, and registry architecture.
- Explain image layers and the union filesystem.
- Use Docker Hub to pull public images.
- Manage the full container lifecycle: create, start, stop, remove.
- Run and explore an nginx container.
- Run a Node.js application inside a container.

---

# Part A — Theory

## 1. Containers vs Virtual Machines

### Virtual Machines (VMs)

A VM runs a **complete operating system** on top of a hypervisor (like VirtualBox or VMware). Every VM includes:

- A full OS kernel
- Virtual hardware (CPU, RAM, disk)
- The application and its dependencies

VMs are isolated and secure but heavy — each VM can take several GB of disk space and minutes to boot.

![Alt text](virtual_machines.png)
### Containers

A container shares the **host OS kernel** and isolates only the application and its dependencies. No full OS copy needed.

Containers are:
- Lightweight (megabytes, not gigabytes)
- Fast to start (milliseconds to seconds)
- Portable — runs the same on any machine with Docker

![Alt text](container-what-is-container.png)
### Comparison Table

| Feature          | Virtual Machine        | Container              |
|------------------|------------------------|------------------------|
| OS               | Full OS per VM         | Shares host kernel     |
| Boot time        | Minutes                | Seconds or less        |
| Disk size        | GBs                    | MBs                    |
| Isolation        | Strong (hardware-level)| Process-level          |
| Portability      | Limited                | High                   |
| Resource usage   | High                   | Low                    |

---

## 2. How Containers Work — Namespaces and cgroups

Containers are not magic. They are built on two Linux kernel features: **namespaces** and **cgroups**.

### Namespaces — Isolation

A **namespace** limits what a process can see. Each container gets its own set of namespaces so it cannot see or interfere with other containers or the host.

| Namespace | What it isolates                         |
|-----------|------------------------------------------|
| `pid`     | Process IDs — container sees its own PIDs |
| `net`     | Network interfaces, IP addresses, ports  |
| `mnt`     | Filesystem mount points                  |
| `uts`     | Hostname and domain name                 |
| `ipc`     | Inter-process communication              |
| `user`    | User and group IDs                       |

Example: Process with PID `1` inside a container is a completely different process from PID `1` on the host.

### cgroups — Resource Limits

**cgroups** (control groups) limit and track how much CPU, memory, disk I/O, and network a process can use.

Without cgroups, one container could consume all available CPU and starve others.

Example: Limit a container to 512 MB of RAM:

```bash
docker run -m 512m nginx
```

---

## 3. Docker Architecture

Docker uses a **client-server architecture**.

![Alt text](docker-architecture.gif)
### Components

| Component        | Role                                                             |
|------------------|------------------------------------------------------------------|
| Docker Client    | CLI tool (`docker`) that sends commands to the daemon           |
| Docker Daemon    | Background service (`dockerd`) that manages containers/images   |
| Docker Registry  | Remote storage for images (Docker Hub is the default public one)|

When you run `docker run nginx`:
1. The **client** sends the command to the **daemon**.
2. The daemon checks if the `nginx` image exists locally.
3. If not, the daemon **pulls** it from Docker Hub.
4. The daemon creates and starts the container.

---

## 4. Images vs Containers

### What is a Docker Image?

A Docker **image** is a read-only template that describes:
- The OS base (e.g. Ubuntu, Alpine)
- Installed packages and dependencies
- Application code
- Configuration and startup command

An image is like a **recipe** or a **class** in programming.

### What is a Container?

A **container** is a running instance of an image.

An image is like a **class**, a container is like an **object** (instance) of that class.

You can run many containers from the same image:

```bash
docker run nginx   # container 1
docker run nginx   # container 2
docker run nginx   # container 3
```

All three containers use the same nginx image but are completely isolated from each other.

---

## 5. Image Layers and the Union Filesystem

Docker images are built in **layers**. Each instruction in a Dockerfile adds a new layer on top of the previous one.

Example image layer stack for nginx:

```text
┌─────────────────────────────────┐  ← Layer 4: nginx config files
├─────────────────────────────────┤  ← Layer 3: nginx binary installed
├─────────────────────────────────┤  ← Layer 2: apt packages updated
└─────────────────────────────────┘  ← Layer 1: debian base image
```

### Why layers?

- **Caching** — If layer 1 and 2 haven't changed, Docker reuses them and only rebuilds from the changed layer.
- **Sharing** — If two images share the same base layer, Docker stores that layer only once on disk.

### Union Filesystem

Docker uses a **union filesystem** (e.g. OverlayFS) to merge multiple read-only image layers into a single unified view.

When a container runs:
- All image layers remain **read-only**.
- A thin **writable layer** is added on top for the running container.
- Any changes made inside the container (created files, modified configs) go into this writable layer.
- When the container is deleted, the writable layer is discarded. The image layers remain unchanged.

```text
┌─────────────────────────────────┐  ← Writable layer (container)
├─────────────────────────────────┤  ← Image layer 4 (read-only)
├─────────────────────────────────┤  ← Image layer 3 (read-only)
├─────────────────────────────────┤  ← Image layer 2 (read-only)
└─────────────────────────────────┘  ← Image layer 1 (read-only)
```

---

## 6. Docker Hub

**Docker Hub** (`hub.docker.com`) is the default public registry where Docker images are stored and shared.

Types of images on Docker Hub:

| Type             | Example              | Description                               |
|------------------|----------------------|-------------------------------------------|
| Official images  | `nginx`, `node`, `ubuntu` | Maintained by Docker or the software vendor |
| Verified images  | `bitnami/nginx`      | Verified publishers                       |
| Community images | `username/myapp`     | Uploaded by anyone                        |

Pull an image manually:

```bash
docker pull ubuntu:24.04
docker pull nginx:latest
docker pull node:20-alpine
```

Image naming format:

```text
[registry/][username/]image[:tag]

nginx              → official image, latest tag
nginx:1.25         → official image, version 1.25
node:20-alpine     → official node image, Alpine Linux variant
ubuntu:24.04       → official Ubuntu 24.04 image
```

---

## 7. Container Lifecycle

A container moves through these states:

![Alt text](docker_lifecycle.png)

| Command             | Description                                           |
|---------------------|-------------------------------------------------------|
| `docker create`     | Create a container but do not start it                |
| `docker start`      | Start a created or stopped container                  |
| `docker run`        | Create and start a container in one step              |
| `docker stop`       | Gracefully stop a running container (sends SIGTERM)   |
| `docker kill`       | Force-stop a container immediately (sends SIGKILL)    |
| `docker rm`         | Remove a stopped container                            |
| `docker ps`         | List running containers                               |
| `docker ps -a`      | List all containers including stopped ones            |

---

# Part B — Practical Lab

## Lab Setup

Verify Docker is installed:

```bash
docker --version
docker info
```

---

## Lab 1 — Pull and Inspect an Image

### Pull the nginx image

```bash
docker pull nginx:latest
```

Watch Docker download each **layer** separately. Each line is one layer.

### List local images

```bash
docker images
```

Output columns explained:

```text
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
nginx        latest    a72860cb95fd   2 weeks ago    188MB
```

### Inspect image layers

```bash
docker image inspect nginx:latest
```

Look for the `Layers` section — you will see a list of SHA256 digests, one per layer.

View layer history:

```bash
docker image history nginx:latest
```

Each row is one layer, showing the command that created it and its size.

---

## Lab 2 — Run Your First Container

### Run nginx in the foreground

```bash
docker run nginx
```

Press `Ctrl+C` to stop it.

### Run nginx in detached mode (background)

```bash
docker run -d --name my-nginx -p 8080:80 nginx
```

Flags explained:

| Flag              | Meaning                                              |
|-------------------|------------------------------------------------------|
| `-d`              | Detached mode — run in background                    |
| `--name my-nginx` | Give the container a name                            |
| `-p 8080:80`      | Map host port 8080 to container port 80              |

### Test it

```bash
curl http://localhost:8080
```

Or open `http://localhost:8080` in a browser. You should see the nginx welcome page.

### Check running containers

```bash
docker ps
```

---

## Lab 3 — Exec Into a Running Container

```bash
docker exec -it my-nginx bash
```

Flags:

| Flag | Meaning                             |
|------|-------------------------------------|
| `-i` | Interactive — keep stdin open        |
| `-t` | Allocate a pseudo-TTY (terminal)     |

Now you are inside the container. Explore:

```bash
whoami
hostname
cat /etc/os-release
ls /etc/nginx/
cat /etc/nginx/nginx.conf
ls /usr/share/nginx/html/
cat /usr/share/nginx/html/index.html
```

Check the nginx process:

```bash
ps aux
```

You will only see the nginx process — no other system processes from the host are visible (namespace isolation).

Exit the container:

```bash
exit
```

---

## Lab 4 — Container Logs and Inspect

View container logs:

```bash
docker logs my-nginx
```

Follow logs in real time (like `tail -f`):

```bash
docker logs -f my-nginx
```

Press `Ctrl+C` to stop following.

Inspect full container details:

```bash
docker inspect my-nginx
```

Look for:
- `NetworkSettings.IPAddress` — container's internal IP
- `Mounts` — any volumes attached
- `State` — running status and start time
- `Config.Image` — which image it came from

---

## Lab 5 — Container Lifecycle Practice

### Stop the container

```bash
docker stop my-nginx
```

Check status:

```bash
docker ps       # not shown — it's stopped
docker ps -a    # shown with status "Exited"
```

### Start it again

```bash
docker start my-nginx
docker ps
```

### Create without starting

```bash
docker create --name test-nginx -p 9090:80 nginx
docker ps -a    # status: Created
docker start test-nginx
docker ps       # now running on port 9090
```

### Stop and remove

```bash
docker stop my-nginx test-nginx
docker rm my-nginx test-nginx
docker ps -a    # both gone
```

### Remove an image

```bash
docker rmi nginx:latest
```

> An image cannot be removed while a container (even stopped) is using it. Remove the container first.

---

## Lab 6 — Useful Cleanup Commands

Remove all stopped containers:

```bash
docker container prune
```

Remove all unused images:

```bash
docker image prune
```

Remove everything unused (containers, images, networks, build cache):

```bash
docker system prune
```

Check disk usage:

```bash
docker system df
```

---

# Assignment 11 — Run a Node.js App in a Container

## Goal

Run a simple Node.js HTTP server inside a Docker container using the official `node` image — without writing a Dockerfile.

## Step 1 — Create the Application File

On your host machine, create a project directory:

```bash
mkdir ~/node-docker-app
cd ~/node-docker-app
```

Create the application file `app.js`:

```javascript
const http = require('http');

const PORT = 3000;

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end(`Hello from Node.js inside Docker!\nHostname: ${require('os').hostname()}\n`);
});

server.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

## Step 2 — Run the App Inside a Node Container

```bash
docker run -it --rm \
  --name node-app \
  -p 3000:3000 \
  -v $(pwd):/app \
  -w /app \
  node:20-alpine \
  node app.js
```

Flags explained:

| Flag              | Meaning                                                   |
|-------------------|-----------------------------------------------------------|
| `-it`             | Interactive terminal                                      |
| `--rm`            | Automatically remove the container when it stops         |
| `--name node-app` | Container name                                            |
| `-p 3000:3000`    | Map host port 3000 to container port 3000                 |
| `-v $(pwd):/app`  | Mount current directory into `/app` inside the container  |
| `-w /app`         | Set working directory inside the container to `/app`      |
| `node:20-alpine`  | Official Node.js 20 image (Alpine = small size)           |
| `node app.js`     | Command to run inside the container                       |

## Step 3 — Test the App

Open a new terminal and run:

```bash
curl http://localhost:3000
```

Expected output:

```text
Hello from Node.js inside Docker!
Hostname: a3f9c2d1b042
```

The hostname is the container ID — proof that you are running inside a container.

Or open `http://localhost:3000` in a browser.

## Step 4 — Verify It Is Running in a Container

Open another terminal and check:

```bash
docker ps
```

You should see `node-app` listed.

Exec into it while it runs:

```bash
docker exec -it node-app sh
```

```bash
node --version
cat /etc/os-release   # Alpine Linux
ls /app               # your app.js is here
exit
```

## Step 5 — Stop the App

Press `Ctrl+C` in the terminal running the node server.

Because `--rm` was used, the container is automatically removed.

Verify:

```bash
docker ps -a    # node-app is gone
```

---

## Assignment Checklist

Before submitting, verify you can answer these questions:

- [ ] What is the difference between `docker run` and `docker start`?
- [ ] What does the `-v` flag do in the run command above?
- [ ] What does `-w /app` do?
- [ ] Why is `node:20-alpine` smaller than `node:20`?
- [ ] What happens to the container when you press `Ctrl+C` and why?
- [ ] What does the hostname in the curl response represent?

---

## Quick Reference Card

### Image Commands

| Command                        | Description                     |
|--------------------------------|---------------------------------|
| `docker pull IMAGE`            | Download an image               |
| `docker images`                | List local images               |
| `docker image history IMAGE`   | Show image layers               |
| `docker image inspect IMAGE`   | Full image metadata             |
| `docker rmi IMAGE`             | Remove an image                 |

### Container Commands

| Command                        | Description                           |
|--------------------------------|---------------------------------------|
| `docker run IMAGE`             | Create and start a container          |
| `docker run -d IMAGE`          | Run in background (detached)          |
| `docker run --rm IMAGE`        | Auto-remove on exit                   |
| `docker run -p HOST:CONT`      | Map a port                            |
| `docker run -v HOST:CONT`      | Mount a volume                        |
| `docker run -w /path`          | Set working directory                 |
| `docker ps`                    | List running containers               |
| `docker ps -a`                 | List all containers                   |
| `docker stop NAME`             | Gracefully stop a container           |
| `docker rm NAME`               | Remove a stopped container            |
| `docker exec -it NAME bash`    | Open shell in running container       |
| `docker logs NAME`             | View container output                 |
| `docker inspect NAME`          | Full container metadata               |

### Cleanup Commands

| Command                    | Description                          |
|----------------------------|--------------------------------------|
| `docker container prune`   | Remove all stopped containers        |
| `docker image prune`       | Remove unused images                 |
| `docker system prune`      | Remove all unused resources          |
| `docker system df`         | Show Docker disk usage               |
