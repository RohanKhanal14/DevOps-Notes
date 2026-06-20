# Production-grade DevSecOps CI/CD pipeline

## Jenkins Controller-Agent + GitHub + SonarQube + Trivy + OWASP ZAP + Docker/ECR + Argo CD + EKS/Minikube + Prometheus/Grafana/Loki

Your uploaded Day 29 project already follows a strong DevSecOps base: Jenkins, SonarQube, Trivy FS scan, Trivy image scan, ECR, EKS, IAM/RBAC, and production enhancements such as GitOps, image digests, signed images, policy-as-code, Helm/Kustomize, Argo Rollouts, and observability. I will build on that structure and convert it into a complete production-grade architecture for a **three-tier CRUD app: ReactJS frontend, Node.js backend, and MongoDB**. 

---

# 1. Final target architecture

For production, I recommend this split:

| Layer                         | Tool / AWS Service                                                            | Purpose                                                        |
| ----------------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------------------- |
| Source control                | GitHub                                                                        | Application repo and GitOps repo                               |
| CI orchestrator               | Jenkins controller + SSH/build agents                                         | Build, test, scan, package, push image, update GitOps repo     |
| SAST                          | SonarQube                                                                     | Code quality, vulnerabilities, bugs, security hotspots         |
| Dependency/container/IaC scan | Trivy                                                                         | Filesystem, image, secret, config, and Kubernetes/IaC scanning |
| DAST                          | OWASP ZAP                                                                     | Runtime web/API security testing                               |
| Image registry                | Amazon ECR preferred; Docker Hub for lab                                      | Private, IAM-integrated image storage                          |
| Kubernetes                    | EKS for production; Minikube for local lab                                    | Container orchestration                                        |
| CD/GitOps                     | Argo CD                                                                       | Pull-based deployment, drift detection, rollback               |
| Packaging                     | Helm                                                                          | Environment-specific Kubernetes release management             |
| Database                      | Amazon DocumentDB / MongoDB Atlas for production; MongoDB StatefulSet for lab | Persistent database tier                                       |
| Monitoring                    | Prometheus + Grafana                                                          | Metrics, dashboards, alerts                                    |
| Logging                       | Loki + Promtail / Grafana Alloy                                               | Centralized pod logs                                           |
| Alerting                      | Alertmanager + Grafana alerts                                                 | Slack/email/on-call notifications                              |
| Secrets                       | AWS Secrets Manager + External Secrets Operator                               | Secret rotation and no hardcoded credentials                   |
| Ingress                       | AWS Load Balancer Controller + ALB                                            | HTTPS routing to frontend/backend                              |
| DNS/TLS                       | Route 53 + ACM                                                                | Friendly domain and managed certificates                       |

For production, Jenkins should **not directly deploy using `kubectl apply`**. Jenkins should build and validate artifacts, push images to ECR, update a GitOps repository, and then Argo CD should pull the desired state into Kubernetes. Argo CD is designed for declarative GitOps delivery where deployments track Git branches, tags, or specific commits, making Git the source of truth. ([Argo CD][1])

---

# 2. High-level architecture diagram

```mermaid
flowchart LR
    DEV[Developer] --> GHAPP[GitHub App Repo]

    GHAPP -->|Webhook| JENKINS[Jenkins Controller]
    JENKINS --> AGENT[Jenkins Build Agent]

    AGENT -->|Checkout code| GHAPP
    AGENT -->|npm test / lint| TEST[Test Stage]
    AGENT -->|SAST| SONAR[SonarQube]
    AGENT -->|FS + Secret + IaC Scan| TRIVY[Trivy]
    AGENT -->|docker build| DOCKER[Docker Build]
    DOCKER -->|Image Scan| TRIVYIMG[Trivy Image Scan]
    DOCKER -->|Push signed image| ECR[Amazon ECR]

    AGENT -->|Update image tag/digest| GITOPS[GitHub GitOps Repo]

    GITOPS -->|Pull desired state| ARGO[Argo CD]
    ARGO -->|Sync Helm release| EKS[Amazon EKS / Minikube]

    EKS --> FE[React Frontend Pods]
    EKS --> API[Node.js API Pods]
    API --> MDB[(MongoDB / DocumentDB)]

    EKS --> PROM[Prometheus]
    EKS --> LOKI[Loki]
    PROM --> GRAFANA[Grafana]
    LOKI --> GRAFANA

    ZAP[OWASP ZAP] -->|DAST scan staging URL| ALB[AWS ALB / Ingress]
    ALB --> FE
    ALB --> API

    PROM --> ALERT[Alertmanager]
    GRAFANA --> ALERT
```

---

# 3. CI/CD flow diagram

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GitHub as GitHub App Repo
    participant Jenkins as Jenkins Controller
    participant Agent as Jenkins Agent
    participant Sonar as SonarQube
    participant ECR as Amazon ECR
    participant GitOps as GitOps Repo
    participant Argo as Argo CD
    participant K8s as EKS/Minikube
    participant Zap as OWASP ZAP

    Dev->>GitHub: Push feature branch / PR
    GitHub->>Jenkins: Webhook trigger
    Jenkins->>Agent: Run pipeline on build agent
    Agent->>GitHub: Checkout source
    Agent->>Agent: npm install, lint, unit tests
    Agent->>Sonar: SAST + quality gate
    Agent->>Agent: Trivy filesystem + secret scan
    Agent->>Agent: Docker build frontend/backend
    Agent->>Agent: Trivy image scan
    Agent->>ECR: Push immutable image tags
    Agent->>GitOps: Update Helm values with image tag/digest
    Argo->>GitOps: Detect Git change
    Argo->>K8s: Sync desired state
    K8s->>K8s: Rolling update + HPA scaling
    Zap->>K8s: DAST scan staging endpoint
