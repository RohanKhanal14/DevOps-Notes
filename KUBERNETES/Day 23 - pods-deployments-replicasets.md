# Day 23 — Pods, Deployments & ReplicaSets

> Core workload objects, rolling updates, rollbacks, resource limits

---

## Theory

### Pod: The Smallest Deployable Unit

A **Pod** is the atomic unit of Kubernetes. It wraps one or more containers that share the same network namespace and storage volumes. Kubernetes schedules and manages pods, not individual containers.

```
┌──────────────────────────────────────────────────┐
│                      POD                         │
│                                                  │
│  ┌─────────────────┐   ┌──────────────────────┐  │
│  │  Main Container │   │  Sidecar Container   │  │
│  │                 │   │                      │  │
│  │  my-app:1.0     │   │  log-shipper:latest  │  │
│  │  port 3000      │   │  fluentd / envoy     │  │
│  └─────────────────┘   └──────────────────────┘  │
│                                                  │
│  Shared: Network (localhost), Volumes, IPC       │
│  IP address: one per pod (not per container)     │
└──────────────────────────────────────────────────┘
```

#### Key Pod properties

- All containers in a pod share the same **IP address** and **port space** — they talk to each other via `localhost`
- All containers in a pod are **scheduled together** on the same node — they are never split across nodes
- A pod is **ephemeral** — when it dies, it is gone; Kubernetes does not resurrect it (that is the job of a ReplicaSet)
- Pods are **not** self-healing on their own; they need a controller (Deployment, StatefulSet, etc.) to restart them

#### Single-container pod (most common)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  labels:
    app: my-app
spec:
  containers:
    - name: my-app
      image: my-app:1.0.0
      ports:
        - containerPort: 3000
```

#### Multi-container pod patterns

| Pattern | Description | Example |
|---|---|---|
| **Sidecar** | Augments the main container | Log shipper, metrics exporter, service mesh proxy |
| **Ambassador** | Proxy for external communication | Local proxy to a remote database |
| **Adapter** | Normalizes output for external systems | Transforms app logs into a standard format |

```yaml
# Multi-container pod: app + log sidecar
spec:
  containers:
    - name: my-app
      image: my-app:1.0.0
      ports:
        - containerPort: 3000
      volumeMounts:
        - name: logs
          mountPath: /var/log/app

    - name: log-shipper
      image: fluent/fluent-bit:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app   # same volume, both containers read logs

  volumes:
    - name: logs
      emptyDir: {}
```

---

### ReplicaSet: Maintaining Desired Pod Count

A **ReplicaSet** ensures a specified number of identical pod replicas are running at all times. It uses a **label selector** to identify the pods it manages.

```
ReplicaSet (desired: 3)
  │
  ├── Pod A  [app=my-app]  ← Running ✅
  ├── Pod B  [app=my-app]  ← Running ✅
  └── Pod C  [app=my-app]  ← Crashed ❌
                                │
                         Controller sees: actual=2, desired=3
                                │
                         Creates Pod D  ← New pod started ✅
```

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-app-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app        # Manages pods with this label
  template:              # Pod template — used to create new pods
    metadata:
      labels:
        app: my-app      # Must match selector above
    spec:
      containers:
        - name: my-app
          image: my-app:1.0.0
```

> **In practice, you rarely write ReplicaSets directly.** Deployments create and manage ReplicaSets for you. The ReplicaSet layer exists so Deployments can do rolling updates by managing multiple ReplicaSets simultaneously.

---

### Deployment: Declarative Updates

A **Deployment** manages ReplicaSets and provides:
- Declarative pod updates (change the spec, Deployment handles the transition)
- Rolling update strategy (zero-downtime by default)
- Rollback to any previous revision

```
Deployment
  │
  ├── ReplicaSet v1  (image: my-app:1.0.0)  replicas: 0  (old, scaled down)
  └── ReplicaSet v2  (image: my-app:1.1.0)  replicas: 3  (current, active)
```

