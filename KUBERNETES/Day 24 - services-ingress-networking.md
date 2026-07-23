# Day 24 — Services, Ingress & Networking

> Expose applications, Ingress controller, DNS, network policies

---

This is a github repo that has all the codes of k8s : https://github.com/RohanKhanal14/three-tier-crud-app  

Note : Make sure to use k8s branch cause the codes are in there.
## Theory

### The Problem Services Solve

Pods are ephemeral — they crash, get rescheduled, scaled up and down, and replaced during rolling updates. Their IP addresses change every time. You cannot hardcode a pod IP.

A **Service** gives you a **stable virtual IP (ClusterIP)** and DNS name that never changes, even as the pods behind it come and go. kube-proxy keeps the routing rules updated automatically.

```
Without a Service:                With a Service:
                                  
  Pod A  10.0.0.5  ← crashed      ┌─── Service (10.96.0.1) ───┐
  Pod A  10.0.0.9  ← new IP       │   stable IP + DNS name    │
  Pod B  10.0.0.6                 └─────────────┬─────────────┘
                                                │ kube-proxy routes
  Consumer must track IPs              ┌────────┴────────┐
  → impossible at scale                ▼                 ▼
                                   Pod A 10.0.0.9   Pod B 10.0.0.6
```

A Service selects its pods using **label selectors** — the same labels you put on pods in Deployments.

---

### Service Types

#### 1. ClusterIP (default)

Exposes the service on an internal IP that is **only reachable from within the cluster**. This is the most common type — used for internal service-to-service communication.

```
┌─────────────────────── Cluster ──────────────────────────┐
│                                                          │
│   Frontend Pod ──────▶ ClusterIP: 10.96.0.42:80 ─────▶ Backend Pods
│                        (only reachable inside cluster)   │
│                                                          │
│   External traffic ✗  (cannot reach ClusterIP from outside)
└──────────────────────────────────────────────────────────┘
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP          # Default; can omit this line
  selector:
    app: backend           # Routes to pods with this label
  ports:
    - port: 80             # Port the Service listens on
      targetPort: 3000     # Port the container listens on
      protocol: TCP
```

```bash
# Accessible within the cluster at:
#   http://backend-svc           (same namespace)
#   http://backend-svc.default   (same namespace, explicit)
#   http://backend-svc.default.svc.cluster.local  (FQDN)
```

---

#### 2. NodePort

Exposes the service on **every node's IP** at a static port (30000–32767). External traffic can reach pods at `<NodeIP>:<NodePort>`. Used for development and direct node access — not recommended for production (bypasses load balancers, exposes node IPs).

```
External User
     │
     ▼
Node IP:32000  (any node in the cluster)
     │
     ▼  kube-proxy routes
ClusterIP:80
     │
     ▼
Pod(s)  port 3000
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-app-nodeport
spec:
  type: NodePort
  selector:
    app: node-app
  ports:
    - port: 80             # ClusterIP port (internal)
      targetPort: 3000     # Container port
      nodePort: 32000      # External port on every node (30000–32767)
                           # Omit to let K8s assign a random port in range
```

```bash
# Access via Minikube:
minikube ip            # → 192.168.49.2
curl http://192.168.49.2:32000

# Or let minikube open it:
minikube service node-app-nodeport --url
```

---

#### 3. LoadBalancer

Provisions a **cloud provider's external load balancer** (AWS ELB, GCP Load Balancer, Azure LB) and assigns it a public IP. The standard way to expose services in production on managed Kubernetes (EKS, GKE, AKS).

```
Internet
    │
    ▼
Cloud Load Balancer (external public IP)
    │
    ▼
NodePort (every node)
    │
    ▼
ClusterIP
    │
    ▼
Pod(s)
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-app-lb
spec:
  type: LoadBalancer
  selector:
    app: node-app
  ports:
    - port: 80
      targetPort: 3000
```

```bash
# On a cloud cluster:
kubectl get svc node-app-lb
# NAME          TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)
# node-app-lb   LoadBalancer   10.96.0.44    34.120.55.101    80:31245/TCP

# In Minikube (no cloud LB available — use tunnel):
minikube tunnel    # runs in a separate terminal, assigns 127.0.0.1 as EXTERNAL-IP
curl http://127.0.0.1:80
```