```

---

# 4. Recommended repository structure

Use **two repositories**.

## 4.1 Application repo

```text
three-tier-crud-app/
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── package.json
│   └── src/
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   ├── src/
│   ├── tests/
│   └── sonar-project.properties
├── docker-compose.yml
├── Jenkinsfile
├── .trivyignore
├── .dockerignore
└── README.md
```

## 4.2 GitOps repo

```text
three-tier-gitops/
├── charts/
│   └── three-tier-app/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       ├── values-staging.yaml
│       ├── values-prod.yaml
│       └── templates/
│           ├── frontend-deployment.yaml
│           ├── frontend-service.yaml
│           ├── backend-deployment.yaml
│           ├── backend-service.yaml
│           ├── ingress.yaml
│           ├── hpa-frontend.yaml
│           ├── hpa-backend.yaml
│           ├── pdb-frontend.yaml
│           ├── pdb-backend.yaml
│           ├── networkpolicy.yaml
│           ├── servicemonitor.yaml
│           └── externalsecret.yaml
└── argocd/
    ├── app-dev.yaml
    ├── app-staging.yaml
    └── app-prod.yaml
```

**Why two repos?**

Application repo stores source code and CI logic. GitOps repo stores desired Kubernetes state. This separation is cleaner, auditable, and safer because Jenkins cannot silently change the live cluster; it must commit a visible Git change that Argo CD applies.

---

# 5. Production environment design

## Option A: EKS production design

Use this for real production.

```text
VPC
├── Public subnets
│   └── ALB / NAT Gateway
├── Private subnets
│   ├── EKS worker nodes
│   ├── Jenkins agent if self-hosted
│   └── SonarQube if VM-based
└── Isolated/private DB subnets
    └── DocumentDB / MongoDB
```

Production EKS should follow AWS EKS security best practices: private worker nodes, least-privilege IAM, RBAC, secrets protection, network controls, and day-2 operational practices. AWS maintains an EKS Best Practices Guide specifically for security and operations. ([AWS Documentation][2])

## Option B: Minikube lab design

Use this for local teaching and testing.

```text
Laptop / VM
├── Minikube
├── Argo CD
├── Prometheus
├── Grafana
├── Loki
├── MongoDB StatefulSet
├── Frontend Deployment
└── Backend Deployment
```

Use Minikube for learning, but do not call it production. It is good for validating Kubernetes manifests, Helm charts, Argo CD sync, HPA basics, and local app behavior.

---

# 6. Jenkins master-slave architecture

Modern Jenkins terminology is **controller-agent**, but many teams still say master-slave. The production pattern is:

```text
Jenkins Controller
├── Runs UI
├── Stores job configuration
├── Orchestrates pipelines
├── Does not run heavy builds
└── Connects to agents

Jenkins Agent
├── Runs builds
├── Has Docker, Node.js, Trivy, AWS CLI, kubectl, Helm
├── Uses IAM role for AWS access
└── Can be replaced safely
```

Jenkins officially supports using agents to distribute build workloads, and SSH-based agents use credentials configured in Jenkins to connect to build nodes. ([Jenkins][3])

## Why not build on the controller?

Because the controller holds sensitive configuration, credentials, job metadata, and plugin state. If a build script is compromised and runs on the controller, the blast radius is much higher. Build workloads should run on agents.

---

# 7. AWS infrastructure setup

## 7.1 Required AWS services

Use these AWS services for production:

| AWS Service     | Purpose                                                  |
| --------------- | -------------------------------------------------------- |
| EC2             | Jenkins controller, Jenkins agent, SonarQube if VM-based |
| IAM             | Jenkins agent role, EKS access, ECR permissions          |
| ECR             | Private Docker image registry                            |
| EKS             | Kubernetes cluster                                       |
| VPC             | Network isolation                                        |
| ALB             | Public HTTPS entry point                                 |
| ACM             | TLS certificate                                          |
| Route 53        | DNS                                                      |
| Secrets Manager | Application/database secrets                             |
| CloudWatch      | AWS-native logs/metrics backup                           |
| DocumentDB      | Managed MongoDB-compatible database                      |
| S3              | Jenkins backup, artifacts, Terraform state               |
| KMS             | Encryption keys                                          |
| SNS             | Alert routing if needed                                  |

---

# 8. Jenkins controller installation on EC2

## 8.1 EC2 sizing

For a small team:

```text
AMI: Ubuntu 24.04 LTS
Instance type: t3.medium minimum, t3.large recommended
Disk: 50 GB gp3
Subnet: private subnet preferred
Access: via VPN/bastion/SSM
```

For a lab, you can expose Jenkins on `8080` from your IP only. For production, put Jenkins behind an internal ALB with HTTPS.

## 8.2 Security group

```text
Inbound:
- 22 from your IP or bastion only
- 8080 from your IP/internal ALB only

Outbound:
- 443 to internet or NAT for plugin downloads
- 22 to Jenkins agents
```

## 8.3 Install Java and Jenkins

```bash
sudo apt update
sudo apt install -y fontconfig openjdk-21-jdk curl gnupg

java -version

sudo mkdir -p /usr/share/keyrings

curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins

sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

## 8.4 Production hardening

Do this after installation:

```bash
sudo systemctl edit jenkins
```

Add:

```ini
[Service]
Environment="JAVA_OPTS=-Duser.timezone=Asia/Kathmandu -Djenkins.install.runSetupWizard=false"
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

Best practices:

| Practice                      | Why                                                  |
| ----------------------------- | ---------------------------------------------------- |
| Put Jenkins behind HTTPS      | Protect credentials/session cookies                  |
| Use SSO/OIDC                  | Centralized identity and MFA                         |
| Disable anonymous access      | Prevent unauthenticated discovery                    |
| Use role-based access control | Developers should not manage global Jenkins settings |
| Backup `/var/lib/jenkins`     | Jenkins state lives there                            |
| Keep plugins minimal          | Every plugin increases attack surface                |
| Use agents for builds         | Protect controller from untrusted build code         |

---

# 9. Jenkins agent installation

## 9.1 EC2 sizing

```text
AMI: Ubuntu 24.04 LTS
Instance type: t3.large or c7i.large
Disk: 80–100 GB gp3
Subnet: private subnet
IAM role: jenkins-agent-role
```

## 9.2 Install tools

```bash
sudo apt update
sudo apt install -y \
  openjdk-21-jdk \
  git \
  curl \
  wget \
  unzip \
  jq \
  ca-certificates \
  gnupg \
  lsb-release