During a rolling update, the Deployment creates a new ReplicaSet and gradually shifts replicas from old to new:

```
Start:     RS-v1: 3 pods    RS-v2: 0 pods
Step 1:    RS-v1: 2 pods    RS-v2: 1 pod
Step 2:    RS-v1: 1 pod     RS-v2: 2 pods
Step 3:    RS-v1: 0 pods    RS-v2: 3 pods   ← rollout complete
```

#### Rolling update strategy parameters

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # Max pods that can be unavailable during update
                           # Can be a number or percentage (e.g. "25%")
      maxSurge: 1          # Max extra pods above desired count during update
                           # Can be a number or percentage (e.g. "25%")
```

With `replicas: 3`, `maxUnavailable: 1`, `maxSurge: 1`:
- At most 2 pods unavailable at once (never below 2 running)
- At most 4 pods total at once (3 desired + 1 surge)

#### Other strategy: Recreate

```yaml
spec:
  strategy:
    type: Recreate    # Kill ALL old pods, then start new ones
                      # Causes downtime — use only when old/new can't coexist
```

---

### kubectl rollout Commands

```bash
# Watch a rollout in progress
kubectl rollout status deployment/my-app
# Waiting for deployment "my-app" rollout to finish: 1 out of 3 new replicas updated...
# Waiting for deployment "my-app" rollout to finish: 2 out of 3 new replicas updated...
# deployment "my-app" successfully rolled out

# View rollout history (revisions)
kubectl rollout history deployment/my-app
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         kubectl set image deployment/my-app my-app=my-app:1.1.0

# View details of a specific revision
kubectl rollout history deployment/my-app --revision=2

# Rollback to the previous revision
kubectl rollout undo deployment/my-app

# Rollback to a specific revision
kubectl rollout undo deployment/my-app --to-revision=1

# Pause a rollout (stop mid-way to verify)
kubectl rollout pause deployment/my-app

# Resume a paused rollout
kubectl rollout resume deployment/my-app

# Restart all pods (rolling, without changing the image)
kubectl rollout restart deployment/my-app
```

**Adding CHANGE-CAUSE to history** (annotation trick):

```bash
kubectl annotate deployment/my-app \
  kubernetes.io/change-cause="Updated to v1.1.0: added /health endpoint"
```

Or set it in the manifest:

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "v1.1.0 — added /health endpoint"
```

---

### Resource Requests and Limits

Every container should declare how much CPU and memory it needs. This tells the scheduler how to place pods and tells the kernel when to throttle or kill them.

```
┌───────────────────────────────────────────────────────┐
│                      NODE                             │
│  Total CPU: 2 cores   Total Memory: 4Gi               │
│                                                       │
│  ┌──────────────────────────────────────────────┐     │
│  │  Pod A                                       │     │
│  │  CPU request: 500m  ←── reserved for pod     │     │
│  │  CPU limit:   1000m ←── hard ceiling         │     │
│  │  Mem request: 256Mi ←── reserved for pod     │     │
│  │  Mem limit:   512Mi ←── OOMKilled if exceeded│     │
│  └──────────────────────────────────────────────┘     │
│                                                       │
│  Scheduler sees: 1500m CPU, 3.75Gi memory still free  │
└───────────────────────────────────────────────────────┘
```

#### CPU units

- `1` = 1 vCPU / 1 core / 1 hyperthread
- `500m` = 0.5 CPU (m = millicores, 1000m = 1 CPU)
- `100m` = 0.1 CPU — smallest recommended value for most workloads
- CPU is **compressible** — if a container exceeds its limit, it is **throttled** (slowed down), not killed

#### Memory units

- `128Mi` = 128 mebibytes (1 Mi = 1024 × 1024 bytes)
- `256Mi`, `512Mi`, `1Gi` — common values
- Memory is **incompressible** — if a container exceeds its limit, it is **OOMKilled** (killed immediately) and restarted

