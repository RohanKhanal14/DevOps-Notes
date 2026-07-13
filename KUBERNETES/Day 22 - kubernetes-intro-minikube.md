# Day 22 — Kubernetes Introduction & Minikube

> K8s architecture, cluster setup with Minikube, kubectl basics

---

## Theory

### Why Kubernetes?

Running containers with plain Docker works for a single machine, but falls apart at scale. Kubernetes solves the operational problems that appear when you run many containers across many machines.

| Problem (without K8s) | Kubernetes Solution |
|---|---|
| Container crashes, stays down | **Self-healing** — restarts failed containers automatically |
| Traffic spike, need more instances | **Horizontal scaling** — adds/removes pods based on load |
| Deploy new version = manual coordination | **Declarative management** — describe desired state; K8s reconciles |
| Routing traffic to healthy containers | **Load balancing** — built-in service discovery and traffic distribution |
| Secrets, config scattered across hosts | **ConfigMaps & Secrets** — centralized, injected at runtime |
| Rolling out updates safely | **Rolling updates & rollbacks** — zero-downtime deployments |
| Packing containers efficiently onto nodes | **Bin packing** — scheduler places workloads to maximize resource usage |

**The core idea:** you tell Kubernetes *what* you want (desired state), and it continuously works to make reality match that description. This is called the **reconciliation loop**.

```
You declare:  "I want 3 replicas of my-app running"
K8s observes: "There are currently 2 running"
K8s acts:     "Starting 1 more pod..."
K8s observes: "There are now 3 running" ✅
```

---

### Kubernetes Architecture

A Kubernetes cluster has two logical layers: the **Control Plane** (the brain) and **Worker Nodes** (the muscle).

![Alt text](images/K8/k8-architecture.svg)

---

#### Control Plane Components (deep dive)

**API Server (`kube-apiserver`)**
- The single entry point for all cluster operations
- Every `kubectl` command, every controller, every node — all communicate through the API server
- Validates and persists objects to etcd
- Handles authentication, authorization (RBAC), and admission control

**etcd**
- A distributed key-value store — the source of truth for all cluster state
- Stores every Kubernetes object (pods, services, secrets, etc.)
- Runs as a cluster itself (usually 3 or 5 nodes) for high availability
- If etcd is lost without a backup, the cluster state is gone — always back it up

**Scheduler (`kube-scheduler`)**
- Watches for pods that have no assigned node (`nodeName` is empty)
- Selects the best node based on: available resources, affinity/anti-affinity rules, taints/tolerations, topology spread
- Writes the node assignment back to the API server (does not start the pod directly)

**Controller Manager (`kube-controller-manager`)**
- Runs multiple control loops (controllers) in a single binary
- Each controller watches a resource type and reconciles actual state with desired state:
  - **ReplicaSet controller** — ensures the right number of pod replicas exist
  - **Node controller** — marks nodes as not-ready when they stop responding
  - **Job controller** — runs batch jobs to completion
  - **Service Account controller** — creates default service accounts in new namespaces

---

#### Worker Node Components (deep dive)

**kubelet**
- The primary node agent, runs on every worker node
- Watches the API server for pods assigned to its node
- Instructs the container runtime to start/stop containers
- Reports node and pod status back to the API server
- Runs liveness and readiness probes

**kube-proxy**
- Runs on every node; maintains network rules (iptables or IPVS)
- Implements the `Service` abstraction — routes traffic from a Service's ClusterIP to the correct pod IPs
- Handles load balancing across multiple pod endpoints

**Container Runtime**
- The software that actually runs containers: `containerd` (most common), `CRI-O`
- Implements the Container Runtime Interface (CRI)
- Pulls images, creates containers, manages their lifecycle

---

### Minikube: Single-Node K8s for Local Development

Minikube runs a complete Kubernetes cluster inside a single VM or container on your laptop. Both control plane and worker node components run on the same machine.

```
Your Machine
└── Minikube VM / Docker container
    ├── Control Plane (API server, etcd, scheduler, controller-manager)
    └── Worker Node   (kubelet, kube-proxy, containerd)
        └── Your pods
```

**Key Minikube concepts:**
- **Driver** — how Minikube runs the cluster: `docker` (default), `virtualbox`, `hyperkit`, `kvm2`
- **Profile** — a named cluster; you can run multiple isolated clusters with different profiles
- **Addons** — optional components: `metrics-server`, `ingress`, `dashboard`, `registry`

```bash
# Common Minikube commands
minikube start                          # Start with defaults
minikube start --cpus=4 --memory=8192  # Custom resources
minikube start --driver=docker          # Specify driver
minikube stop                           # Stop (preserves state)
minikube delete                         # Delete cluster completely
minikube status                         # Check component health
minikube dashboard                      # Open web UI in browser
minikube addons list                    # Show available addons
minikube addons enable metrics-server   # Enable an addon
minikube tunnel                         # Expose LoadBalancer services locally
minikube image load my-app:latest       # Load local Docker image into cluster
```

