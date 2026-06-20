# Day 26: Kubernetes Continuous Delivery with Argo CD, Minikube, Rolling Updates, and Helm

## Class Overview

**Course:** 30-Day Production DevOps Syllabus  
**Day:** 26  
**Topic:** Kubernetes CD with Argo CD  
**Duration:** 1 hour 30 minutes  
**Main Goal:** Deploy a Node.js application to Minikube using Argo CD and Helm, understand rolling updates, and parameterize image tags through Helm values.

---

## Learning Objectives

By the end of this class, students will be able to:

1. Explain how Argo CD works as a GitOps continuous delivery tool for Kubernetes.
2. Understand the difference between CI and CD in a Kubernetes-based workflow.
3. Configure Kubernetes deployment from Git using Argo CD.
4. Explain rolling updates in Kubernetes Deployments.
5. Understand the difference between `kubectl set image`, `kubectl apply`, and Helm-based deployment.
6. Install and use Helm locally.
7. Convert basic Kubernetes manifests into a Helm chart.
8. Parameterize a Docker image tag using `values.yaml`.
9. Deploy or upgrade a Kubernetes application using:

   ```bash
   helm upgrade --install
   ```

10. Understand when to use Helm and when raw Kubernetes manifests are enough.

---

## 1. Concept: Kubernetes Continuous Delivery

Continuous Delivery means automatically preparing and deploying application changes to an environment after the build and test stages are completed.

In Kubernetes, CD usually means:

1. A developer pushes code to GitHub.
2. CI pipeline builds and pushes a Docker image.
3. Kubernetes manifests or Helm values are updated with the new image tag.
4. Argo CD detects the Git change.
5. Argo CD syncs the Kubernetes cluster with the desired state stored in Git.
6. Kubernetes performs a rolling update.

---

## 2. CI vs CD in Kubernetes

| Area | CI | CD |
|---|---|---|
| Full form | Continuous Integration | Continuous Delivery / Deployment |
| Main job | Build, test, scan, package | Deploy application to environment |
| Common tools | Jenkins, GitHub Actions, GitLab CI | Argo CD, Flux, Spinnaker |
| Output | Docker image, test report, artifact | Running app in Kubernetes |
| Example | Build image and push to Docker Hub | Deploy image to Minikube using Argo CD |

### Simple flow

```text
Developer Push
     |
     v
Jenkins / CI Pipeline
     |
     v
Build Docker Image
     |
     v
Push Image to Registry
     |
     v
Update Git Manifest / Helm values.yaml
     |
     v
Argo CD Detects Git Change
     |
     v
Argo CD Syncs Kubernetes
     |
     v
New Version Runs in Minikube
```

---

## 3. What is Argo CD?

Argo CD is a GitOps continuous delivery tool for Kubernetes.

In a GitOps workflow, Git becomes the single source of truth. Instead of manually applying changes directly to the Kubernetes cluster, the desired state is stored in a Git repository.

Argo CD continuously compares:

- Desired state from Git
- Actual state inside Kubernetes

If there is a difference, Argo CD marks the application as **OutOfSync** and can sync the cluster to match Git.

---

## 4. Argo CD Architecture

### Main Components

| Component | Purpose |
|---|---|
| API Server | Exposes Argo CD UI, CLI, and API |
| Repository Server | Connects to Git repositories and renders manifests |
| Application Controller | Compares desired state and actual cluster state |
| Dex / SSO | Optional authentication integration |
| Redis | Caching layer used by Argo CD |

### Argo CD deployment flow

```text
Git Repository
    |
    v
Argo CD Repository Server
    |
    v
Manifest Rendered
    |
    v
Argo CD Application Controller
    |
    v
Kubernetes API Server
    |
    v
Pods / Services / Deployments Created
```

---

## 5. Why Use Argo CD Instead of Only Jenkins?

Jenkins can deploy to Kubernetes, but Argo CD provides a cleaner GitOps-based CD model.

| Feature | Jenkins CD | Argo CD GitOps |
|---|---|---|
| Deployment trigger | Pipeline stage | Git state change |
| Source of truth | Pipeline logic | Git repository |
| Kubernetes drift detection | Manual | Built in |
| Rollback visibility | Depends on pipeline | Git history and app history |
| UI for app health | Not native | Built in |
| Sync status | Manual scripting | Native |

