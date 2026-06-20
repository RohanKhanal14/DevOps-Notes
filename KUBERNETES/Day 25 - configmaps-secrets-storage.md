# Day 25 — ConfigMaps, Secrets & Persistent Storage

> Configuration management, secret handling, PVs and PVCs

---

## Theory

### ConfigMap: Env Vars and File Mounts

A **ConfigMap** decouples configuration from container images. Instead of baking environment-specific values (ports, URLs, feature flags) into the image, you store them in a ConfigMap and inject them at runtime. The same image runs in dev, staging, and production — only the ConfigMap changes.

```
Without ConfigMap:            With ConfigMap:
  image:app-dev:1.0           image:app:1.0  (one image for all envs)
  image:app-staging:1.0         ↕ ConfigMap-dev
  image:app-prod:1.0            ↕ ConfigMap-staging
                                ↕ ConfigMap-prod
```

ConfigMaps hold **non-sensitive** data — plain text, URLs, feature flags, configuration files. Never put passwords or tokens in a ConfigMap.

#### Creating a ConfigMap

**Method 1 — Declarative YAML (recommended):**

```yaml
# app-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # Simple key-value pairs → used as env vars
  PORT: "3000"
  NODE_ENV: "production"
  DB_HOST: "postgres-svc"
  DB_PORT: "5432"
  LOG_LEVEL: "info"
  FEATURE_NEW_UI: "true"

  # Multi-line value → mounted as a file
  app.properties: |
    server.port=3000
    server.timeout=30s
    cache.ttl=300

  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://localhost:3000;
      }
    }
```

**Method 2 — Imperative commands:**

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=PORT=3000 \
  --from-literal=NODE_ENV=production \
  --from-literal=DB_HOST=postgres-svc

# From a file (key = filename, value = file contents)
kubectl create configmap nginx-config --from-file=nginx.conf

# From a directory (one key per file)
kubectl create configmap app-configs --from-file=./config/

# View the ConfigMap
kubectl get configmap app-config -o yaml
kubectl describe configmap app-config
```

---

#### Using ConfigMap as Environment Variables

**Option A — Individual keys (`valueFrom`):**

```yaml
spec:
  containers:
    - name: node-app
      image: my-app:1.0.0
      env:
        - name: PORT             # Env var name in the container
          valueFrom:
            configMapKeyRef:
              name: app-config   # ConfigMap name
              key: PORT          # Key inside the ConfigMap

        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: NODE_ENV
```

**Option B — All keys at once (`envFrom`):**

```yaml
spec:
  containers:
    - name: node-app
      image: my-app:1.0.0
      envFrom:
        - configMapRef:
            name: app-config   # Injects ALL keys as env vars
```

---

#### Using ConfigMap as a Volume (File Mount)

Mounting a ConfigMap as a volume creates files inside the container — one file per key. This is how you inject configuration files (nginx.conf, app.properties, etc.) without baking them into the image.

```yaml
spec:
  containers:
    - name: node-app
      image: my-app:1.0.0
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app/config    # Directory inside the container
          readOnly: true

  volumes:
    - name: config-volume
      configMap:
        name: app-config               # Mount all keys as files
        # OR mount specific keys:
        items:
          - key: app.properties
            path: app.properties        # Filename in the container
          - key: nginx.conf
            path: nginx/nginx.conf      # Subdirectory path
```

Result inside the container:
```
/etc/app/config/
  app.properties       ← contents of ConfigMap key "app.properties"
  nginx/nginx.conf     ← contents of ConfigMap key "nginx.conf"
```

> **ConfigMap update behavior:** When mounted as a volume, Kubernetes automatically syncs changes (within ~1 minute). When used as env vars, changes only take effect after pod restart.

---

### Secret: Base64 Encoding, Sealed Secrets Concept

A **Secret** stores sensitive data: passwords, API keys, TLS certificates, SSH keys. Secrets are stored in etcd and accessed only by pods that explicitly reference them.

#### Important: Secrets are NOT encrypted by default

Base64 is **encoding, not encryption**. Anyone with cluster access can decode a Secret:

```bash
echo "bXlwYXNzd29yZA==" | base64 --decode
# → mypassword
```

For real security, enable **etcd encryption at rest** or use external secret management (Vault, AWS Secrets Manager, Sealed Secrets).

#### Creating a Secret

**Declarative YAML — values must be base64 encoded:**

```yaml
# db-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: default
type: Opaque     # Generic secret (other types: kubernetes.io/tls, kubernetes.io/dockerconfigjson)
data:
  # Values MUST be base64 encoded
  # echo -n "mypassword" | base64  →  bXlwYXNzd29yZA==
  DB_PASSWORD: bXlwYXNzd29yZA==
  DB_USERNAME: cG9zdGdyZXM=

  # Use stringData to avoid manual base64 encoding (K8s encodes it for you)