---

### kubectl: Context, Config, Basic Commands

`kubectl` is the CLI for interacting with any Kubernetes cluster. It reads from `~/.kube/config` to know which cluster to talk to.

#### Kubeconfig and Contexts

```yaml
# ~/.kube/config (simplified)
apiVersion: v1
kind: Config

clusters:
  - name: minikube
    cluster:
      server: https://192.168.49.2:8443
      certificate-authority: /path/to/ca.crt

users:
  - name: minikube
    user:
      client-certificate: /path/to/client.crt
      client-key: /path/to/client.key

contexts:
  - name: minikube           # A context = cluster + user + namespace
    context:
      cluster: minikube
      user: minikube
      namespace: default

current-context: minikube   # Which context kubectl uses right now
```

```bash
# Context management
kubectl config get-contexts                    # List all contexts
kubectl config current-context                 # Show active context
kubectl config use-context minikube            # Switch context
kubectl config set-context --current \
  --namespace=my-namespace                     # Set default namespace

# Multi-cluster tip: use kubectx (brew install kubectx) for fast switching
kubectx minikube
kubectx production   # switch to prod cluster
```

---

#### Essential kubectl Commands

**Cluster info**
```bash
kubectl cluster-info                    # API server and CoreDNS URLs
kubectl get nodes                       # List nodes with status
kubectl get nodes -o wide              # Include IP, OS, container runtime
kubectl describe node minikube         # Full node details: capacity, conditions, pods
kubectl top nodes                      # CPU/memory usage (needs metrics-server)
```

**Working with resources**
```bash
# Get resources
kubectl get pods                        # Pods in current namespace
kubectl get pods -n kube-system        # Pods in kube-system namespace
kubectl get pods -A                    # Pods in ALL namespaces
kubectl get pods -o wide               # Include node, IP
kubectl get pods -o yaml               # Full YAML output
kubectl get pods -w                    # Watch (live updates)
kubectl get all                        # Pods, services, deployments, etc.

# Describe (human-readable detail)
kubectl describe pod <pod-name>        # Events, containers, volumes, conditions
kubectl describe node <node-name>

# Logs
kubectl logs <pod-name>                # Stdout of the container
kubectl logs <pod-name> -f             # Follow (tail -f equivalent)
kubectl logs <pod-name> --previous     # Logs from a crashed previous container
kubectl logs <pod-name> -c <container> # Specific container in multi-container pod

# Execute commands
kubectl exec -it <pod-name> -- bash   # Shell into a pod
kubectl exec <pod-name> -- env         # Run a single command

# Apply and delete
kubectl apply -f manifest.yaml         # Create or update from file
kubectl apply -f ./k8s/               # Apply all YAMLs in a directory
kubectl delete -f manifest.yaml        # Delete resources defined in file
kubectl delete pod <pod-name>          # Delete a specific pod

# Editing
kubectl edit deployment my-app         # Open resource in $EDITOR
kubectl scale deployment my-app --replicas=5  # Scale immediately
kubectl rollout status deployment my-app      # Watch rollout progress
kubectl rollout undo deployment my-app        # Rollback to previous version
```

**Namespaces**
```bash
kubectl get namespaces                 # List all namespaces
kubectl create namespace staging       # Create a namespace
kubectl get pods -n staging            # Work in a specific namespace
```

---

### YAML Manifests: Structure

Every Kubernetes object is described in YAML with four top-level fields:

```yaml
apiVersion: apps/v1        # API group + version (who handles this object)
kind: Deployment           # The type of object
metadata:                  # Identity and labels
  name: my-app
  namespace: default
  labels:
    app: my-app
    env: production
  annotations:
    description: "Main application deployment"
spec:                      # Desired state (varies per kind)
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:1.0.0
          ports:
            - containerPort: 3000
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
```

**`apiVersion` reference for common kinds:**

| Kind | apiVersion |
|---|---|
| Pod | `v1` |
| Service | `v1` |
| ConfigMap | `v1` |
| Secret | `v1` |
| Namespace | `v1` |
| Deployment | `apps/v1` |
| StatefulSet | `apps/v1` |
| DaemonSet | `apps/v1` |
| Job | `batch/v1` |
| CronJob | `batch/v1` |
| Ingress | `networking.k8s.io/v1` |
| HorizontalPodAutoscaler | `autoscaling/v2` |

---

## Practical Lab

### 1. Install Minikube and kubectl

**macOS:**
```bash
# kubectl
brew install kubectl

# Minikube
brew install minikube

# Verify
kubectl version --client
minikube version
```

**Linux (Ubuntu/Debian):**
```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verify
kubectl version --client
minikube version
```

