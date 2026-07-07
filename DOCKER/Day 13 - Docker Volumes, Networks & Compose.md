# Day 13 — Docker Volumes, Networks & Compose
## Persistent Storage, Container Networking, Multi-Container Apps

---

## Learning Goals

By the end of this class, students should be able to:

- Explain the difference between named volumes, bind mounts, and tmpfs.
- Understand Docker network drivers: bridge, host, and none.
- Use Docker DNS to communicate between containers by name.
- Write a `docker-compose.yml` with multiple services, volumes, networks, and `depends_on`.
- Inject environment variables into containers from Compose.
- Run a Node.js + MongoDB stack with persistent storage.
- Add Redis to a Compose file for session caching.

---

# Part A — Theory

## 1. The Problem with Container Storage

Containers have a writable layer on top of their read-only image layers. Any files written inside a running container — database records, uploaded files, logs — live in this writable layer.

When the container is removed, that writable layer is **permanently deleted**.

```text
docker run mongo     ← database writes go to writable layer
docker rm mongo      ← all data is gone forever
```

Docker solves this with three storage options:

---

## 2. Named Volumes vs Bind Mounts vs tmpfs

### Named Volumes

A **named volume** is storage managed entirely by Docker. Docker creates and maintains the volume directory on the host, and you refer to it by name.

```bash
docker volume create mydata
docker run -v mydata:/data/db mongo
```

The actual data lives somewhere like `/var/lib/docker/volumes/mydata/_data` on the host, but you never need to know or care about that path. Docker manages it.

```text
Host (Docker managed)                  Container
/var/lib/docker/volumes/mydata/_data ←→ /data/db
```

**When to use:** Databases, persistent application state, anything that needs to survive container removal. This is the recommended approach for production data.

Characteristics:
- Managed by Docker
- Portable between hosts with `docker volume`
- Survive container removal
- Can be shared between multiple containers
- Work the same on Linux, Mac, and Windows

---

### Bind Mounts

A **bind mount** maps a specific directory or file from the host filesystem directly into the container.

```bash
docker run -v /home/student/myapp:/app node:20-alpine node server.js
# or using the newer --mount syntax:
docker run --mount type=bind,source=/home/student/myapp,target=/app node:20-alpine node server.js
```

```text
Host                             Container
/home/student/myapp         ←→  /app
```

**When to use:** Local development — you want the container to immediately see code changes without rebuilding. Not recommended for production databases.

Characteristics:
- Host path must exist
- Host can read and write files — and so can the container
- Changes on either side are immediately visible to the other
- Tied to host filesystem structure (less portable)

---

### tmpfs Mounts

A **tmpfs** mount stores data in the host's **memory**, not on disk. Data is lost when the container stops.

```bash
docker run --tmpfs /tmp myapp
# or:
docker run --mount type=tmpfs,target=/tmp myapp
```

**When to use:** Sensitive temporary data (tokens, secrets in temp files), performance-critical scratch space, cache directories that should never be persisted.

---

### Comparison Table