---

#### Service Type Comparison

| Type             | Reachable from               | Use case                                |
| ---------------- | ---------------------------- | --------------------------------------- |
| **ClusterIP**    | Inside cluster only          | Service-to-service communication        |
| **NodePort**     | Node IP + port               | Dev/testing, direct node access         |
| **LoadBalancer** | Public internet via cloud LB | Production on managed K8s (EKS/GKE/AKS) |
| **ExternalName** | Inside cluster               | Alias to an external DNS name           |

---

### kube-dns: Service Discovery by Name

Every cluster runs a DNS server (`CoreDNS` in modern K8s) that automatically creates DNS records for Services. Pods can reach any Service by name — no IP address needed.

#### DNS name format

```
<service-name>.<namespace>.svc.cluster.local

Examples:
  backend-svc.default.svc.cluster.local   ← full FQDN
  backend-svc.default                     ← short form across namespaces
  backend-svc                             ← short form in same namespace
```

#### DNS resolution in practice

```bash
# From inside a pod, test DNS resolution:
kubectl run dns-test --image=busybox --rm -it --restart=Never -- sh

# Inside the shell:
nslookup backend-svc
# Server:    10.96.0.10
# Address:   10.96.0.10:53
# Name:      backend-svc.default.svc.cluster.local
# Address:   10.96.0.42

wget -qO- http://backend-svc:80
```

#### How it works

```
Pod starts
  → /etc/resolv.conf set to: nameserver 10.96.0.10 (CoreDNS ClusterIP)
  → search: default.svc.cluster.local svc.cluster.local cluster.local

Pod queries: "backend-svc"
  → CoreDNS resolves → 10.96.0.42 (Service ClusterIP)
  → kube-proxy routes → one of the backend pod IPs
```

---

### Ingress: Path-Based and Host-Based Routing

A **LoadBalancer Service per app** is expensive each one provisions a cloud load balancer with its own public IP. **Ingress** solves this with a single entry point for all HTTP/HTTPS traffic, routing to different services based on URL path or hostname.

```
Internet
    │
    ▼
Single Load Balancer / NodePort
    │
    ▼
Ingress Controller (nginx pod)
    │
    ├── /api/*        ──▶  api-service:80        ──▶  api pods
    ├── /             ──▶  frontend-service:80   ──▶  frontend pods
    │
    ├── api.myapp.com ──▶  api-service:80
    └── myapp.com     ──▶  frontend-service:80
```

#### Path-based routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: node-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /    # Strip path prefix before forwarding
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

#### Host-based routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80

    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

  # TLS termination
  tls:
    - hosts:
        - myapp.com
        - api.myapp.com
      secretName: myapp-tls-secret   # kubectl create secret tls ...
```

#### pathType values

| pathType                 | Behavior                                                                  |
| ------------------------ | ------------------------------------------------------------------------- |
| `Exact`                  | Match the exact path only (`/api` does NOT match `/api/users`)            |
| `Prefix`                 | Match path and all sub-paths (`/api` matches `/api/users`, `/api/v2/...`) |
| `ImplementationSpecific` | Behavior depends on the Ingress controller                                |

---

### Ingress Controller (nginx-ingress in Minikube)

An **Ingress resource** is just a configuration object — it does nothing on its own. An **Ingress controller** is the actual software (a pod) that reads Ingress resources and configures a reverse proxy (nginx, HAProxy, Traefik, etc.).

```
Ingress Resource (YAML)  ──▶  Ingress Controller Pod  ──▶  nginx config
   (K8s object)                    (watches API server)       (auto-updated)
```

In Minikube, the easiest option is the built-in nginx addon:

```bash
minikube addons enable ingress