#### Requests vs Limits

| | Request | Limit |
|---|---|---|
| **Purpose** | Scheduling hint — minimum guaranteed | Hard ceiling — maximum allowed |
| **Used by** | kube-scheduler (picking a node) | kubelet + kernel (enforcement) |
| **CPU behavior** | Guaranteed CPU share | Throttled if exceeded |
| **Memory behavior** | Guaranteed memory | OOMKilled if exceeded |
| **Best practice** | Set based on normal load | Set 2× request as a starting point |

#### QoS Classes (determined automatically from requests/limits)

| Class | Condition | Eviction priority |
|---|---|---|
| **Guaranteed** | requests == limits for all containers | Last to be evicted |
| **Burstable** | requests < limits (or only one set) | Middle priority |
| **BestEffort** | No requests or limits set | First to be evicted |

Always set both requests and limits for production workloads to achieve **Guaranteed** or at minimum **Burstable** QoS.

---

## Practical Lab

### 1. Write a Deployment Manifest for Node.js App

```yaml
# node-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app
  namespace: default
  labels:
    app: node-app
    version: "1.0.0"
  annotations:
    kubernetes.io/change-cause: "v1.0.0 — initial deployment"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: node-app          # Must match template labels below

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1

  template:
    metadata:
      labels:
        app: node-app        # Must match selector above
    spec:
      containers:
        - name: node-app
          image: node:18-alpine   # Use a public image for lab testing
          command: ["node", "-e", "
            const http = require('http');
            const server = http.createServer((req, res) => {
              res.writeHead(200);
              res.end('Hello from v1.0.0\\n');
            });
            server.listen(3000);
            console.log('Server running on port 3000');
          "]
          ports:
            - containerPort: 3000
              name: http

          # Resource requests and limits (Assignment 23 values)
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"

          # Liveness probe: restart if app hangs
          livenessProbe:
            httpGet:
              path: /
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3

          # Readiness probe: only send traffic when ready
          readinessProbe:
            httpGet:
              path: /
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
```

```bash
# Apply the manifest
kubectl apply -f node-app-deployment.yaml

# Verify deployment was created
kubectl get deployments
# NAME       READY   UP-TO-DATE   AVAILABLE   AGE
# node-app   3/3     3            3           30s

# Verify ReplicaSet was created
kubectl get replicasets
# NAME                  DESIRED   CURRENT   READY   AGE
# node-app-6d8f9b7c4f   3         3         3       30s

# Verify pods are running
kubectl get pods -l app=node-app
# NAME                        READY   STATUS    RESTARTS   AGE
# node-app-6d8f9b7c4f-2xkpt   1/1     Running   0          30s
# node-app-6d8f9b7c4f-7mqhs   1/1     Running   0          30s
# node-app-6d8f9b7c4f-nz9wl   1/1     Running   0          30s

# Verify resource limits are applied
kubectl describe pod node-app-6d8f9b7c4f-2xkpt | grep -A 6 "Limits:"
# Limits:
#   cpu:     500m
#   memory:  256Mi
# Requests:
#   cpu:     100m
#   memory:  128Mi
```

---

### 2. Scale Replicas

```bash
# Scale imperatively (immediate, no manifest edit)
kubectl scale deployment/node-app --replicas=5

# Watch pods come up
kubectl get pods -l app=node-app -w
# NAME                        READY   STATUS              RESTARTS
# node-app-6d8f9b7c4f-2xkpt   1/1     Running             0
# node-app-6d8f9b7c4f-7mqhs   1/1     Running             0
# node-app-6d8f9b7c4f-nz9wl   1/1     Running             0
# node-app-6d8f9b7c4f-p4rlm   0/1     ContainerCreating   0
# node-app-6d8f9b7c4f-q8vkj   0/1     ContainerCreating   0
# node-app-6d8f9b7c4f-p4rlm   1/1     Running             0
# node-app-6d8f9b7c4f-q8vkj   1/1     Running             0

# Scale back down
kubectl scale deployment/node-app --replicas=3
```