### Recommended production pattern

Use:

```text
Jenkins for CI
Argo CD for CD
```

That means:

- Jenkins builds and pushes the image.
- Jenkins updates the image tag in Git.
- Argo CD deploys the application from Git.

---

## 6. Minikube Setup

Minikube is used in this class as the local Kubernetes cluster.

### Start Minikube

```bash
minikube start
```

### Check cluster status

```bash
minikube status
```

### Check nodes

```bash
kubectl get nodes
```

Expected output:

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   10m   v1.xx.x
```

---

## 7. Install Argo CD in Minikube

### Create namespace

```bash
kubectl create namespace argocd
```

### Install Argo CD

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Check Argo CD Pods

```bash
kubectl get pods -n argocd
```

Wait until all Pods are running:

```text
argocd-application-controller   Running
argocd-server                   Running
argocd-repo-server              Running
argocd-redis                    Running
```

---

## 8. Access Argo CD UI

### Port forward Argo CD server

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

### Get initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### Login details

```text
Username: admin
Password: <password-from-secret>
```

---

## 9. Install Argo CD CLI

### Linux installation

```bash
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

rm argocd-linux-amd64
```

### Verify installation

```bash
argocd version --client
```

### Login using CLI

```bash
argocd login localhost:8080
```

Because this is local and self-signed, you may need:

```bash
argocd login localhost:8080 --insecure
```

---

## 10. Kubernetes Deployment Strategy

A Kubernetes Deployment manages ReplicaSets and Pods.

When a new version of an application is deployed, Kubernetes does not usually delete all old Pods at once. Instead, it performs a rolling update.

### Rolling update meaning

A rolling update gradually replaces old Pods with new Pods.

Example:

```text
Current version: app:v1
New version:     app:v2

Step 1: Start one new v2 Pod
Step 2: Wait until v2 Pod is healthy
Step 3: Remove one old v1 Pod
Step 4: Repeat until all Pods run v2
```

This avoids downtime when configured correctly.

---

## 11. Rolling Update Configuration

Example Deployment strategy:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

### Explanation

| Field | Meaning |
|---|---|
| `type: RollingUpdate` | Kubernetes updates Pods gradually |
| `maxUnavailable: 1` | At most one Pod can be unavailable during update |
| `maxSurge: 1` | At most one extra Pod can be created during update |

### Example with replicas

If replicas are set to `3`:

```yaml
replicas: 3
```

During update, Kubernetes may temporarily run:

```text
3 old Pods + 1 new Pod = 4 Pods
```

Then it removes old Pods one by one.

---

## 12. Deployment Method 1: kubectl set image

The `kubectl set image` command updates the container image of an existing Kubernetes workload.

Example:

```bash
kubectl set image deployment/nodejs-app \
  nodejs-app=rohini/nodejs-app:v2