```

## 9.3 Create Jenkins user

```bash
sudo useradd -m -s /bin/bash jenkins
sudo mkdir -p /home/jenkins/.ssh
sudo chown -R jenkins:jenkins /home/jenkins/.ssh
sudo chmod 700 /home/jenkins/.ssh
```

## 9.4 Install Docker

```bash
sudo apt remove docker docker-engine docker.io containerd runc -y || true
sudo apt update
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker jenkins
sudo systemctl enable docker
sudo systemctl start docker
```

## 9.5 Install Node.js 20 LTS

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

node -v
npm -v
```

## 9.6 Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install

aws --version
```

## 9.7 Install kubectl

For EKS, match the Kubernetes version.

```bash
curl -LO "https://dl.k8s.io/release/v1.32.0/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

kubectl version --client
```

## 9.8 Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

## 9.9 Install Trivy

```bash
wget https://github.com/aquasecurity/trivy/releases/download/v0.67.2/trivy_0.67.2_Linux-64bit.deb
sudo dpkg -i trivy_0.67.2_Linux-64bit.deb
trivy --version
```

Trivy is suitable here because it scans container images, filesystems, Git repositories, Kubernetes configurations, secrets, and misconfigurations. ([Trivy][4])

---

# 10. Connect Jenkins controller to agent

On Jenkins controller:

```bash
sudo su - jenkins
ssh-keygen -t ed25519 -f ~/.ssh/jenkins-agent-key -C "jenkins-agent-access"
cat ~/.ssh/jenkins-agent-key.pub
```

On Jenkins agent:

```bash
sudo -i -u jenkins
mkdir -p ~/.ssh
vim ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

From controller:

```bash
sudo -u jenkins ssh-keyscan <agent-private-ip> | \
  sudo -u jenkins tee -a /var/lib/jenkins/.ssh/known_hosts

sudo -u jenkins ssh -i /var/lib/jenkins/.ssh/jenkins-agent-key \
  jenkins@<agent-private-ip> hostname
```

In Jenkins UI:

```text
Manage Jenkins
→ Nodes
→ New Node
→ Permanent Agent
```

Use:

```text
Name: docker-node-trivy-aws
Remote root directory: /home/jenkins
Labels: docker nodejs trivy aws helm kubectl
Launch method: SSH
Host: <agent-private-ip>
Credentials: SSH username with private key
Host key verification: Known hosts file verification
Executors: 1 or 2
```

**Why one or two executors only?**
Docker builds, Trivy scans, and Node builds are CPU/disk intensive. Too many parallel executors on one agent causes slow builds, image layer corruption risk, and disk pressure.

---

# 11. SonarQube setup

For production, use one of these:

| Option                            | Recommendation                                     |
| --------------------------------- | -------------------------------------------------- |
| SonarQube on EC2 + RDS PostgreSQL | Good production baseline                           |
| SonarQube Data Center Edition     | Enterprise/high availability                       |
| SonarCloud                        | Simplest SaaS option                               |
| SonarQube inside Kubernetes       | Possible, but needs careful persistence and sizing |

## 11.1 Jenkins integration

Install Jenkins plugins:

```text
SonarQube Scanner
Pipeline
Credentials Binding
GitHub Branch Source
Docker Pipeline
```

In Jenkins:

```text
Manage Jenkins
→ System
→ SonarQube servers
```

Add:

```text
Name: sonarqube-prod
Server URL: https://sonar.example.com
Token: sonarqube-token
```

Use `waitForQualityGate abortPipeline: true` so Jenkins fails when SonarQube quality gate is not green. Jenkins documents that `waitForQualityGate` can abort a pipeline when the quality gate fails. ([Jenkins][5])

---

# 12. GitHub setup

## 12.1 Branching model

Use this:

```text
feature/* → pull request → develop
develop → auto deploy to dev
release/* → auto deploy to staging
main → manual approval → production
```

## 12.2 Branch protection

Enable:

```text
Require pull request before merge
Require status checks
Require SonarQube quality gate
Require Jenkins build success
Require code owner review
Disallow force push
Disallow direct push to main
Require signed commits where possible
```

## 12.3 Webhook

GitHub repo:

```text
Settings
→ Webhooks
→ Add webhook
```

```text
Payload URL: https://jenkins.example.com/github-webhook/
Content type: application/json
Secret: strong random secret
Events: push and pull_request
```

---

# 13. ECR setup

## 13.1 Create repositories

Create one repo per service:

```bash
aws ecr create-repository \
  --repository-name three-tier/frontend \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true

aws ecr create-repository \
  --repository-name three-tier/backend \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true
```

**Why ECR over Docker Hub for production?**

| ECR                   | Docker Hub                          |
| --------------------- | ----------------------------------- |
| IAM-native            | Token/password-based                |
| Private by default    | Public/private mix                  |
| Same AWS network path | External dependency                 |
| Lifecycle policies    | Available but less AWS-integrated   |
| Better for EKS        | Needs imagePullSecret unless public |

Use Docker Hub only for teaching labs or public images.

---

# 14. Jenkins agent IAM role

Attach IAM role to Jenkins agent EC2:

```text
Role name: jenkins-agent-role
Trusted entity: EC2
```