# Verify the controller is running
kubectl get pods -n ingress-nginx
# NAME                                        READY   STATUS
# ingress-nginx-controller-7dcdbcff84-xzpmk   1/1     Running
```

---

### Network Policies: Pod-to-Pod Traffic Rules

By default, **all pods in a Kubernetes cluster can communicate with all other pods**, regardless of namespace. This is a security risk — a compromised frontend pod could directly query the database.

**NetworkPolicy** objects let you define firewall rules at the pod level using label selectors.

#### Key concepts

- NetworkPolicies are **namespaced** — they only apply to pods in the same namespace
- A pod with **no NetworkPolicy** allows all traffic (ingress + egress)
- Once **any** NetworkPolicy selects a pod, **all traffic not explicitly allowed is denied** (default-deny for that direction)
- Policies are **additive** — multiple policies on a pod combine with OR logic

#### Policy anatomy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: default
spec:
  podSelector:           # Applies to pods matching these labels (the target)
    matchLabels:
      app: backend

  policyTypes:
    - Ingress            # Control incoming traffic
    - Egress             # Control outgoing traffic

  ingress:
    - from:              # Allow FROM these sources
        - podSelector:
            matchLabels:
              app: frontend    # Only frontend pods
      ports:
        - protocol: TCP
          port: 3000

  egress:
    - to:                # Allow TO these destinations
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

#### Traffic direction reference

```
INGRESS = traffic coming IN to the selected pods
EGRESS  = traffic going OUT from the selected pods

Frontend ──(egress from frontend)──▶ Backend ──(ingress to backend)──▶ Backend

To allow frontend → backend communication you need:
  Option A: NetworkPolicy on backend allowing ingress FROM frontend
  Option B: NetworkPolicy on frontend allowing egress TO backend
  Option C: Both (for strict bidirectional control)
```

---

## Practical Lab

### 1. Create NodePort Service for Node.js App

```yaml
# node-app-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: node-app-svc
  labels:
    app: node-app
spec:
  type: NodePort
  selector:
    app: node-app          # Must match pod labels from Day 23 deployment
  ports:
    - name: http
      port: 80             # ClusterIP port
      targetPort: 3000     # Container port
      nodePort: 32080      # External port (choose 30000–32767)
```

```bash
# Apply the service
kubectl apply -f node-app-service.yaml

# Verify
kubectl get svc node-app-svc
# NAME           TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
# node-app-svc   NodePort   10.96.10.44   <none>        80:32080/TCP   10s

# Describe to see endpoint details (shows which pod IPs are targeted)
kubectl describe svc node-app-svc
# Endpoints: 10.244.0.5:3000,10.244.0.6:3000,10.244.0.7:3000

# Get Minikube IP
minikube ip
# 192.168.49.2

# Access the app
curl http://192.168.49.2:32080

# Shortcut — let Minikube open it for you
minikube service node-app-svc
minikube service node-app-svc --url   # just print the URL
```

---

### 2. Install nginx-ingress Addon & Create Ingress

#### Enable the addon

```bash
minikube addons enable ingress

# Wait for controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