```

Check rollout status:

```bash
kubectl rollout status deployment/nodejs-app
```

Check rollout history:

```bash
kubectl rollout history deployment/nodejs-app
```

Rollback:

```bash
kubectl rollout undo deployment/nodejs-app
```

### When to use `kubectl set image`

Use it when:

- You need a quick manual update.
- You are testing locally.
- You are deploying from a simple CI pipeline.
- You do not need GitOps tracking.

### Limitation

The change is made directly in the cluster. Git may not know about the new state. This can create drift between Git and Kubernetes.

---

## 13. Deployment Method 2: kubectl apply

The `kubectl apply` command applies Kubernetes YAML files to the cluster.

Example:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

### When to use `kubectl apply`

Use it when:

- You have simple YAML files.
- You want direct manifest-based deployment.
- You are not using Helm.
- The application is small and has few configurable values.

### Limitation

When applications grow, raw manifests become repetitive. For example, you may need separate YAML files for:

```text
dev
staging
production
```

This is where Helm becomes useful.

---

## 14. Deployment Method 3: Helm

Helm is a package manager for Kubernetes.

A Helm chart contains Kubernetes templates and configurable values.

Instead of writing separate YAML files for each environment, Helm allows you to write templates and change values using `values.yaml`.

---

## 15. Helm Basic Concepts

| Helm Concept | Meaning |
|---|---|
| Chart | A package containing Kubernetes templates |
| Release | An installed instance of a chart |
| `values.yaml` | Default configuration values |
| Template | YAML file with variables |
| `helm install` | Install a chart for the first time |
| `helm upgrade` | Upgrade an existing release |
| `helm upgrade --install` | Install if missing, upgrade if already exists |

---

## 16. Install Helm

### Linux installation

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Verify Helm

```bash
helm version
```

---

## 17. Raw Kubernetes Manifests for Node.js App

Before converting to Helm, assume we have these basic manifests.

### `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
  labels:
    app: nodejs-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodejs-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: nodejs-app
    spec:
      containers:
        - name: nodejs-app
          image: rohini/nodejs-app:v1
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
```

### `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodejs-app-service
spec:
  type: NodePort
  selector:
    app: nodejs-app
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30080
```

---

## 18. Image Pull Policy: Always

For CI/CD pipelines, `imagePullPolicy: Always` is commonly used so that Kubernetes checks the registry whenever a Pod starts.

Example:

```yaml
imagePullPolicy: Always
```

### Important note

Even with `imagePullPolicy: Always`, it is better to use unique image tags such as:

```text
v1
v2
build-26
git-commit-sha
```

Avoid using only:

```text
latest
```

in production because it makes rollback and debugging harder.

Recommended CI/CD image tag examples:

```text
nodejs-app:1.0.0
nodejs-app:2026-05-24-001
nodejs-app:git-a1b2c3d
```

---

## 19. Convert Raw Manifests to Helm Chart

Create a Helm chart:

```bash
helm create nodejs-app
```

This creates:

```text
nodejs-app/
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── serviceaccount.yaml
    ├── hpa.yaml
    └── tests/
```

For this class, simplify the chart.

### Clean unnecessary templates

```bash
cd nodejs-app/templates

rm -f ingress.yaml hpa.yaml serviceaccount.yaml
rm -rf tests
```

Final structure:

```text
nodejs-app/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    └── service.yaml
```

---

## 20. Helm `Chart.yaml`

Edit `Chart.yaml`:

```yaml
apiVersion: v2
name: nodejs-app
description: A basic Node.js application Helm chart for Kubernetes CD with Argo CD
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### Explanation

| Field | Meaning |
|---|---|
| `apiVersion` | Helm chart API version |
| `name` | Chart name |
| `description` | Short chart description |
| `type` | Application chart |
| `version` | Helm chart version |
| `appVersion` | Application version |

---

## 21. Helm `values.yaml`

Edit `values.yaml`:

```yaml
replicaCount: 2

image:
  repository: rohini/nodejs-app
  tag: v1
  pullPolicy: Always

service:
  type: NodePort
  port: 80
  targetPort: 3000
  nodePort: 30080

container:
  port: 3000

strategy:
  type: RollingUpdate
  maxUnavailable: 1
  maxSurge: 1
```

### Why `values.yaml` is important

Instead of hardcoding image tag inside the Deployment template, we place the image tag in:

```yaml
image:
  tag: v1
```

Later, CI/CD can update only this value:

```yaml
image:
  tag: v2
```

Then Argo CD detects the Git change and deploys the new version.

---

## 22. Helm Deployment Template

Create or edit:

```text
templates/deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  strategy:
    type: {{ .Values.strategy.type }}
    rollingUpdate:
      maxUnavailable: {{ .Values.strategy.maxUnavailable }}
      maxSurge: {{ .Values.strategy.maxSurge }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.container.port }}
```

---

## 23. Helm Service Template

Create or edit:

```text
templates/service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Chart.Name }}-service
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Chart.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      nodePort: {{ .Values.service.nodePort }}
```

---

## 24. Validate Helm Chart

Run:

```bash
helm lint ./nodejs-app
```

Expected output:

```text
==> Linting ./nodejs-app
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

Render templates locally:

```bash
helm template nodejs-app ./nodejs-app
```

This shows the final Kubernetes YAML generated by Helm.

---

## 25. Deploy Helm Chart Manually First

Before connecting Argo CD, test the chart manually.

```bash
helm upgrade --install nodejs-app ./nodejs-app
```

### Explanation

| Part | Meaning |
|---|---|
| `helm upgrade` | Upgrade an existing release |
| `--install` | Install if the release does not already exist |
| `nodejs-app` | Release name |
| `./nodejs-app` | Chart path |

Check resources:

```bash
kubectl get pods
kubectl get svc
kubectl get deployment
```

Check rollout:

```bash
kubectl rollout status deployment/nodejs-app
```

Access app using Minikube:

```bash
minikube service nodejs-app-service
```

---

## 26. Upgrade Image Tag with Helm

Change image tag directly from CLI:

```bash
helm upgrade --install nodejs-app ./nodejs-app \
  --set image.tag=v2
```

Check rollout:

```bash
kubectl rollout status deployment/nodejs-app
```

Check image:

```bash
kubectl describe deployment nodejs-app | grep Image
```

Expected:

```text
Image: rohini/nodejs-app:v2
```

---

## 27. Git Repository Structure for Argo CD

Recommended repository structure:

```text
nodejs-k8s-gitops/
├── app/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
└── helm/
    └── nodejs-app/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── deployment.yaml
            └── service.yaml
```

Argo CD should point to:

```text
helm/nodejs-app
```

---

## 28. Push Helm Chart to GitHub

Initialize Git repository:

```bash
git init
git add .
git commit -m "Add Helm chart for Node.js app"
```

Add remote:

```bash
git remote add origin https://github.com/<username>/nodejs-k8s-gitops.git
```

Push:

```bash
git branch -M main
git push -u origin main
```

---

## 29. Create Argo CD Application Using CLI

Example:

```bash
argocd app create nodejs-app \
  --repo https://github.com/<username>/nodejs-k8s-gitops.git \
  --path helm/nodejs-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

### Explanation

| Option | Meaning |
|---|---|
| `nodejs-app` | Argo CD application name |
| `--repo` | Git repository URL |
| `--path` | Path to Helm chart in Git |
| `--dest-server` | Kubernetes cluster API |
| `--dest-namespace` | Target namespace |

---

## 30. Sync Argo CD Application

Manual sync:

```bash
argocd app sync nodejs-app
```

Check app status:

```bash
argocd app get nodejs-app
```

Expected:

```text
Health Status: Healthy
Sync Status: Synced
```

---

## 31. Enable Auto Sync

Auto sync means Argo CD automatically applies changes when Git changes.

```bash
argocd app set nodejs-app --sync-policy automated
```

Optional self-heal:

```bash
argocd app set nodejs-app --self-heal
```

Optional prune:

```bash
argocd app set nodejs-app --auto-prune
```

### Explanation

| Option | Meaning |
|---|---|
| `automated` | Sync automatically when Git changes |
| `self-heal` | Fix manual drift in cluster |
| `auto-prune` | Delete cluster resources removed from Git |

---

## 32. Declarative Argo CD Application Manifest

Instead of creating the app using CLI, you can define the Argo CD Application in YAML.

Create:

```text
argocd-app.yaml
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nodejs-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/<username>/nodejs-k8s-gitops.git
    targetRevision: main
    path: helm/nodejs-app
    helm:
      valueFiles:
        - values.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: default

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply it:

```bash
kubectl apply -f argocd-app.yaml
```

Check:

```bash
argocd app get nodejs-app
```

---

## 33. Argo CD with Helm Values

Argo CD can deploy Helm charts directly.

The important files are:

```text
Chart.yaml
values.yaml
templates/
```

When Argo CD sees a Helm chart, it renders the templates and applies the generated Kubernetes manifests to the cluster.

### Example value override in Argo CD CLI

```bash
argocd app set nodejs-app -p image.tag=v2
```

However, in a GitOps workflow, it is better to update the `values.yaml` file in Git.

Recommended GitOps approach:

```text
Update values.yaml
Commit change
Push to GitHub
Argo CD syncs automatically
```

---

## 34. CI/CD Flow with Jenkins and Argo CD

Even though this day focuses on Argo CD, Jenkins can still be used for CI.

### Recommended flow

```text
Developer pushes code
        |
        v