> **Note:** Scaling imperatively is fine for testing. In production, always update the `replicas` field in the manifest and re-apply, so your Git repo reflects the actual state.

---

### 3. Trigger Rolling Update by Changing Image Tag

```bash
# Method 1 — Imperative (good for quick testing)
kubectl set image deployment/node-app node-app=node:20-alpine

# Method 2 — Edit the manifest, change image tag, re-apply (recommended)
# In node-app-deployment.yaml, change:
#   image: node:18-alpine  →  image: node:20-alpine
# Also update the change-cause annotation
kubectl apply -f node-app-deployment.yaml

# Watch the rolling update as it happens (open in a second terminal)
kubectl rollout status deployment/node-app
# Waiting for deployment "node-app" rollout to finish: 1 out of 3 new replicas updated...
# Waiting for deployment "node-app" rollout to finish: 1 old replicas are pending termination...
# Waiting for deployment "node-app" rollout to finish: 2 out of 3 new replicas updated...
# Waiting for deployment "node-app" rollout to finish: 1 old replicas are pending termination...
# deployment "node-app" successfully rolled out

# Watch individual pods cycle through (second terminal)
kubectl get pods -l app=node-app -w

# Confirm two ReplicaSets now exist (old scaled to 0, new to 3)
kubectl get replicasets
# NAME                  DESIRED   CURRENT   READY   AGE
# node-app-6d8f9b7c4f   0         0         0       5m    ← old RS (v1.0)
# node-app-7c9f8d2a1b   3         3         3       1m    ← new RS (v1.1)

# Check rollout history
kubectl rollout history deployment/node-app
# REVISION  CHANGE-CAUSE
# 1         v1.0.0 — initial deployment
# 2         v1.1.0 — upgrade to node:20-alpine
```

---

### 4. Perform Rollback with kubectl rollout undo

```bash
# Rollback to the previous revision (revision 1)
kubectl rollout undo deployment/node-app

# Watch the rollback
kubectl rollout status deployment/node-app
# Waiting for deployment "node-app" rollout to finish: 2 out of 3 new replicas updated...
# deployment "node-app" successfully rolled out

# Verify the old ReplicaSet is back at 3 replicas
kubectl get replicasets
# NAME                  DESIRED   CURRENT   READY   AGE
# node-app-6d8f9b7c4f   3         3         3       10m   ← old RS restored
# node-app-7c9f8d2a1b   0         0         0       5m    ← new RS scaled to 0

# Rollback creates revision 3 (not back to 1; history moves forward)
kubectl rollout history deployment/node-app
# REVISION  CHANGE-CAUSE
# 2         v1.1.0 — upgrade to node:20-alpine
# 3         v1.0.0 — initial deployment   ← revision 1 was re-applied as 3

# Rollback to a specific revision by number
kubectl rollout undo deployment/node-app --to-revision=2
```

---

## Assignment 23

> Set **CPU request 100m, limit 500m** and **memory request 128Mi, limit 256Mi** in the manifest.

The complete production-ready manifest with resource limits, probes, and rollout strategy:

```yaml
# node-app-deployment.yaml  (final version with all Assignment 23 requirements)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app
  namespace: default
  labels:
    app: node-app
  annotations:
    kubernetes.io/change-cause: "v1.0.0 — initial deployment with resource limits"
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

  # Keep last 5 ReplicaSets for rollback history
  revisionHistoryLimit: 5

  template:
    metadata:
      labels:
        app: node-app
    spec:
      containers:
        - name: node-app
          image: my-nodejs-app:1.0.0
          ports:
            - containerPort: 3000
              name: http

          # ── Assignment 23: Resource Requests and Limits ───────────────────
          resources:
            requests:
              cpu: "100m"     # 0.1 vCPU guaranteed — used by scheduler
              memory: "128Mi" # 128 MiB guaranteed — used by scheduler
            limits:
              cpu: "500m"     # 0.5 vCPU ceiling — throttled if exceeded
              memory: "256Mi" # 256 MiB ceiling — OOMKilled if exceeded
          # ─────────────────────────────────────────────────────────────────

          # Environment variables
          env:
            - name: NODE_ENV
              value: "production"
            - name: PORT
              value: "3000"

          # Liveness probe — kubelet restarts container if this fails
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15   # Wait for app startup
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3

          # Readiness probe — pod removed from Service if this fails
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3

      # Graceful shutdown — allow in-flight requests to complete
      terminationGracePeriodSeconds: 30
```