stringData:
  DB_URL: "postgresql://postgres:mypassword@postgres-svc:5432/mydb"
```

```bash
# Encode values
echo -n "mypassword" | base64      # → bXlwYXNzd29yZA==
echo -n "postgres" | base64        # → cG9zdGdyZXM=

# Decode to verify
echo "bXlwYXNzd29yZA==" | base64 --decode
```

**Imperative (K8s handles base64 automatically):**

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=mypassword \
  --from-literal=DB_USERNAME=postgres

# TLS secret from cert files
kubectl create secret tls myapp-tls \
  --cert=tls.crt \
  --key=tls.key

# Docker registry credentials
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword
```

---

#### Using Secrets in Pods

The API is identical to ConfigMap — just replace `configMapKeyRef` with `secretKeyRef`:

```yaml
spec:
  containers:
    - name: node-app
      image: my-app:1.0.0

      # Individual secret keys as env vars
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD

        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_USERNAME

      # All secret keys as env vars
      envFrom:
        - secretRef:
            name: db-secret

      # Secret mounted as files (e.g., TLS certs, SSH keys)
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true

  volumes:
    - name: secret-volume
      secret:
        secretName: db-secret
        defaultMode: 0400    # Read-only by owner (important for SSH keys)
```

---

#### Sealed Secrets Concept

**Sealed Secrets** (by Bitnami) solves the problem of committing secrets to Git. A `SealedSecret` is encrypted with the cluster's public key — only the cluster can decrypt it.

```
Developer                   Cluster
    │                          │
    │  kubectl seal (encrypt)  │
    │◀─── public key ──────────│
    │                          │
    │─── SealedSecret (safe    │
    │    to commit to Git) ───▶│
    │                          │
    │        Sealed Secrets    │
    │        controller        │
    │        decrypts ────────▶│ Secret (in etcd)
```

```bash
# Install kubeseal CLI
brew install kubeseal

# Fetch the cluster's public key and encrypt a secret
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=mypassword \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > db-sealed-secret.yaml

# db-sealed-secret.yaml is safe to commit to Git
# The controller in the cluster decrypts it back to a Secret
kubectl apply -f db-sealed-secret.yaml
```

Other secret management approaches: **HashiCorp Vault** (external KMS), **AWS Secrets Manager + External Secrets Operator**, **SOPS** (file-level encryption).

---

### PersistentVolume and PersistentVolumeClaim

Containers are stateless by default — their filesystem is destroyed when the container restarts. For databases and stateful workloads, you need storage that outlives the pod.

```
Pod lifecycle:           Storage lifecycle:
  created  ─┐             PV created (by admin or StorageClass)
  running   │               │
  crashed   │               │  data persists ✅
  restarted │               │
  deleted  ─┘             PV still exists ✅
                            │
  new pod created ─────────▶ PVC binds to same PV
                            │
                          data available in new pod ✅
```

#### PersistentVolume (PV)

A **PersistentVolume** is a piece of storage provisioned by an administrator (or dynamically by a StorageClass). It is a cluster-level resource — not namespaced.

```yaml
# pv-manual.yaml (static provisioning — admin creates this)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongo-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce       # One node can mount read-write at a time
  persistentVolumeReclaimPolicy: Retain   # Keep data after PVC deleted
  storageClassName: standard
  hostPath:
    path: /data/mongo     # On the node (Minikube only — don't use in prod)
```

**Access modes:**

| Mode | Abbreviation | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | One node, read-write (most databases) |
| `ReadOnlyMany` | ROX | Many nodes, read-only |
| `ReadWriteMany` | RWX | Many nodes, read-write (NFS, shared storage) |
| `ReadWriteOncePod` | RWOP | One pod, read-write (K8s 1.22+) |

**Reclaim policies:**