Jenkins pipeline starts
        |
        v
Jenkins builds Docker image
        |
        v
Jenkins pushes image to Docker Hub / ECR
        |
        v
Jenkins updates Helm values.yaml image tag
        |
        v
Jenkins commits and pushes values.yaml
        |
        v
Argo CD detects Git change
        |
        v
Argo CD deploys to Minikube
```

---

## 35. Example Jenkins Stage to Update Helm Image Tag

This is an example CI stage. It updates the Helm image tag in Git so that Argo CD can deploy it.

```groovy
stage('Update Helm Image Tag') {
    steps {
        sh '''
          git config user.email "jenkins@example.com"
          git config user.name "jenkins"

          sed -i "s/tag: .*/tag: ${BUILD_NUMBER}/" helm/nodejs-app/values.yaml

          git add helm/nodejs-app/values.yaml
          git commit -m "Update image tag to ${BUILD_NUMBER}"
          git push origin main
        '''
    }
}
```

### Important

This stage assumes:

1. Jenkins has permission to push to GitHub.
2. The Docker image was already built and pushed with tag `${BUILD_NUMBER}`.
3. Argo CD is watching the same Git repository.

---

## 36. Practical Lab

### Lab Goal

Deploy a Node.js app to Minikube using Argo CD and Helm.

### Required tools

Students need:

```text
Docker
Minikube
kubectl
Helm
Argo CD
GitHub account
```

---

## 37. Lab Part 1: Prepare Node.js App

Create project:

```bash
mkdir nodejs-k8s-gitops
cd nodejs-k8s-gitops
mkdir app
cd app
```

Create `package.json`:

```json
{
  "name": "nodejs-k8s-gitops",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

Create `server.js`:

```javascript
const express = require("express");

const app = express();
const port = process.env.PORT || 3000;

app.get("/", (req, res) => {
  res.send("Hello from Node.js app deployed by Argo CD and Helm - v1");
});

app.listen(port, () => {
  console.log(`App running on port ${port}`);
});
```

Create `Dockerfile`:

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

---

## 38. Lab Part 2: Build Docker Image

For Docker Hub:

```bash
docker build -t <dockerhub-username>/nodejs-app:v1 .
docker push <dockerhub-username>/nodejs-app:v1
```

For Minikube local image testing:

```bash
eval $(minikube docker-env)
docker build -t nodejs-app:v1 .
```

If you use local Minikube image, set image repository accordingly in `values.yaml`.

---

## 39. Lab Part 3: Create Helm Chart

Go back to root folder:

```bash
cd ..
mkdir helm
cd helm
helm create nodejs-app
```

Clean extra files:

```bash
cd nodejs-app/templates
rm -f ingress.yaml hpa.yaml serviceaccount.yaml
rm -rf tests
cd ../../..
```

Update:

```text
helm/nodejs-app/values.yaml
```

```yaml
replicaCount: 2

image:
  repository: <dockerhub-username>/nodejs-app
  tag: v1
  pullPolicy: Always

service:
  type: NodePort
  port: 80
  targetPort: 3000
  nodePort: 30080

container:
  port: 3000

strategy:
  type: RollingUpdate
  maxUnavailable: 1
  maxSurge: 1
```

Update:

```text
helm/nodejs-app/templates/deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  strategy:
    type: {{ .Values.strategy.type }}
    rollingUpdate:
      maxUnavailable: {{ .Values.strategy.maxUnavailable }}
      maxSurge: {{ .Values.strategy.maxSurge }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.container.port }}
```

Update:

```text
helm/nodejs-app/templates/service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Chart.Name }}-service
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Chart.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      nodePort: {{ .Values.service.nodePort }}
```

---

## 40. Lab Part 4: Test Helm Locally

```bash
helm lint ./helm/nodejs-app
```

```bash
helm template nodejs-app ./helm/nodejs-app
```

Deploy:

```bash
helm upgrade --install nodejs-app ./helm/nodejs-app
```

Check:

```bash
kubectl get pods
kubectl get svc
kubectl rollout status deployment/nodejs-app
```

Access:

```bash
minikube service nodejs-app-service
```

---

## 41. Lab Part 5: Push Chart to GitHub

```bash
git init
git add .
git commit -m "Add Node.js Helm chart for Argo CD deployment"
git branch -M main
git remote add origin https://github.com/<username>/nodejs-k8s-gitops.git
git push -u origin main
```

---

## 42. Lab Part 6: Create Argo CD Application

```bash
argocd app create nodejs-app \
  --repo https://github.com/<username>/nodejs-k8s-gitops.git \
  --path helm/nodejs-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

Sync:

```bash
argocd app sync nodejs-app
```

Check:

```bash
argocd app get nodejs-app
```

---

## 43. Lab Part 7: Test Rolling Update

Update Node.js message:

```javascript
res.send("Hello from Node.js app deployed by Argo CD and Helm - v2");
```

Build and push new image:

```bash
docker build -t <dockerhub-username>/nodejs-app:v2 ./app
docker push <dockerhub-username>/nodejs-app:v2
```

Update Helm values:

```yaml
image:
  repository: <dockerhub-username>/nodejs-app
  tag: v2
  pullPolicy: Always
```

Commit and push:

```bash
git add helm/nodejs-app/values.yaml
git commit -m "Update image tag to v2"
git push origin main
```

Argo CD will detect the change.

Manual sync if auto sync is not enabled:

```bash
argocd app sync nodejs-app
```

Watch rollout:

```bash
kubectl rollout status deployment/nodejs-app
```

Check Pods:

```bash
kubectl get pods -w
```

Access app:

```bash
minikube service nodejs-app-service
```

---

## 44. Assignment 26

### Task

Parameterize the image tag in Helm `values.yaml` and deploy the application using Argo CD with:

```bash
helm upgrade --install
```

### Requirements

Students must:

1. Create a Node.js application.
2. Dockerize the application.
3. Build and push image version `v1`.
4. Create a Helm chart.
5. Move the image repository and tag into `values.yaml`.
6. Use the tag in `deployment.yaml` template.
7. Deploy the app locally using:

   ```bash
   helm upgrade --install nodejs-app ./helm/nodejs-app
   ```

8. Push the Helm chart to GitHub.
9. Create an Argo CD application pointing to the Helm chart.
10. Build and push image version `v2`.
11. Update only the image tag in `values.yaml`.
12. Commit and push the change.
13. Sync using Argo CD.
14. Verify rolling update.
15. Submit screenshots or command outputs.

---

## 45. Assignment Submission Checklist

Students should submit:

- GitHub repository URL
- Screenshot of Argo CD application showing `Synced`
- Screenshot of Argo CD application showing `Healthy`
- Output of:

  ```bash
  kubectl get pods
  ```

- Output of:

  ```bash
  kubectl get svc
  ```

- Output of:

  ```bash
  kubectl rollout status deployment/nodejs-app
  ```

- Screenshot or output showing app version `v2`
- `values.yaml` showing parameterized image tag
- `deployment.yaml` showing Helm template usage

---

## 46. Common Errors and Fixes

### Error 1: Argo CD app is OutOfSync

Check:

```bash
argocd app get nodejs-app
```

Fix:

```bash
argocd app sync nodejs-app
```

---

### Error 2: ImagePullBackOff

Check Pod events:

```bash
kubectl describe pod <pod-name>
```

Common causes:

- Wrong image name
- Wrong image tag
- Private registry without image pull secret
- Image not pushed to registry

Fix:

```bash
docker push <dockerhub-username>/nodejs-app:v2
```

Then update `values.yaml`.

---

### Error 3: Helm template error

Run:

```bash
helm lint ./helm/nodejs-app
helm template nodejs-app ./helm/nodejs-app
```

Common causes:

- Wrong indentation
- Missing value in `values.yaml`
- Wrong Helm variable path
- Typo such as `.Value` instead of `.Values`

---

### Error 4: NodePort already allocated

If port `30080` is already used, change:

```yaml
nodePort: 30081
```

Then upgrade:

```bash
helm upgrade --install nodejs-app ./helm/nodejs-app
```

---

### Error 5: Argo CD cannot access GitHub repository

Check:

```bash
argocd repo list
```

For public repository, ensure the URL is correct.

For private repository, add repository credentials:

```bash
argocd repo add https://github.com/<username>/<repo>.git \
  --username <username> \
  --password <token>
```

---

## 47. Best Practices

### 1. Use Git as the source of truth

Avoid manually editing Kubernetes objects in production.

Bad:

```bash
kubectl edit deployment nodejs-app
```

Good:

```text
Update Git -> Argo CD syncs cluster
```

---

### 2. Use unique image tags

Avoid:

```text
latest
```

Better:

```text
v1.0.1
build-26
git-a1b2c3d
```

---

### 3. Keep Helm values clean

Good `values.yaml`:

```yaml
image:
  repository: rohini/nodejs-app
  tag: v2
```

Avoid hardcoding image tag inside template.

---

### 4. Validate before pushing

Run:

```bash
helm lint ./helm/nodejs-app
helm template nodejs-app ./helm/nodejs-app
```

---

### 5. Use Argo CD auto sync carefully

For learning and development, auto sync is useful.

For production, use controlled sync with:

- Pull request review
- Separate environment branches
- Manual approval
- Rollback plan

---

## 48. Interview Questions

### Q1. What is Argo CD?

Argo CD is a GitOps continuous delivery tool for Kubernetes. It watches a Git repository and syncs the Kubernetes cluster to match the desired state stored in Git.

### Q2. What is GitOps?

GitOps is a deployment approach where Git stores the desired infrastructure and application state. Tools like Argo CD automatically apply that desired state to Kubernetes.

### Q3. What is a rolling update?

A rolling update gradually replaces old Pods with new Pods to reduce downtime during deployment.

### Q4. What is Helm?

Helm is a Kubernetes package manager that uses charts, templates, and values files to deploy applications.

### Q5. What is `values.yaml`?

`values.yaml` stores configurable values for a Helm chart, such as image repository, image tag, replica count, service type, and ports.

### Q6. What is the difference between Helm and raw manifests?

Raw manifests are static YAML files. Helm charts are templated and reusable, making them better for configurable applications and multiple environments.

### Q7. What does `helm upgrade --install` do?

It upgrades an existing Helm release if it exists. If it does not exist, it installs the release.

### Q8. Why should image tags be parameterized?

Because CI/CD pipelines can update only the image tag without rewriting the entire Kubernetes manifest.

### Q9. Why should production avoid the `latest` tag?

Because `latest` makes rollbacks, debugging, and version tracking difficult.

### Q10. What is the difference between Jenkins and Argo CD?

Jenkins is commonly used for CI tasks such as build and test. Argo CD is used for GitOps-based Kubernetes deployment.

---

## 49. Class Summary

In this class, students learned how Kubernetes continuous delivery works with Argo CD and Helm. They installed Argo CD in Minikube, understood rolling updates, compared different deployment strategies, converted raw Kubernetes manifests into a Helm chart, parameterized the image tag in `values.yaml`, and deployed the application using both Helm and Argo CD.

The most important concept is:

```text
Jenkins builds the image.
Git stores the desired deployment state.
Argo CD deploys the desired state to Kubernetes.
Helm makes the deployment reusable and configurable.
```

---

## 50. References

- Argo CD Helm documentation: https://argo-cd.readthedocs.io/en/latest/user-guide/helm/
- Argo CD declarative setup: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/
- Helm upgrade documentation: https://helm.sh/docs/helm/helm_upgrade/
- Helm charts documentation: https://helm.sh/docs/topics/charts/
- Helm values files documentation: https://helm.sh/docs/chart_template_guide/values_files/
- Kubernetes rolling update tutorial: https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/
- Kubernetes Deployments documentation: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Kubernetes image documentation: https://kubernetes.io/docs/concepts/containers/images/
- kubectl set image documentation: https://kubernetes.io/docs/reference/kubectl/generated/kubectl_set/kubectl_set_image/