kubectl get pods -n ingress-nginx
# NAME                                        READY   STATUS    AGE
# ingress-nginx-controller-7dcdbcff84-xzpmk   1/1     Running   60s
```

#### Create ClusterIP services for path routing

```yaml
# services.yaml — ClusterIP services (Ingress routes to these)
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 3000
```

#### Create the Ingress resource

```yaml
# node-app-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: node-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.local    # Add to /etc/hosts: <minikube ip> myapp.local
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-service
                port:
                  number: 80

          - path: /()(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

```bash
kubectl apply -f node-app-ingress.yaml

# Verify Ingress has an address
kubectl get ingress node-app-ingress
# NAME               CLASS   HOSTS         ADDRESS          PORTS
# node-app-ingress   nginx   myapp.local   192.168.49.2     80

# Add Minikube IP to /etc/hosts
echo "$(minikube ip) myapp.local" | sudo tee -a /etc/hosts

# Test path routing
curl http://myapp.local/          # → frontend-service
curl http://myapp.local/api/users # → api-service

# Inspect routing rules Ingress controller applied
kubectl describe ingress node-app-ingress
```

---

## Assignment 24

> Write a **NetworkPolicy that allows only frontend pods to talk to backend pods**.

### The Setup

Assume these pods exist with the following labels:

```
frontend pods:   app=frontend, tier=frontend
backend pods:    app=backend,  tier=backend
database pods:   app=postgres, tier=database
```

---

### Policy 1 — Restrict ingress to backend (recommended approach)

This policy makes the backend pods only accept traffic from frontend pods. All other sources (other services, external) are denied.

```yaml
# netpol-backend-allow-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend-only
  namespace: default
spec:
  # This policy applies to backend pods
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress      # Only restrict incoming traffic to backend

  ingress:
    - from:
        # Allow ingress ONLY from pods with app=frontend label
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 3000   # Backend container port
```

```bash
kubectl apply -f netpol-backend-allow-frontend.yaml

# Verify the policy
kubectl get networkpolicy
# NAME                           POD-SELECTOR   AGE
# backend-allow-frontend-only    app=backend    5s

kubectl describe networkpolicy backend-allow-frontend-only
```

---

### Policy 2 — Default deny all (add before the allow policy in production)

Without this, pods with no NetworkPolicy still allow all traffic. This policy denies everything by default for backend pods, and the allow policy above then opens the specific exception.

```yaml
# netpol-default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}     # Matches ALL pods in the namespace
  policyTypes:
    - Ingress
    - Egress
  # No ingress or egress rules = deny all traffic
```

---

### Policy 3 — Full stack: frontend → backend → database

A complete three-tier network policy setup:

```yaml
# netpol-three-tier.yaml

# ── Policy 1: Backend accepts traffic ONLY from frontend ──────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress-from-frontend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 3000

---
# ── Policy 2: Database accepts traffic ONLY from backend ──────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-ingress-from-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5432

---
# ── Policy 3: Frontend allows egress ONLY to backend + DNS ───────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 3000

    # Always allow DNS — without this, name resolution breaks
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

---
# ── Policy 4: Backend allows egress ONLY to database + DNS ───────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-to-database
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432

    # Always allow DNS
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

```bash
kubectl apply -f netpol-three-tier.yaml

# Test connectivity (from inside the cluster)
# Frontend → Backend (should work ✅)
kubectl exec -it <frontend-pod> -- curl http://backend-svc:3000

# Frontend → Database directly (should be blocked ❌)
kubectl exec -it <frontend-pod> -- nc -zv postgres-svc 5432
# nc: postgres-svc:5432: Connection timed out

# Another service → Backend (should be blocked ❌)
kubectl run test-pod --image=busybox --rm -it -- wget -qO- http://backend-svc:3000
# wget: download timed out

# Cleanup
kubectl delete networkpolicy --all
```

> **Important:** NetworkPolicy enforcement requires a CNI plugin that supports it. Minikube's default CNI does not enforce NetworkPolicies. Install Calico or Cilium for enforcement:
> ```bash
> minikube start --cni=calico
> # or
> minikube start --cni=cilium
> ```

---

## Quick Reference

```
Service types:
  ClusterIP      → internal only, stable virtual IP + DNS
  NodePort       → NodeIP:port (30000-32767), dev/testing
  LoadBalancer   → cloud LB with public IP, production

DNS pattern:
  <svc>.<namespace>.svc.cluster.local
  Short form (same namespace): <svc>

Ingress:
  Requires an Ingress controller (nginx, traefik, haproxy)
  Path-based:  /api → api-svc,  / → frontend-svc
  Host-based:  api.myapp.com → api-svc,  myapp.com → frontend-svc

NetworkPolicy rules:
  No policy on a pod     → all traffic allowed
  Any policy on a pod    → all unmatched traffic denied
  Multiple policies      → combined with OR (any match allows)
  Always allow DNS (53)  → or name resolution breaks

podSelector: {}          → matches ALL pods (use for default-deny)
podSelector: {matchLabels: {app: x}} → matches specific pods
```

| Service field | Purpose |
|---|---|
| `spec.selector` | Which pods get the traffic |
| `spec.ports[].port` | Port the Service exposes (ClusterIP port) |
| `spec.ports[].targetPort` | Port on the container |
| `spec.ports[].nodePort` | External node port (NodePort type only) |
| `spec.type` | ClusterIP / NodePort / LoadBalancer |

| NetworkPolicy field | Purpose |
|---|---|
| `spec.podSelector` | Which pods this policy applies to |
| `spec.policyTypes` | `Ingress`, `Egress`, or both |
| `spec.ingress[].from` | Allowed traffic sources |
| `spec.egress[].to` | Allowed traffic destinations |
| `podSelector` in from/to | Match pods by label |
| `namespaceSelector` in from/to | Match pods in specific namespaces |
| `ipBlock` in from/to | Match external CIDR ranges |