| Policy | Behavior after PVC deleted |
|---|---|
| `Retain` | PV kept, data intact, must be manually reclaimed |
| `Delete` | PV and underlying storage deleted automatically |
| `Recycle` | Deprecated — basic scrub (`rm -rf`) then make available again |

---

#### PersistentVolumeClaim (PVC)

A **PVC** is a request for storage by a user/application. It specifies size and access mode, and Kubernetes binds it to a matching PV.

```yaml
# mongo-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: standard   # Must match PV's storageClassName
```

```
PVC "mongo-pvc" (requests 2Gi, RWO, standard)
    │
    │  Kubernetes finds a matching PV:
    │  capacity ≥ 2Gi + accessMode RWO + storageClassName standard
    │
    ▼
PV "mongo-pv" (5Gi, RWO, standard)   ← BOUND ✅
```

---

#### Using a PVC in a Pod

```yaml
spec:
  containers:
    - name: mongodb
      image: mongo:6
      volumeMounts:
        - name: mongo-storage
          mountPath: /data/db      # MongoDB data directory

  volumes:
    - name: mongo-storage
      persistentVolumeClaim:
        claimName: mongo-pvc       # Reference the PVC by name
```

---

### StorageClass and Dynamic Provisioning

**Static provisioning** requires an admin to pre-create PVs. **Dynamic provisioning** automatically creates a PV when a PVC is created, using a **StorageClass** that knows how to provision storage on the underlying platform.

```
PVC created (requests 10Gi, gp2)
    │
    ▼
StorageClass "gp2" (provisioner: ebs.csi.aws.com)
    │
    ▼
AWS EBS volume created automatically (10Gi, gp2)
    │
    ▼
PV created automatically and bound to PVC ✅
```

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com          # AWS EBS CSI driver
parameters:
  type: gp3                           # SSD volume type
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete                 # Delete EBS volume when PVC deleted
allowVolumeExpansion: true            # Allow resizing PVC after creation
volumeBindingMode: WaitForFirstConsumer  # Wait until pod is scheduled to provision
```

```bash
# List StorageClasses in the cluster
kubectl get storageclass
# NAME                 PROVISIONER                RECLAIMPOLICY   AGE
# standard (default)   k8s.io/minikube-hostpath   Delete          1h
# fast-ssd             ebs.csi.aws.com            Delete          5m

# PVC using the fast-ssd StorageClass
# (no pre-created PV needed — StorageClass creates it dynamically)
```

---

### Minikube Storage Provisioner

Minikube includes a built-in storage provisioner (`k8s.io/minikube-hostpath`) that satisfies PVCs automatically using the node's local filesystem. This is why PVCs work in Minikube without creating PVs manually.

```bash
# Verify the default StorageClass
kubectl get storageclass
# NAME                 PROVISIONER                RECLAIMPOLICY
# standard (default)   k8s.io/minikube-hostpath   Delete

# After creating a PVC, check it's bound (no manual PV needed)
kubectl get pvc
# NAME        STATUS   VOLUME                                     CAPACITY
# mongo-pvc   Bound    pvc-a1b2c3d4-...                          2Gi
```

---

## Practical Lab

### 1. Create ConfigMap — Mount as Env Vars and Volume

```yaml
# app-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  PORT: "3000"
  NODE_ENV: "production"
  DB_HOST: "postgres-svc"
  DB_PORT: "5432"
  LOG_LEVEL: "info"
  DB_URL: "postgresql://postgres-svc:5432/mydb"

  # Config file to be mounted as a volume
  config.json: |
    {
      "port": 3000,
      "logLevel": "info",
      "features": {
        "newUI": true,
        "darkMode": false
      }
    }