```bash
# Apply the final manifest
kubectl apply -f node-app-deployment.yaml

# Verify resource limits are set correctly on a pod
kubectl describe pod -l app=node-app | grep -A 8 "Limits:"
# Limits:
#   cpu:     500m
#   memory:  256Mi
# Requests:
#   cpu:     100m
#   memory:  128Mi

# Check QoS class (should be Burstable since requests < limits)
kubectl get pod -l app=node-app -o jsonpath='{.items[0].status.qosClass}'
# Burstable

# To achieve Guaranteed QoS, set requests == limits:
# requests:
#   cpu: "500m"
#   memory: "256Mi"
# limits:
#   cpu: "500m"
#   memory: "256Mi"
```

**Understanding the assignment values:**

```
CPU:
  100m request = app is guaranteed 0.1 CPU core at all times
  500m limit   = app can burst up to 0.5 CPU core; throttled beyond that
  Ratio: 5:1 (allow bursting but cap runaway CPU usage)

Memory:
  128Mi request = app is guaranteed 128 MiB RAM
  256Mi limit   = app is OOMKilled if it uses more than 256 MiB
  Ratio: 2:1 (tight ceiling — good for Node.js apps with known heap size)

QoS Class: Burstable
  → requests < limits means app can burst when resources are available
  → will be evicted before Guaranteed pods if node is under memory pressure
```

---

## Quick Reference

```
Pod       = one or more containers sharing network + volumes (ephemeral)
ReplicaSet = maintains N identical pod replicas (rarely written directly)
Deployment = manages ReplicaSets; handles rolling updates + rollbacks

Rolling update flow:
  Apply new spec → new ReplicaSet created → pods shifted old→new gradually
  maxUnavailable: how many can be down at once
  maxSurge:       how many extra pods can exist during transition

Rollout commands:
  kubectl rollout status deployment/<name>       watch progress
  kubectl rollout history deployment/<name>      list revisions
  kubectl rollout undo deployment/<name>         rollback one step
  kubectl rollout undo deployment/<name> --to-revision=N  specific
  kubectl rollout pause / resume deployment/<name>

Resource units:
  CPU:    1000m = 1 core    100m = 0.1 core
  Memory: 1024Mi = 1Gi      128Mi, 256Mi, 512Mi are common values

CPU exceeded  → container throttled (slowed, not killed)
Memory exceeded → container OOMKilled (restarted immediately)

QoS classes:
  Guaranteed  requests == limits   ← last evicted
  Burstable   requests < limits    ← middle
  BestEffort  nothing set          ← first evicted
```

| Field | What it controls |
|---|---|
| `spec.replicas` | Number of pod copies |
| `spec.selector` | Which pods this Deployment owns |
| `spec.template` | The pod blueprint |
| `spec.strategy.type` | `RollingUpdate` or `Recreate` |
| `spec.strategy.rollingUpdate.maxUnavailable` | Min availability during update |
| `spec.strategy.rollingUpdate.maxSurge` | Extra pods allowed during update |
| `spec.revisionHistoryLimit` | How many old ReplicaSets to keep |
| `resources.requests.cpu` | Scheduler reservation |
| `resources.limits.cpu` | Throttle threshold |
| `resources.requests.memory` | Scheduler reservation |
| `resources.limits.memory` | OOMKill threshold |