| Feature           | Named Volume | Bind Mount               | tmpfs             |
| ----------------- | ------------ | ------------------------ | ----------------- |
| Managed by        | Docker       | Host OS                  | Memory only       |
| Persists after rm | Yes          | Yes (it's your host dir) | No                |
| Use in production | Recommended  | Avoid for databases      | Secrets / scratch |
| Dev workflow      | Possible     | Best for live reload     | Rarely            |
| Portability       | High         | Low (host path required) | N/A               |
| Performance       | Good         | Good on Linux            | Fastest           |

---

## 3. Volume Commands Reference

```bash
docker volume create mydata          # create a named volume
docker volume ls                     # list all volumes
docker volume inspect mydata         # show volume details and mount path
docker volume rm mydata              # remove a volume
docker volume prune                  # remove all unused volumes
```

---

## 4. Docker Networking

Every container is isolated by default. To allow containers to communicate, Docker uses **networks**.

### Network Drivers

#### Bridge (default)

The **bridge** driver creates a private internal network on the host. Containers on the same bridge network can communicate with each other. Containers are isolated from containers on other networks.

```text
Host
├── bridge network: my-network
│   ├── container: app      (IP: 172.18.0.2)
│   └── container: mongo    (IP: 172.18.0.3)
└── bridge network: default
    └── container: other    (IP: 172.17.0.2)
```

`app` can reach `mongo`. `app` cannot reach `other` (different network).

**User-defined bridge networks** (created with `docker network create`) are strongly preferred over the default bridge because they provide DNS resolution by container name — explained in the next section.

#### Host

The **host** driver removes network isolation. The container shares the host's network stack directly, as if the process were running directly on the host.

```bash
docker run --network host nginx
# nginx is accessible on the host's port 80 directly, no -p needed
```

Use cases: High-performance networking, situations where port mapping overhead matters, tools that need to monitor host network interfaces.

Security note: A container on host networking can reach any service on the host's loopback interface, which reduces isolation.

#### None

The **none** driver disables all networking for the container. It has no network interface except a loopback.

```bash
docker run --network none myapp
```

Use cases: Batch processing jobs that read input files and write output files with no network access needed, maximum security isolation.

---

## 5. Docker DNS — Container-to-Container Communication

When you create a **user-defined bridge network**, Docker runs an embedded DNS server. Containers on the same user-defined network can reach each other using their **container name** (or service name in Compose) as a hostname.

```bash
# Create a user-defined network
docker network create mynet

# Run MongoDB — its hostname on the network is "mongo"
docker run -d --name mongo --network mynet mongo:7

# Run a Node app — it can connect to MongoDB using "mongo" as the host
docker run -d --name app --network mynet \
  -e MONGO_URL=mongodb://mongo:27017/mydb \
  myapp
```

Inside `app`, the connection string `mongodb://mongo:27017/mydb` works because Docker's DNS resolves `mongo` to the container's internal IP automatically.

> **Important:** This DNS resolution only works on **user-defined** networks, not the default `docker0` bridge. Always create a named network.

```text
Container: app                  Container: mongo
MONGO_URL=mongodb://mongo:27017 ──DNS──► 172.18.0.3:27017
```

---

## 6. Docker Compose

Managing multiple containers with `docker run` commands becomes unwieldy:

```bash
# What you'd have to run manually:
docker network create mynet
docker volume create mongo-data
docker run -d --name mongo --network mynet -v mongo-data:/data/db mongo:7
docker run -d --name app --network mynet -e MONGO_URL=mongodb://mongo:27017 -p 3000:3000 myapp
```

**Docker Compose** lets you define the entire stack in one declarative `docker-compose.yml` file and manage it with simple commands.

### Core Compose Commands

```bash
docker compose up -d          # create and start all services (detached)
docker compose down           # stop and remove containers and networks
docker compose down -v        # also remove named volumes
docker compose ps             # list running services
docker compose logs -f        # follow logs for all services
docker compose logs app       # logs for a specific service
docker compose exec app bash  # exec into a running service container
docker compose build          # rebuild images
docker compose restart app    # restart a specific service
```

---

## 7. `docker-compose.yml` Structure

```yaml
version: "3.9"              # Compose file format version

services:                   # Each service is one container
  servicename:
    image: someimage        # Use a pre-built image
    build: .                # Or build from a Dockerfile
    ports:
      - "host:container"
    environment:
      KEY: value
    env_file:
      - .env
    volumes:
      - namedvolume:/path/in/container
      - ./localdir:/path/in/container
    networks:
      - mynetwork
    depends_on:
      - otherservice
    restart: unless-stopped

volumes:                    # Declare named volumes
  namedvolume:

networks:                   # Declare custom networks
  mynetwork:
    driver: bridge
```

---

## 8. Key Compose Concepts

### `depends_on`

Controls **startup order**. Docker starts dependencies before the dependent service.

```yaml
services:
  app:
    depends_on:
      - mongo
      - redis
```

> **Important limitation:** `depends_on` only waits for the container to **start**, not for the service inside it to be **ready**. MongoDB takes a few seconds to become ready after its container starts. Handle this in your application with retry logic or a health check.

### Environment Variables

Three ways to inject environment variables:

**Inline in `docker-compose.yml`:**

```yaml
environment:
  NODE_ENV: production
  PORT: 3000
  MONGO_URL: mongodb://mongo:27017/mydb
```

**From a `.env` file (automatic):**

Docker Compose automatically reads a `.env` file in the same directory and makes those variables available for substitution:

```bash
# .env file
MONGO_PASSWORD=secret123
NODE_ENV=production
```

```yaml
# docker-compose.yml
services:
  app:
    environment:
      MONGO_PASSWORD: ${MONGO_PASSWORD}
      NODE_ENV: ${NODE_ENV}
```

**Using `env_file`:**

```yaml
services:
  app:
    env_file:
      - .env          # entire file passed as environment variables
```

> **Security:** Never commit `.env` files containing real passwords to version control. Add `.env` to `.gitignore`.

### `restart` Policy

| Value             | Behavior                                              |
|-------------------|-------------------------------------------------------|
| `no`              | Never restart (default)                               |
| `always`          | Always restart, even on clean stop                    |
| `unless-stopped`  | Restart unless manually stopped — recommended         |
| `on-failure`      | Restart only if it exits with a non-zero code         |

---

# Part B — Practical Lab

## Lab Goal

Build a full Node.js + Express + MongoDB stack with Docker Compose. Use a named volume for MongoDB data persistence, a custom network for DNS-based communication, and environment variable injection.

---

## Step 1 — Project Structure

```bash
mkdir ~/compose-lab && cd ~/compose-lab
```

Create the following structure:

```text
compose-lab/
├── app/
│   ├── src/
│   │   └── server.js
│   ├── package.json
│   └── Dockerfile
├── docker-compose.yml
└── .env
```

---

## Step 2 — Create the Express + MongoDB App

### `app/package.json`

```json
{
  "name": "compose-lab",
  "version": "1.0.0",
  "main": "src/server.js",
  "dependencies": {
    "express": "^4.18.2",
    "mongoose": "^8.1.0"
  }
}
```

### `app/src/server.js`

```javascript
const express = require('express');
const mongoose = require('mongoose');

const app = express();
app.use(express.json());

const PORT = process.env.PORT || 3000;
const MONGO_URL = process.env.MONGO_URL || 'mongodb://localhost:27017/labdb';

// Simple Note model
const Note = mongoose.model('Note', new mongoose.Schema({
  text: String,
  createdAt: { type: Date, default: Date.now }
}));

// Connect to MongoDB with retry
async function connectWithRetry(retries = 10, delay = 3000) {
  for (let i = 1; i <= retries; i++) {
    try {
      await mongoose.connect(MONGO_URL);
      console.log('Connected to MongoDB');
      return;
    } catch (err) {
      console.log(`MongoDB not ready (attempt ${i}/${retries}), retrying in ${delay}ms...`);
      await new Promise(r => setTimeout(r, delay));
    }
  }
  console.error('Could not connect to MongoDB. Exiting.');
  process.exit(1);
}

app.get('/', (req, res) => {
  res.json({
    message: 'Node.js + MongoDB on Docker Compose',
    mongo: mongoose.connection.readyState === 1 ? 'connected' : 'disconnected',
    hostname: require('os').hostname()
  });
});

app.post('/notes', async (req, res) => {
  const note = await Note.create({ text: req.body.text });
  res.status(201).json(note);
});

app.get('/notes', async (req, res) => {
  const notes = await Note.find().sort({ createdAt: -1 });
  res.json(notes);
});

connectWithRetry().then(() => {
  app.listen(PORT, () => console.log(`Server on port ${PORT}`));
});
```

### `app/Dockerfile`

```dockerfile
FROM node:20-alpine AS production

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY package.json ./
RUN npm install --omit=dev && npm cache clean --force

COPY src/ ./src/

RUN chown -R appuser:appgroup /app

USER appuser

ENV NODE_ENV=production

EXPOSE 3000

CMD ["node", "src/server.js"]
```

---

## Step 3 — Create the `.env` File

```bash
# .env
MONGO_INITDB_ROOT_USERNAME=admin
MONGO_INITDB_ROOT_PASSWORD=secret123
MONGO_INITDB_DATABASE=labdb
NODE_ENV=production
PORT=3000
```

Add `.env` to `.gitignore`:

```bash
echo ".env" >> .gitignore
```

---

## Step 4 — Write the `docker-compose.yml`

```yaml
version: "3.9"

services:

  # ── Node.js Application ──────────────────────────────────
  app:
    build: ./app
    container_name: compose-app
    ports:
      - "3000:3000"
    environment:
      PORT: ${PORT}
      NODE_ENV: ${NODE_ENV}
      MONGO_URL: mongodb://${MONGO_INITDB_ROOT_USERNAME}:${MONGO_INITDB_ROOT_PASSWORD}@mongo:27017/${MONGO_INITDB_DATABASE}?authSource=admin
    depends_on:
      - mongo
    networks:
      - app-network
    restart: unless-stopped

  # ── MongoDB ──────────────────────────────────────────────
  mongo:
    image: mongo:7
    container_name: compose-mongo
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_INITDB_ROOT_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_INITDB_ROOT_PASSWORD}
      MONGO_INITDB_DATABASE: ${MONGO_INITDB_DATABASE}
    volumes:
      - mongo-data:/data/db       # named volume for persistence
    networks:
      - app-network
    restart: unless-stopped

# ── Named Volumes ─────────────────────────────────────────
volumes:
  mongo-data:

# ── Networks ──────────────────────────────────────────────
networks:
  app-network:
    driver: bridge
```

---

## Step 5 — Start the Stack

```bash
docker compose up -d
```

Watch the startup:

```bash
docker compose logs -f
```

Wait until you see:

```text
compose-app  | Connected to MongoDB
compose-app  | Server on port 3000
```

Press `Ctrl+C` to stop following logs.

Check running services:

```bash
docker compose ps
```

---

## Step 6 — Test the Application

Check the root endpoint:

```bash
curl http://localhost:3000
```

Create some notes:

```bash
curl -X POST http://localhost:3000/notes \
  -H "Content-Type: application/json" \
  -d '{"text": "First note — persisted in MongoDB!"}'

curl -X POST http://localhost:3000/notes \
  -H "Content-Type: application/json" \
  -d '{"text": "Second note"}'
```

Read all notes:

```bash
curl http://localhost:3000/notes
```

---

## Step 7 — Test Volume Persistence

Stop and remove the containers (but keep the volume):

```bash
docker compose down
```

Check the volume still exists:

```bash
docker volume ls | grep mongo-data
```

Start the stack again:

```bash
docker compose up -d
```

Read notes — they are still there:

```bash
curl http://localhost:3000/notes
```

Now test what happens when you destroy everything including the volume:

```bash
docker compose down -v     # -v removes named volumes too
docker compose up -d
curl http://localhost:3000/notes    # empty — data is gone
```

This demonstrates exactly why named volumes (and backups) matter in production.

---

## Step 8 — Inspect the Network

```bash
# List networks
docker network ls | grep app-network

# Inspect the network — see which containers are connected and their IPs
docker network inspect compose-lab_app-network
```

Verify DNS resolution works between containers:

```bash
docker compose exec app sh -c "ping -c 3 mongo"
```

The `app` container resolves `mongo` to the MongoDB container's IP automatically — no hardcoded IPs needed.

---

## Step 9 — Exec and Explore

Exec into the app container:

```bash
docker compose exec app sh
whoami    # appuser — non-root
exit
```

Exec into MongoDB and run a query:

```bash
docker compose exec mongo mongosh \
  -u admin -p secret123 --authenticationDatabase admin \
  labdb --eval "db.notes.find().pretty()"
```

---

# Assignment 13 — Add Redis for Session Caching

## Goal

Extend the existing `docker-compose.yml` to add a **Redis** service. Update the Node.js app to store a visit counter in Redis, demonstrating container-to-container DNS resolution across a third service.

---

## Step 1 — Install Redis Client

Update `app/package.json` to add the Redis client:

```json
{
  "name": "compose-lab",
  "version": "1.0.0",
  "main": "src/server.js",
  "dependencies": {
    "express": "^4.18.2",
    "mongoose": "^8.1.0",
    "redis": "^4.6.13"
  }
}
```

---

## Step 2 — Update the App to Use Redis

Update `app/src/server.js` to add Redis-backed visit counter and session cache:

```javascript
const express = require('express');
const mongoose = require('mongoose');
const { createClient } = require('redis');

const app = express();
app.use(express.json());

const PORT = process.env.PORT || 3000;
const MONGO_URL = process.env.MONGO_URL || 'mongodb://localhost:27017/labdb';
const REDIS_URL = process.env.REDIS_URL || 'redis://localhost:6379';

// MongoDB Note model
const Note = mongoose.model('Note', new mongoose.Schema({
  text: String,
  createdAt: { type: Date, default: Date.now }
}));

// Redis client
const redisClient = createClient({ url: REDIS_URL });

redisClient.on('error', err => console.error('Redis error:', err));
redisClient.on('connect', () => console.log('Connected to Redis'));

// Connect with retry
async function connectWithRetry(name, connectFn, retries = 10, delay = 3000) {
  for (let i = 1; i <= retries; i++) {
    try {
      await connectFn();
      return;
    } catch (err) {
      console.log(`${name} not ready (attempt ${i}/${retries}), retrying...`);
      await new Promise(r => setTimeout(r, delay));
    }
  }
  console.error(`Could not connect to ${name}. Exiting.`);
  process.exit(1);
}

// Routes
app.get('/', async (req, res) => {
  // Increment visit counter in Redis
  const visits = await redisClient.incr('visit_count');

  res.json({
    message: 'Node.js + MongoDB + Redis on Docker Compose',
    mongo: mongoose.connection.readyState === 1 ? 'connected' : 'disconnected',
    redis: redisClient.isReady ? 'connected' : 'disconnected',
    total_visits: visits,
    hostname: require('os').hostname()
  });
});

app.post('/notes', async (req, res) => {
  const note = await Note.create({ text: req.body.text });
  // Invalidate cached notes list in Redis
  await redisClient.del('notes_cache');
  res.status(201).json(note);
});

app.get('/notes', async (req, res) => {
  // Check Redis cache first
  const cached = await redisClient.get('notes_cache');
  if (cached) {
    return res.json({ source: 'redis-cache', data: JSON.parse(cached) });
  }

  // Cache miss — fetch from MongoDB
  const notes = await Note.find().sort({ createdAt: -1 });

  // Store in Redis with 60-second TTL
  await redisClient.setEx('notes_cache', 60, JSON.stringify(notes));

  res.json({ source: 'mongodb', data: notes });
});

app.get('/cache/clear', async (req, res) => {
  await redisClient.flushAll();
  res.json({ message: 'Redis cache cleared' });
});

// Start everything
async function start() {
  await connectWithRetry('MongoDB', () => mongoose.connect(MONGO_URL));
  await connectWithRetry('Redis', () => redisClient.connect());
  app.listen(PORT, () => console.log(`Server on port ${PORT}`));
}

start();
```

---

## Step 3 — Update `.env`

```bash
# .env
MONGO_INITDB_ROOT_USERNAME=admin
MONGO_INITDB_ROOT_PASSWORD=secret123
MONGO_INITDB_DATABASE=labdb
NODE_ENV=production
PORT=3000
REDIS_URL=redis://redis:6379
```

---

## Step 4 — Update `docker-compose.yml`

```yaml
version: "3.9"

services:

  # ── Node.js Application ──────────────────────────────────
  app:
    build: ./app
    container_name: compose-app
    ports:
      - "3000:3000"
    environment:
      PORT: ${PORT}
      NODE_ENV: ${NODE_ENV}
      MONGO_URL: mongodb://${MONGO_INITDB_ROOT_USERNAME}:${MONGO_INITDB_ROOT_PASSWORD}@mongo:27017/${MONGO_INITDB_DATABASE}?authSource=admin
      REDIS_URL: ${REDIS_URL}
    depends_on:
      - mongo
      - redis
    networks:
      - app-network
    restart: unless-stopped

  # ── MongoDB ──────────────────────────────────────────────
  mongo:
    image: mongo:7
    container_name: compose-mongo
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_INITDB_ROOT_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_INITDB_ROOT_PASSWORD}
      MONGO_INITDB_DATABASE: ${MONGO_INITDB_DATABASE}
    volumes:
      - mongo-data:/data/db
    networks:
      - app-network
    restart: unless-stopped

  # ── Redis ─────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: compose-redis
    volumes:
      - redis-data:/data           # persist Redis data across restarts
    networks:
      - app-network
    restart: unless-stopped
    command: redis-server --appendonly yes   # enable AOF persistence

# ── Named Volumes ─────────────────────────────────────────
volumes:
  mongo-data:
  redis-data:

# ── Networks ──────────────────────────────────────────────
networks:
  app-network:
    driver: bridge
```

---

## Step 5 — Rebuild and Start

```bash
docker compose down
docker compose up -d --build
docker compose logs -f
```

Wait for:

```text
compose-app  | Connected to MongoDB
compose-app  | Connected to Redis
compose-app  | Server on port 3000
```

---

## Step 6 — Test Redis Integration

Check visit counter (increments on each call):

```bash
curl http://localhost:3000
curl http://localhost:3000
curl http://localhost:3000
```

Each response shows an incrementing `total_visits` value stored in Redis.

Test cache behavior:

```bash
# Create a note (clears cache)
curl -X POST http://localhost:3000/notes \
  -H "Content-Type: application/json" \
  -d '{"text": "Testing Redis cache"}'

# First read — fetches from MongoDB, caches in Redis
curl http://localhost:3000/notes

# Second read — served from Redis cache (notice "source": "redis-cache")
curl http://localhost:3000/notes
```

Connect directly to Redis and inspect:

```bash
docker compose exec redis redis-cli

# Inside redis-cli:
KEYS *
GET visit_count
GET notes_cache
TTL notes_cache
EXIT
```

Verify DNS — Redis is reachable by name from the app container:

```bash
docker compose exec app sh -c "ping -c 3 redis"
```

---

## Assignment Checklist

- [ ] Redis service added to `docker-compose.yml` using `redis:7-alpine`
- [ ] `redis-data` named volume declared and mounted at `/data`
- [ ] Redis on the same `app-network` as `app` and `mongo`
- [ ] `app` service has `redis` in `depends_on`
- [ ] `REDIS_URL=redis://redis:6379` in `.env` and passed to app via `environment`
- [ ] `GET /` returns `total_visits` that increments on each request
- [ ] `GET /notes` returns `"source": "redis-cache"` on the second call
- [ ] `POST /notes` clears the cache so next `GET /notes` fetches fresh data from MongoDB
- [ ] `docker compose exec redis redis-cli KEYS *` shows `visit_count` and `notes_cache`
- [ ] `docker compose ps` shows all three services as running
- [ ] Ping from app to redis works: `docker compose exec app ping -c 3 redis`

---

# Quick Reference

## Storage

| Type         | Command example                                  | Persists | Use case                    |
|--------------|--------------------------------------------------|----------|-----------------------------|
| Named volume | `-v mydata:/data/db`                             | Yes      | Databases, production data  |
| Bind mount   | `-v ./app:/app`                                  | Yes      | Local development           |
| tmpfs        | `--tmpfs /tmp`                                   | No       | Secrets, scratch space      |

## Network Drivers

| Driver   | Isolation     | DNS by name | Use case                        |
|----------|---------------|-------------|---------------------------------|
| `bridge` | Yes           | User-defined only | Default for most apps    |
| `host`   | No            | N/A         | High-performance, monitoring    |
| `none`   | Full          | N/A         | Batch jobs, maximum isolation   |

## Compose Commands

| Command                         | Description                              |
|---------------------------------|------------------------------------------|
| `docker compose up -d`          | Start all services in background         |
| `docker compose up -d --build`  | Rebuild images then start                |
| `docker compose down`           | Stop and remove containers + networks    |
| `docker compose down -v`        | Also remove named volumes                |
| `docker compose ps`             | List service status                      |
| `docker compose logs -f`        | Follow all service logs                  |
| `docker compose logs -f app`    | Follow logs for one service              |
| `docker compose exec app sh`    | Open shell in running service            |
| `docker compose restart app`    | Restart one service                      |
| `docker compose build`          | Rebuild all images                       |