```

```bash
kubectl apply -f app-configmap.yaml
kubectl describe configmap app-config
```

```yaml
# node-app-with-config.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: node-app
  template:
    metadata:
      labels:
        app: node-app
    spec:
      containers:
        - name: node-app
          image: node:18-alpine
          command: ["node", "-e", "
            const http = require('http');
            const fs = require('fs');
            const port = process.env.PORT || 3000;
            const env = process.env.NODE_ENV;
            const dbUrl = process.env.DB_URL;
            console.log('PORT:', port, 'ENV:', env, 'DB_URL:', dbUrl);
            http.createServer((req, res) => {
              res.end(JSON.stringify({port, env, dbUrl}));
            }).listen(port);
          "]

          # Method 1: All ConfigMap keys as env vars
          envFrom:
            - configMapRef:
                name: app-config

          # Method 2: Specific keys (can mix with envFrom)
          env:
            - name: APP_PORT        # Override/rename a key
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: PORT

          # ConfigMap mounted as a file
          volumeMounts:
            - name: app-config-volume
              mountPath: /etc/app
              readOnly: true

          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"

      volumes:
        - name: app-config-volume
          configMap:
            name: app-config
            items:
              - key: config.json
                path: config.json
```

```bash
kubectl apply -f node-app-with-config.yaml

# Verify env vars inside the pod
kubectl exec -it deploy/node-app -- env | grep -E "PORT|NODE_ENV|DB_HOST|LOG_LEVEL"

# Verify mounted file
kubectl exec -it deploy/node-app -- cat /etc/app/config.json
```

---

### 2. Create Secret for DB Password, Reference in Deployment

```bash
# Create secret imperatively (no base64 needed)
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=supersecret123 \
  --from-literal=DB_USERNAME=postgres

# Or declaratively:
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_PASSWORD: supersecret123
  DB_USERNAME: postgres
EOF

# Verify (values are base64 encoded in output)
kubectl get secret db-secret -o yaml
kubectl describe secret db-secret   # Shows key names but NOT values
```

```yaml
# node-app-with-secret.yaml (add to the Deployment above)
spec:
  template:
    spec:
      containers:
        - name: node-app
          envFrom:
            - configMapRef:
                name: app-config
          env:
            # Inject specific secret keys
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_USERNAME
```

```bash
kubectl apply -f node-app-with-secret.yaml

# Verify secret is injected (value should be visible in env)
kubectl exec -it deploy/node-app -- env | grep DB_
# DB_PASSWORD=supersecret123
# DB_USERNAME=postgres
```

---

### 3. Attach a PVC to MongoDB Deployment

```yaml
# mongo-stack.yaml
---
# Step 1: PVC — Minikube's StorageClass creates the PV automatically
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: standard   # Minikube's default StorageClass
---
# Step 2: Secret for MongoDB credentials
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
type: Opaque
stringData:
  MONGO_INITDB_ROOT_USERNAME: admin
  MONGO_INITDB_ROOT_PASSWORD: mongosecret
---
# Step 3: MongoDB Deployment using the PVC
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  replicas: 1    # Databases are typically single-replica (use StatefulSet for HA)
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:6
          ports:
            - containerPort: 27017
          envFrom:
            - secretRef:
                name: mongo-secret
          volumeMounts:
            - name: mongo-data
              mountPath: /data/db    # MongoDB default data directory
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
      volumes:
        - name: mongo-data
          persistentVolumeClaim:
            claimName: mongo-pvc    # Reference the PVC
---
# Step 4: Service so the Node.js app can reach MongoDB
apiVersion: v1
kind: Service
metadata:
  name: mongodb-svc
spec:
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
```

```bash
kubectl apply -f mongo-stack.yaml

# Check PVC is bound (Minikube provisions it automatically)
kubectl get pvc
# NAME        STATUS   VOLUME                                     CAPACITY
# mongo-pvc   Bound    pvc-a1b2c3d4-5e6f-7890-abcd-ef1234567890   2Gi

# Verify pod is running with persistent storage
kubectl get pods -l app=mongodb
kubectl describe pod -l app=mongodb | grep -A 5 "Volumes:"

# Test data persists across pod restarts
kubectl exec -it deploy/mongodb -- mongosh -u admin -p mongosecret
# > use testdb
# > db.items.insertOne({name: "test"})
# > exit

kubectl delete pod -l app=mongodb    # Pod deleted, Deployment recreates it
kubectl exec -it deploy/mongodb -- mongosh -u admin -p mongosecret --eval \
  "db.getSiblingDB('testdb').items.find()"
# → {name: "test"} still there ✅
```

---

## Assignment 25

> Update Node.js app to read **DB_URL** and **PORT** from ConfigMap env vars.

### Step 1 — ConfigMap with DB_URL and PORT

```yaml
# node-app-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: node-app-config
  namespace: default
data:
  PORT: "3000"
  DB_URL: "mongodb://mongodb-svc:27017/mydb"
  NODE_ENV: "production"
  LOG_LEVEL: "info"
```

```bash
kubectl apply -f node-app-configmap.yaml
```

---

### Step 2 — Node.js App Reading from Env Vars

```javascript
// app.js — reads configuration from environment variables
'use strict';

const http = require('http');

// Read PORT and DB_URL from environment (injected by ConfigMap)
const PORT = process.env.PORT || 3000;
const DB_URL = process.env.DB_URL;
const NODE_ENV = process.env.NODE_ENV || 'development';

if (!DB_URL) {
  console.error('ERROR: DB_URL environment variable is not set');
  process.exit(1);
}

console.log(`Starting server: PORT=${PORT}, DB_URL=${DB_URL}, ENV=${NODE_ENV}`);

const server = http.createServer((req, res) => {
  if (req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'ok', port: PORT, env: NODE_ENV }));
    return;
  }

  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({
    message: 'Hello from Node.js',
    config: {
      port: PORT,
      dbUrl: DB_URL,
      env: NODE_ENV,
    }
  }));
});