Minimal ECR policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRAuth",
      "Effect": "Allow",
      "Action": ["ecr:GetAuthorizationToken"],
      "Resource": "*"
    },
    {
      "Sid": "ECRPushPull",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:BatchGetImage",
        "ecr:DescribeImages"
      ],
      "Resource": [
        "arn:aws:ecr:<region>:<account-id>:repository/three-tier/frontend",
        "arn:aws:ecr:<region>:<account-id>:repository/three-tier/backend"
      ]
    }
  ]
}
```

**Best practice:** avoid AWS access keys in Jenkins. EC2 IAM role is safer because credentials are short-lived and automatically rotated.

---

# 15. Dockerfiles

## 15.1 Backend Dockerfile

`backend/Dockerfile`

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=deps /app/node_modules ./node_modules
COPY src ./src
COPY package*.json ./

USER appuser

EXPOSE 5000

CMD ["node", "src/server.js"]
```

**Why this is better:**

| Practice             | Reason                           |
| -------------------- | -------------------------------- |
| `npm ci`             | Reproducible builds              |
| Alpine/minimal image | Smaller attack surface           |
| Non-root user        | Reduces container privilege risk |
| No `.env` copied     | Prevents secret leakage          |
| Multi-stage          | Keeps image clean                |

## 15.2 Frontend Dockerfile

`frontend/Dockerfile`

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.27-alpine AS runner

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]
```

`frontend/nginx.conf`

```nginx
server {
    listen 8080;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /health {
        access_log off;
        return 200 "healthy\n";
    }
}
```

---

# 16. `.dockerignore`

Create this in both `frontend/` and `backend/`:

```text
node_modules
npm-debug.log
.git
.gitignore
Dockerfile
.dockerignore
.env
.env.*
coverage
dist
build
README.md
```

**Why:** Docker build context should be small and should never include secrets, local dependencies, or Git metadata.

---

# 17. Kubernetes/Helm design

Use Helm for production because it gives repeatable, parameterized deployments. Your uploaded file also recommends Helm/Kustomize as a production enhancement because raw YAML becomes hard to manage across environments. 

## 17.1 Helm values

`charts/three-tier-app/values.yaml`

```yaml
global:
  environment: dev
  imagePullPolicy: IfNotPresent

frontend:
  name: frontend
  replicaCount: 2
  image:
    repository: ""
    tag: ""
  service:
    port: 80
    targetPort: 8080
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 6
    targetCPUUtilizationPercentage: 60

backend:
  name: backend
  replicaCount: 2
  image:
    repository: ""
    tag: ""
  service:
    port: 5000
    targetPort: 5000
  env:
    mongoDatabase: crudapp
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 60

ingress:
  enabled: true
  className: alb
  host: crud.example.com
  certificateArn: ""
```

---

# 18. Backend Deployment template

`templates/backend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-backend
  labels:
    app: {{ .Release.Name }}-backend
spec:
  replicas: {{ .Values.backend.replicaCount }}
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  selector:
    matchLabels:
      app: {{ .Release.Name }}-backend
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "5000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: {{ .Release.Name }}-backend
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: backend
          image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
          imagePullPolicy: {{ .Values.global.imagePullPolicy }}
          ports:
            - containerPort: 5000
          env:
            - name: NODE_ENV
              value: "production"
            - name: MONGO_DATABASE
              value: {{ .Values.backend.env.mongoDatabase | quote }}
            - name: MONGO_URI
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-mongo-secret
                  key: mongo-uri
          readinessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 30
            periodSeconds: 20
            failureThreshold: 3
          resources:
{{ toYaml .Values.backend.resources | indent 12 }}
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false
            capabilities:
              drop:
                - ALL
```

Kubernetes Deployments support rolling updates, gradually replacing old Pods with new Pods; `maxUnavailable` and `maxSurge` control the rollout behavior. ([Kubernetes][6])

---

# 19. Backend Service

`templates/backend-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-backend
  labels:
    app: {{ .Release.Name }}-backend
spec:
  type: ClusterIP
  selector:
    app: {{ .Release.Name }}-backend
  ports:
    - port: {{ .Values.backend.service.port }}
      targetPort: {{ .Values.backend.service.targetPort }}
      protocol: TCP
```

**Why ClusterIP:** backend should not be directly public. Frontend or Ingress should route to it internally.

---

# 20. Frontend Deployment

`templates/frontend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-frontend
  labels:
    app: {{ .Release.Name }}-frontend
spec:
  replicas: {{ .Values.frontend.replicaCount }}
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  selector:
    matchLabels:
      app: {{ .Release.Name }}-frontend
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-frontend
    spec:
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: frontend
          image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
          imagePullPolicy: {{ .Values.global.imagePullPolicy }}
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 20
          resources:
{{ toYaml .Values.frontend.resources | indent 12 }}
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
```

---

# 21. Horizontal Pod Autoscaling

`templates/hpa-backend.yaml`

```yaml
{{- if .Values.backend.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ .Release.Name }}-backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ .Release.Name }}-backend
  minReplicas: {{ .Values.backend.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.backend.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.backend.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
```

The Kubernetes Horizontal Pod Autoscaler adjusts desired replicas based on observed metrics like CPU, memory, or custom metrics. ([Kubernetes][7])

---

# 22. PodDisruptionBudget

`templates/pdb-backend.yaml`

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ .Release.Name }}-backend
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: {{ .Release.Name }}-backend
```

**Why:** During node upgrades or voluntary disruptions, Kubernetes keeps at least one backend pod available.

---

# 23. NetworkPolicy

`templates/networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-only-from-frontend
spec:
  podSelector:
    matchLabels:
      app: {{ .Release.Name }}-backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: {{ .Release.Name }}-frontend
      ports:
        - protocol: TCP
          port: 5000
```

**Why:** frontend can call backend, but random pods cannot. For this to work, your CNI must enforce NetworkPolicy.

---

# 24. Ingress with AWS ALB

`templates/ingress.yaml`

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
  annotations:
    kubernetes.io/ingress.class: {{ .Values.ingress.className | quote }}
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: {{ .Values.ingress.certificateArn | quote }}
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
spec:
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}-backend
                port:
                  number: 5000
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}-frontend
                port:
                  number: 80
{{- end }}
```

**Why ALB instead of NodePort:** NodePort is okay for labs. In production, ALB gives TLS, host/path routing, AWS integration, security groups, WAF integration, and cleaner DNS.

---

# 25. Argo CD installation

For EKS or Minikube:

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Expose locally for testing:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Get initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo
```