**Windows (via Chocolatey):**
```powershell
choco install kubernetes-cli minikube
```

---

### 2. Start the Cluster

```bash
# Start Minikube (Docker driver is default and recommended)
minikube start --cpus=2 --memory=4096

# Expected output:
# 😄  minikube v1.x.x
# ✨  Using the docker driver based on existing profile
# 👍  Starting control plane node minikube in cluster minikube
# 🔥  Creating docker container (CPUs=2, Memory=4096MB) ...
# 🐳  Preparing Kubernetes v1.x.x on Docker xx.x.x ...
# 🔎  Verifying Kubernetes components...
# 🌟  Enabled addons: default-storageclass, storage-provisioner
# 🏄  Done! kubectl is now configured to use "minikube" cluster

# Verify kubectl is pointing to Minikube
kubectl config current-context
# → minikube
```

---

### 3. kubectl get nodes

```bash
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   5m    v1.28.3

kubectl get nodes -o wide
# NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    OS-IMAGE
# minikube   Ready    control-plane   5m    v1.28.3   192.168.49.2   Ubuntu 22.04
```

---

### 4. kubectl describe node

```bash
kubectl describe node minikube
```

Key sections to understand in the output:

```
Name:               minikube
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=amd64
                    kubernetes.io/hostname=minikube
                    node-role.kubernetes.io/control-plane=...

Conditions:                          ← Node health checks
  Type             Status
  MemoryPressure   False             ← Enough memory
  DiskPressure     False             ← Enough disk
  PIDPressure      False             ← Enough PIDs
  Ready            True              ← Node is healthy

Capacity:                            ← Total resources on the node
  cpu:             4
  memory:          8192Mi
  pods:            110

Allocatable:                         ← Resources available to pods
  cpu:             4
  memory:          7936Mi            ← Slightly less (system reserved)
  pods:            110

System Info:
  OS Image:        Ubuntu 22.04
  Container Runtime Version: containerd://1.7.x
  Kubelet Version: v1.28.3

Non-terminated Pods:                 ← Pods currently running on this node
  kube-system/coredns-xxx
  kube-system/etcd-minikube
  kube-system/kube-apiserver-minikube
  ...

Allocated resources:                 ← Sum of all pod resource requests
  Resource    Requests   Limits
  cpu         750m       0
  memory      170Mi      170Mi
```

---

### 5. kubectl cluster-info

```bash
kubectl cluster-info
# Kubernetes control plane is running at https://192.168.49.2:8443
# CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/...

# Explore what's running in kube-system (the control plane namespace)
kubectl get pods -n kube-system
# NAME                               READY   STATUS    RESTARTS
# coredns-xxx                        1/1     Running   0
# etcd-minikube                      1/1     Running   0
# kube-apiserver-minikube            1/1     Running   0
# kube-controller-manager-minikube   1/1     Running   0
# kube-proxy-xxx                     1/1     Running   0
# kube-scheduler-minikube            1/1     Running   0
# storage-provisioner                1/1     Running   0
```

---

### 6. Run Your First Pod

```bash
# Run an nginx pod imperatively (good for quick testing)
kubectl run nginx-test --image=nginx:alpine --port=80

# Check it's running
kubectl get pods
# NAME         READY   STATUS    RESTARTS   AGE
# nginx-test   1/1     Running   0          30s

# Describe it
kubectl describe pod nginx-test

# Access it (port-forward to your local machine)
kubectl port-forward pod/nginx-test 8080:80 &
curl http://localhost:8080

# Clean up
kubectl delete pod nginx-test
```

---

## Quick Reference

```
Cluster = Control Plane + Worker Nodes

Control Plane components:
  kube-apiserver        → single entry point, auth, persists to etcd
  etcd                  → distributed KV store, cluster source of truth
  kube-scheduler        → places pods on nodes
  kube-controller-mgr   → reconciliation loops (replica count, node health)

Worker Node components:
  kubelet               → node agent, runs pods, reports to API server
  kube-proxy            → iptables rules for Service routing
  containerd            → container runtime (pulls images, runs containers)

kubectl config:
  ~/.kube/config        → clusters, users, contexts
  context = cluster + user + namespace
  kubectl config use-context <name>   → switch cluster

YAML manifest top-level fields:
  apiVersion    → which API handles this (v1, apps/v1, batch/v1)
  kind          → object type (Pod, Deployment, Service)
  metadata      → name, namespace, labels, annotations
  spec          → desired state (specific to each kind)
```

| kubectl shortcut | Full resource name |
|---|---|
| `po` | pods |
| `svc` | services |
| `deploy` | deployments |
| `rs` | replicasets |
| `ns` | namespaces |
| `cm` | configmaps |
| `no` | nodes |
| `-A` | `--all-namespaces` |
| `-o wide` | extra columns |
| `-o yaml` | full YAML output |