server.listen(PORT, () => {
  console.log(`Server listening on port ${PORT}`);
});
```

---

### Step 3 — Deployment with ConfigMap and Secret

```yaml
# node-app-final.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app
  namespace: default
  annotations:
    kubernetes.io/change-cause: "v1.1.0 — read config from ConfigMap and Secret"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: node-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: node-app
    spec:
      containers:
        - name: node-app
          image: my-nodejs-app:1.1.0
          ports:
            - containerPort: 3000
              name: http

          # ── Assignment 25: Inject PORT and DB_URL from ConfigMap ──────────
          envFrom:
            - configMapRef:
                name: node-app-config    # Injects PORT, DB_URL, NODE_ENV, LOG_LEVEL
          # ─────────────────────────────────────────────────────────────────

          # Inject DB credentials from Secret (sensitive data separate from ConfigMap)
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_USERNAME

          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"

          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20

          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
```

```bash
kubectl apply -f node-app-final.yaml

# Verify PORT and DB_URL are injected correctly
kubectl exec -it deploy/node-app -- env | grep -E "^PORT=|^DB_URL="
# PORT=3000
# DB_URL=mongodb://mongodb-svc:27017/mydb

# Test the running app returns the config values
kubectl port-forward deploy/node-app 3000:3000 &
curl http://localhost:3000
# {"message":"Hello from Node.js","config":{"port":"3000","dbUrl":"mongodb://mongodb-svc:27017/mydb","env":"production"}}

# Update DB_URL without changing the image (ConfigMap hot-update for volume mounts)
kubectl edit configmap node-app-config
# Change DB_URL to: mongodb://mongodb-svc:27017/productiondb

# For env var injection, trigger a rolling restart to pick up the new values
kubectl rollout restart deployment/node-app
kubectl rollout status deployment/node-app
```

---

## Quick Reference

```
ConfigMap     → non-sensitive config (URLs, ports, flags, config files)
Secret        → sensitive data (passwords, tokens, certs) — base64 encoded
PV            → cluster-level storage resource (provisioned by admin or SC)
PVC           → namespaced storage request (binds to a PV)
StorageClass  → template for dynamic PV provisioning

Injection methods:
  envFrom: configMapRef / secretRef      → all keys as env vars
  env: valueFrom: configMapKeyRef        → individual key as env var
  env: valueFrom: secretKeyRef           → individual secret key as env var
  volumes: configMap / secret            → mounted as files

ConfigMap update:
  Volume mount  → auto-synced (~1 min)
  Env vars      → requires pod restart (kubectl rollout restart)

PVC lifecycle:
  Create PVC → SC provisions PV automatically → PVC bound → attach to pod
  Delete pod → PV and data persist (ReclaimPolicy: Retain or Delete)

Access modes:
  RWO  ReadWriteOnce  → one node, databases
  ROX  ReadOnlyMany   → many nodes, read-only
  RWX  ReadWriteMany  → many nodes, shared (NFS)

Minikube: standard StorageClass uses hostPath — no manual PV needed
```

| Object | Namespaced? | Created by | Purpose |
|---|---|---|---|
| ConfigMap | ✅ Yes | Developer | Non-sensitive app config |
| Secret | ✅ Yes | Developer | Sensitive data (passwords, certs) |
| PVC | ✅ Yes | Developer | Storage request |
| PV | ❌ No | Admin / StorageClass | Actual storage resource |
| StorageClass | ❌ No | Admin | Dynamic provisioning template |