Production hardening:

| Setting          | Production recommendation                       |
| ---------------- | ----------------------------------------------- |
| Argo CD access   | Behind internal ALB/VPN/SSO                     |
| Auth             | SSO/OIDC                                        |
| Admin user       | Disable after SSO                               |
| Projects         | Use AppProjects per team/environment            |
| Sync             | Auto-sync dev/staging; manual approval for prod |
| RBAC             | Developers can view/sync only their apps        |
| Repo credentials | GitHub App or deploy key                        |
| Secrets          | Use External Secrets or Sealed Secrets          |

---

# 26. Argo CD Application

`argocd/app-staging.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: three-tier-staging
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/<org>/three-tier-gitops.git
    targetRevision: main
    path: charts/three-tier-app
    helm:
      valueFiles:
        - values-staging.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: three-tier-staging

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Argo CD supports sync options and automated sync policy to control how desired state is applied to a cluster. ([Argo CD][8])

---

# 27. Jenkins pipeline design

Jenkins should perform CI and GitOps update:

```text
1. Checkout app repo
2. Install dependencies
3. Lint frontend/backend
4. Unit tests
5. SonarQube SAST + quality gate
6. Trivy filesystem scan
7. Docker build frontend/backend
8. Trivy image scan
9. Login to ECR
10. Push images to ECR
11. Get image digest
12. Update GitOps Helm values
13. Commit and push GitOps repo
14. Argo CD syncs automatically
15. Run OWASP ZAP against staging URL
16. Notify result
```

OWASP ZAP Baseline Scan is available through ZAP Docker images and performs spidering plus passive scanning; the full scan can perform active testing and should be used carefully, usually against staging or security test environments. ([ZAP][9])

---

# 28. Jenkins credentials

Create these in Jenkins:

| Credential ID           | Type                            | Purpose             |
| ----------------------- | ------------------------------- | ------------------- |
| `github-app-repo-token` | Username/password or GitHub App | Checkout app repo   |
| `github-gitops-token`   | Username/password or GitHub App | Push GitOps updates |
| `sonarqube-token`       | Secret text                     | Sonar analysis      |
| `zap-baseline-config`   | Secret file optional            | ZAP rules/config    |
| `dockerhub-creds`       | Username/password optional      | Docker Hub lab only |

Use IAM role for AWS/ECR instead of storing AWS keys in Jenkins.

---

# 29. Complete Jenkinsfile

```groovy
pipeline {
  agent { label 'docker nodejs trivy aws helm kubectl' }

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    timeout(time: 60, unit: 'MINUTES')
  }

  environment {
    AWS_REGION = 'ap-south-1'
    AWS_ACCOUNT_ID = '123456789012'

    ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

    FRONTEND_REPO = "three-tier/frontend"
    BACKEND_REPO  = "three-tier/backend"

    FRONTEND_IMAGE = "${ECR_REGISTRY}/${FRONTEND_REPO}"
    BACKEND_IMAGE  = "${ECR_REGISTRY}/${BACKEND_REPO}"

    IMAGE_TAG = "${BUILD_NUMBER}-${GIT_COMMIT_SHORT}"

    SONARQUBE_ENV = 'sonarqube-prod'

    GITOPS_REPO = 'https://github.com/<org>/three-tier-gitops.git'
    GITOPS_BRANCH = 'main'
    GITOPS_VALUES_FILE = 'charts/three-tier-app/values-staging.yaml'

    STAGING_URL = 'https://staging-crud.example.com'
  }

  stages {
    stage('Prepare') {
      steps {
        script {
          env.GIT_COMMIT_SHORT = sh(
            script: "git rev-parse --short=8 HEAD",
            returnStdout: true
          ).trim()

          env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
        }

        sh '''
          echo "Build number: ${BUILD_NUMBER}"
          echo "Git commit: ${GIT_COMMIT}"
          echo "Image tag: ${IMAGE_TAG}"
          node -v
          npm -v
          docker --version
          trivy --version
          aws --version
        '''
      }
    }

    stage('Install Dependencies') {
      parallel {
        stage('Frontend npm ci') {
          steps {
            dir('frontend') {
              sh 'npm ci'
            }
          }
        }

        stage('Backend npm ci') {
          steps {
            dir('backend') {
              sh 'npm ci'
            }
          }
        }
      }
    }

    stage('Lint') {
      parallel {
        stage('Frontend Lint') {
          steps {
            dir('frontend') {
              sh 'npm run lint'
            }
          }
        }

        stage('Backend Lint') {
          steps {
            dir('backend') {
              sh 'npm run lint'
            }
          }
        }
      }
    }

    stage('Unit Tests') {
      parallel {
        stage('Frontend Tests') {
          steps {
            dir('frontend') {
              sh 'npm test -- --watch=false --coverage || npm test -- --coverage'
            }
          }
        }

        stage('Backend Tests') {
          steps {
            dir('backend') {
              sh 'npm test -- --coverage'
            }
          }
        }
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/junit.xml'
        }
      }
    }

    stage('SonarQube SAST') {
      steps {
        withSonarQubeEnv("${SONARQUBE_ENV}") {
          withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
            sh '''
              sonar-scanner \
                -Dsonar.projectKey=three-tier-crud-app \
                -Dsonar.projectName=three-tier-crud-app \
                -Dsonar.sources=frontend/src,backend/src \
                -Dsonar.tests=backend/tests \
                -Dsonar.javascript.lcov.reportPaths=frontend/coverage/lcov.info,backend/coverage/lcov.info \
                -Dsonar.token=${SONAR_TOKEN}
            '''
          }
        }
      }
    }

    stage('Sonar Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Trivy Filesystem Scan') {
      steps {
        sh '''
          trivy fs \
            --exit-code 1 \
            --severity HIGH,CRITICAL \
            --scanners vuln,secret,misconfig \
            .
        '''
      }
    }

    stage('Docker Build') {
      parallel {
        stage('Build Frontend Image') {
          steps {
            sh '''
              docker build \
                --platform linux/amd64 \
                -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                -t ${FRONTEND_IMAGE}:latest \
                ./frontend
            '''
          }
        }

        stage('Build Backend Image') {
          steps {
            sh '''
              docker build \
                --platform linux/amd64 \
                -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                -t ${BACKEND_IMAGE}:latest \
                ./backend
            '''
          }
        }
      }
    }

    stage('Trivy Image Scan') {
      parallel {
        stage('Scan Frontend Image') {
          steps {
            sh '''
              trivy image \
                --exit-code 1 \
                --severity HIGH,CRITICAL \
                ${FRONTEND_IMAGE}:${IMAGE_TAG}
            '''
          }
        }

        stage('Scan Backend Image') {
          steps {
            sh '''
              trivy image \
                --exit-code 1 \
                --severity HIGH,CRITICAL \
                ${BACKEND_IMAGE}:${IMAGE_TAG}
            '''
          }
        }
      }
    }

    stage('ECR Login') {
      steps {
        sh '''
          aws ecr get-login-password --region ${AWS_REGION} | \
          docker login --username AWS --password-stdin ${ECR_REGISTRY}
        '''
      }
    }

    stage('Push Images') {
      steps {
        sh '''
          docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
          docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
        '''
      }
    }

    stage('Resolve Image Digests') {
      steps {
        script {
          env.FRONTEND_DIGEST = sh(
            script: """
              aws ecr describe-images \
                --repository-name ${FRONTEND_REPO} \
                --image-ids imageTag=${IMAGE_TAG} \
                --region ${AWS_REGION} \
                --query 'imageDetails[0].imageDigest' \
                --output text
            """,
            returnStdout: true
          ).trim()

          env.BACKEND_DIGEST = sh(
            script: """
              aws ecr describe-images \
                --repository-name ${BACKEND_REPO} \
                --image-ids imageTag=${IMAGE_TAG} \
                --region ${AWS_REGION} \
                --query 'imageDetails[0].imageDigest' \
                --output text
            """,
            returnStdout: true
          ).trim()
        }

        sh '''
          echo "Frontend digest: ${FRONTEND_DIGEST}"
          echo "Backend digest: ${BACKEND_DIGEST}"
        '''
      }
    }

    stage('Update GitOps Repo') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'github-gitops-token',
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_TOKEN'
        )]) {
          sh '''
            rm -rf gitops
            git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/<org>/three-tier-gitops.git gitops

            cd gitops
            git checkout ${GITOPS_BRANCH}

            yq -i ".frontend.image.repository = \\"${FRONTEND_IMAGE}\\"" ${GITOPS_VALUES_FILE}
            yq -i ".frontend.image.tag = \\"${IMAGE_TAG}\\"" ${GITOPS_VALUES_FILE}

            yq -i ".backend.image.repository = \\"${BACKEND_IMAGE}\\"" ${GITOPS_VALUES_FILE}
            yq -i ".backend.image.tag = \\"${IMAGE_TAG}\\"" ${GITOPS_VALUES_FILE}

            git config user.email "jenkins@company.com"
            git config user.name "jenkins-ci"

            git add ${GITOPS_VALUES_FILE}
            git commit -m "Deploy ${IMAGE_TAG} to staging" || echo "No changes to commit"
            git push origin ${GITOPS_BRANCH}
          '''
        }
      }
    }

    stage('Wait for Argo CD Deployment') {
      steps {
        sh '''
          echo "Argo CD will sync from GitOps repo."
          echo "For stricter pipeline control, use argocd CLI wait here."
        '''
      }
    }

    stage('OWASP ZAP DAST - Staging') {
      steps {
        sh '''
          mkdir -p zap-report

          docker run --rm \
            -v $(pwd)/zap-report:/zap/wrk/:rw \
            ghcr.io/zaproxy/zaproxy:stable \
            zap-baseline.py \
              -t ${STAGING_URL} \
              -r zap-baseline-report.html \
              -J zap-baseline-report.json \
              -w zap-baseline-report.md \
              -I
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'zap-report/*', allowEmptyArchive: true
        }
      }
    }
  }

  post {
    success {
      echo "Build ${BUILD_NUMBER} succeeded. Image tag: ${IMAGE_TAG}"
    }

    failure {
      echo "Build ${BUILD_NUMBER} failed."
    }

    always {
      sh '''
        docker image prune -f || true
        docker logout ${ECR_REGISTRY} || true
      '''
      cleanWs()
    }
  }
}
```

---

# 30. Why Jenkins updates GitOps instead of deploying directly

| Direct Jenkins deploy             | GitOps with Argo CD               |
| --------------------------------- | --------------------------------- |
| Jenkins needs cluster credentials | Argo CD owns cluster deployment   |
| Harder to audit                   | Git commit is audit trail         |
| Manual drift possible             | Argo detects and repairs drift    |
| Rollback depends on Jenkins job   | Rollback can use Git/Argo history |
| CI and CD tightly coupled         | CI and CD are separated           |

This is a production-grade pattern: **Jenkins = CI**, **Argo CD = CD**.

---

# 31. OWASP ZAP strategy

Use two ZAP modes:

| Environment          | ZAP mode                               | Why                         |
| -------------------- | -------------------------------------- | --------------------------- |
| Pull request preview | Baseline scan                          | Passive, safer, fast        |
| Staging nightly      | Full scan                              | Deeper active testing       |
| Production           | Passive only or external approved scan | Avoid breaking live systems |

Example full scan for staging only:

```bash
docker run --rm \
  -v $(pwd)/zap-report:/zap/wrk/:rw \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-full-scan.py \
    -t https://staging-crud.example.com \
    -r zap-full-report.html \
    -J zap-full-report.json
```

**Important:** Full active scans can send attack-like traffic. Run them only against staging or approved security environments.

---

# 32. Prometheus, Grafana, and Alertmanager

Install kube-prometheus-stack:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword='StrongPasswordHere'
```

The kube-prometheus-stack Helm chart bundles Kubernetes monitoring manifests, Grafana dashboards, Prometheus rules, and Prometheus Operator components. ([Artifact Hub][10])

Access Grafana:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
```

Open:

```text
http://localhost:3000
```

Recommended dashboards:

```text
Kubernetes cluster overview
Node exporter dashboard
Pod resource usage
Ingress/ALB dashboard
MongoDB exporter dashboard
Node.js application dashboard
Jenkins dashboard
Argo CD dashboard
```

---

# 33. Loki logging

Install Loki:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

kubectl create namespace logging

helm upgrade --install loki grafana/loki \
  --namespace logging

helm upgrade --install promtail grafana/promtail \
  --namespace logging \
  --set "config.clients[0].url=http://loki.logging.svc.cluster.local:3100/loki/api/v1/push"
```

Grafana is used to query, visualize, alert, and understand metrics/logs from data sources such as Prometheus and Loki. ([GitHub][11])

---

# 34. Application logging best practice

Node.js app should log JSON to stdout:

```js
console.log(JSON.stringify({
  level: "info",
  service: "backend",
  message: "request completed",
  method: req.method,
  path: req.path,
  statusCode: res.statusCode,
  timestamp: new Date().toISOString()
}));
```

**Why stdout:** Kubernetes collects container stdout/stderr. Promtail or Grafana Alloy ships logs to Loki. Do not write app logs to local files inside the container unless required.

---

# 35. Prometheus metrics in Node.js

Install:

```bash
npm install prom-client
```

Example:

```js
const client = require("prom-client");
const express = require("express");

const app = express();

client.collectDefaultMetrics();

app.get("/metrics", async (req, res) => {
  res.set("Content-Type", client.register.contentType);
  res.end(await client.register.metrics());
});

app.get("/health", (req, res) => {
  res.status(200).json({ status: "ok" });
});
```

Then create a `ServiceMonitor`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: backend
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: three-tier-backend
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

---

# 36. Alerting rules

Example alert for backend pod crash loops:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: backend-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: backend.rules
      rules:
        - alert: BackendPodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total{container="backend"}[5m]) > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Backend pod is restarting"
            description: "Backend pod has restarted in the last 5 minutes."

        - alert: HighBackendCPU
          expr: avg(rate(container_cpu_usage_seconds_total{container="backend"}[5m])) > 0.8
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High backend CPU usage"
```

---

# 37. MongoDB production approach

## Recommended production option

Use:

```text
MongoDB Atlas
or
Amazon DocumentDB
```

Why not MongoDB StatefulSet for production by default?

| Managed MongoDB / DocumentDB | MongoDB StatefulSet            |
| ---------------------------- | ------------------------------ |
| Managed backups              | You manage backups             |
| Managed patching             | You patch yourself             |
| HA built in                  | You design HA                  |
| Easier encryption            | You configure storage/security |
| Lower ops burden             | Higher Kubernetes ops burden   |

For lab, MongoDB StatefulSet is okay.

---

# 38. MongoDB lab StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb
  replicas: 1
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
          image: mongo:7
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: username
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: password
          volumeMounts:
            - name: mongo-data
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: mongo-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

---

# 39. Secrets management

Do not store MongoDB URI directly in Helm values.

For production, use:

```text
AWS Secrets Manager
+ External Secrets Operator
+ Kubernetes Secret generated automatically
```

Example ExternalSecret:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: mongo-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: three-tier-mongo-secret
    creationPolicy: Owner
  data:
    - secretKey: mongo-uri
      remoteRef:
        key: prod/three-tier/mongodb
        property: mongo-uri
```

**Why:** Jenkins, GitHub, and Helm values never see the real secret.

---

# 40. Argo CD rollback

Rollback options:

## Option 1: Git revert

```bash
git revert <bad-gitops-commit>
git push origin main
```

Argo CD syncs the previous desired state.

## Option 2: Argo CD UI

```text
Argo CD UI
→ Application
→ History and Rollback
→ Select previous revision
→ Rollback
```

## Option 3: Kubernetes rollout undo

Use only for emergency:

```bash
kubectl rollout undo deployment/three-tier-backend -n three-tier-staging
```

Best production rollback is Git-based, because Git remains source of truth.

---

# 41. Scaling strategy

## 41.1 Pod replicas

Start with:

```text
frontend replicas: 2 minimum
backend replicas: 2 minimum
MongoDB: managed DB, not app HPA
```

## 41.2 HPA

Use CPU at first:

```text
frontend: 60% CPU
backend: 60% CPU
```

Later improve with:

```text
request rate
latency
queue depth
custom Prometheus metrics
```

## 41.3 Cluster Autoscaler or Karpenter

In EKS production, add:

```text
Karpenter or Cluster Autoscaler
```

HPA scales pods. Karpenter/Cluster Autoscaler scales nodes. You need both for real elasticity.

---

# 42. Security gates summary

| Stage              | Tool               | Blocks pipeline? | Purpose                     |
| ------------------ | ------------------ | ---------------: | --------------------------- |
| Dependency install | npm audit optional |         Optional | Known package issues        |
| Lint               | ESLint             |              Yes | Code quality                |
| Unit test          | Jest/Vitest        |              Yes | Functional safety           |
| SAST               | SonarQube          |              Yes | Code bugs/security          |
| Quality gate       | SonarQube          |              Yes | Enforce standards           |
| FS scan            | Trivy              |              Yes | Dependencies/secrets/config |
| Image scan         | Trivy              |              Yes | Runtime image CVEs          |
| DAST               | OWASP ZAP          |  Yes for staging | Runtime web/API issues      |
| GitOps diff        | Argo CD            |              Yes | Desired state control       |
| Runtime monitoring | Prometheus/Grafana |            Alert | Production health           |

---

# 43. Production best practices checklist

## Jenkins

```text
Do not run builds on controller
Use agents with labels
Use IAM role instead of AWS keys
Use credentials binding
Use folder-level permissions
Use webhook secret
Back up Jenkins home
Patch plugins regularly
Disable unused plugins
```

## Docker

```text
Use minimal base images
Use non-root users
Use npm ci
Use .dockerignore
Never copy .env
Pin major versions
Scan images before push/deploy
Prefer image digest for prod
```

## Kubernetes

```text
Use namespaces per environment
Use resource requests/limits
Use readiness/liveness probes
Use HPA
Use PodDisruptionBudgets
Use NetworkPolicies
Use RBAC least privilege
Use External Secrets
Use ALB Ingress, not NodePort
Use rolling updates
```

## Argo CD

```text
Use AppProjects
Use SSO
Disable admin user
Use auto-sync for dev/staging
Use manual sync or approval for prod
Use prune/self-heal carefully
Use Git revert for rollback
```

## Observability

```text
Expose /metrics from backend
Use JSON logs
Create dashboards per service
Alert on crash loops, 5xx, latency, CPU, memory, HPA maxed out
Send alerts to Slack/Teams/email
```

---

# 44. End-to-end implementation order

Follow this order exactly:

```text
1. Create GitHub app repo and GitOps repo
2. Build React + Node + Mongo locally with docker-compose
3. Create Dockerfiles and .dockerignore
4. Create ECR repositories
5. Create Jenkins controller EC2
6. Create Jenkins agent EC2
7. Install Docker, Node.js, Trivy, AWS CLI, Helm, kubectl on agent
8. Attach IAM role to Jenkins agent
9. Connect Jenkins controller to agent over SSH
10. Install and configure SonarQube
11. Add Jenkins credentials
12. Create Jenkins multibranch pipeline
13. Create EKS cluster or Minikube cluster
14. Install Argo CD
15. Install Prometheus/Grafana
16. Install Loki/Promtail
17. Install AWS Load Balancer Controller for EKS
18. Create Helm chart
19. Create Argo CD Application
20. Run Jenkins pipeline
21. Confirm images pushed to ECR
22. Confirm GitOps repo updated
23. Confirm Argo CD sync
24. Confirm app reachable through ingress
25. Run ZAP DAST scan
26. Validate dashboards and alerts
27. Test rollback
28. Test scaling
```

---

# 45. Final recommended production flow

The best production-grade flow is:

```text
Developer pushes code
→ GitHub webhook triggers Jenkins
→ Jenkins agent checks out code
→ Jenkins runs lint + unit tests
→ SonarQube SAST quality gate
→ Trivy filesystem/secret scan
→ Docker build
→ Trivy image scan
→ Push immutable image to ECR
→ Jenkins updates GitOps Helm values
→ Argo CD detects GitOps change
→ Argo CD deploys to EKS
→ Kubernetes performs rolling update
→ HPA scales pods
→ Prometheus collects metrics
→ Loki collects logs
→ Grafana dashboards visualize health
→ Alertmanager/Grafana sends alerts
→ OWASP ZAP validates staging runtime security
```

This is the clean separation you want:

```text
Jenkins = CI and artifact promotion
Argo CD = Kubernetes CD
GitHub = source of truth
ECR = artifact registry
EKS = runtime platform
Prometheus/Loki/Grafana = observability
SonarQube/Trivy/ZAP = security gates
```

That is a production-grade DevSecOps architecture for a ReactJS, Node.js, and MongoDB three-tier CRUD application.

[1]: https://argo-cd.readthedocs.io/?utm_source=chatgpt.com "Argo CD - Declarative GitOps CD for Kubernetes"
[2]: https://docs.aws.amazon.com/eks/latest/best-practices/security.html?utm_source=chatgpt.com "Best Practices for Security - Amazon EKS - AWS Documentation"
[3]: https://www.jenkins.io/doc/book/using/using-agents/?utm_source=chatgpt.com "Using Jenkins agents"
[4]: https://trivy.dev/?utm_source=chatgpt.com "Trivy"
[5]: https://www.jenkins.io/doc/pipeline/steps/sonar/?utm_source=chatgpt.com "SonarQube Scanner for Jenkins"
[6]: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/?utm_source=chatgpt.com "Deployments"
[7]: https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/?utm_source=chatgpt.com "Horizontal Pod Autoscaling"
[8]: https://argo-cd.readthedocs.io/en/latest/user-guide/sync-options/?utm_source=chatgpt.com "Sync Options - Argo CD - Declarative GitOps CD for Kubernetes"
[9]: https://www.zaproxy.org/docs/docker/baseline-scan/?utm_source=chatgpt.com "Baseline Scan - Zed Attack Proxy (ZAP)"
[10]: https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack?utm_source=chatgpt.com "kube-prometheus-stack 85.2.2"
[11]: https://github.com/grafana/grafana?utm_source=chatgpt.com "grafana/grafana: The open and composable observability ..."
