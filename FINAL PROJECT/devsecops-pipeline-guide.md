# Production-Grade DevSecOps CI/CD Pipeline (Teaching & Testing Edition)

## Jenkins · GitHub · SonarQube · Trivy · OWASP ZAP · Docker Hub / ECR · Argo CD · Minikube · Prometheus · Grafana · Loki

> **Audience:** DevOps learners and instructors running this on AWS EC2 + Minikube (no EKS required).
> **Stack:** ReactJS frontend · Node.js backend · MongoDB · Full DevSecOps toolchain.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Infrastructure Setup Order](#2-infrastructure-setup-order)
3. [EC2 Sizing: What Runs Where](#3-ec2-sizing-what-runs-where)
4. [Jenkins Controller EC2 Setup](#4-jenkins-controller-ec2-setup)
5. [Jenkins Agent EC2 Setup](#5-jenkins-agent-ec2-setup)
6. [Connect Controller to Agent](#6-connect-controller-to-agent)
7. [SonarQube Setup (EC2 or Docker)](#7-sonarqube-setup-ec2-or-docker)
8. [GitHub Configuration](#8-github-configuration)
9. [Registry Options: Docker Hub vs ECR](#9-registry-options-docker-hub-vs-ecr)
10. [Minikube Setup (Kubernetes for Testing)](#10-minikube-setup-kubernetes-for-testing)
11. [Argo CD Setup — Step by Step](#11-argo-cd-setup--step-by-step)
12. [Prometheus Setup — Step by Step](#12-prometheus-setup--step-by-step)
13. [Grafana Setup — Step by Step](#13-grafana-setup--step-by-step)
14. [Loki Setup — Step by Step](#14-loki-setup--step-by-step)
15. [Repository Structure](#15-repository-structure)
16. [Dockerfiles](#16-dockerfiles)
17. [Jenkins Pipelines: Frontend & Backend Separately](#17-jenkins-pipelines-frontend--backend-separately)
18. [OWASP ZAP — Complete Guide](#18-owasp-zap--complete-guide)
19. [Email Notifications](#19-email-notifications)
20. [Helm Chart Structure](#20-helm-chart-structure)
21. [Argo CD Application Manifests](#21-argo-cd-application-manifests)
22. [Kubernetes Manifests (Minikube)](#22-kubernetes-manifests-minikube)
23. [Security Gates Summary](#23-security-gates-summary)
24. [End-to-End Flow Diagram](#24-end-to-end-flow-diagram)
25. [Best Practices Checklist](#25-best-practices-checklist)
26. [Rollback Procedures](#26-rollback-procedures)
27. [Troubleshooting Quick Reference](#27-troubleshooting-quick-reference)

---

## 1. Architecture Overview

```
Developer → GitHub → Jenkins Controller (EC2-A)
                           │
                    delegates to
                           │
                    Jenkins Agent (EC2-B)
                     ├── npm install, lint, test   ← runs on EC2-B directly
                     ├── sonar-scanner             ← runs on EC2-B, sends to SonarQube server
                     ├── trivy fs scan             ← runs on EC2-B (trivy binary)
                     ├── docker build              ← Docker daemon on EC2-B
                     ├── trivy image scan          ← trivy binary on EC2-B
                     ├── docker push               ← push to Docker Hub or ECR
                     ├── git push GitOps repo      ← updates Helm values
                     └── OWASP ZAP                 ← Docker container on EC2-B (ephemeral)

GitOps Repo ──────→ Argo CD (Minikube pod)
                     └── helm upgrade → frontend pods + backend pods + MongoDB

Minikube (EC2-B or separate EC2-C)
  ├── Namespace: three-tier-staging
  │     ├── frontend Deployment  (nginx serving React build)
  │     ├── backend Deployment   (Node.js)
  │     └── MongoDB StatefulSet
  └── Namespace: monitoring
        ├── Prometheus (kube-prometheus-stack)
        ├── Grafana
        └── Loki + Promtail
```

### Two Registry Options

| | Docker Hub | Amazon ECR |
|---|---|---|
| **Use for** | Teaching, testing, demos | Production |
| **Auth method** | Jenkins Credential (username/password) | IAM Role on Agent EC2 (no stored keys) |
| **Cost** | Free for public; $7/mo private | $0.10/GB storage; free pull within AWS |
| **Rate limits** | 100 pulls/6h (unauthenticated) | None within AWS |
| **Setup effort** | Create account → create repo → add creds to Jenkins | Create repo via CLI → attach IAM role to EC2 |

---

## 2. Infrastructure Setup Order

Follow this order exactly. Skipping steps causes dependency failures.

```
Phase 1 — Core Infrastructure
  Step 1:  Create EC2-A (Jenkins Controller)
  Step 2:  Create EC2-B (Jenkins Agent + Minikube)
  Step 3:  Configure Security Groups
  Step 4:  Install Jenkins on EC2-A
  Step 5:  Install tools on EC2-B (Docker, Node.js, Trivy, AWS CLI, kubectl, Helm, yq)
  Step 6:  Connect EC2-A ↔ EC2-B via SSH agent

Phase 2 — SonarQube
  Step 7:  Install SonarQube on EC2-A (Docker Compose) OR a third EC2
  Step 8:  Configure Jenkins → SonarQube integration
  Step 9:  Install SonarQube Scanner on EC2-B

Phase 3 — Registry
  Step 10: Create Docker Hub repos OR ECR repos
  Step 11: Add credentials to Jenkins

Phase 4 — Kubernetes (Minikube on EC2-B)
  Step 12: Install and start Minikube on EC2-B
  Step 13: Enable Minikube addons (ingress, metrics-server)
  Step 14: Install Argo CD in Minikube
  Step 15: Install kube-prometheus-stack (Prometheus + Grafana)
  Step 16: Install Loki + Promtail

Phase 5 — Application
  Step 17: Create app repo (GitHub)
  Step 18: Create GitOps repo (GitHub)
  Step 19: Create Helm chart in GitOps repo
  Step 20: Create Argo CD Application pointing to GitOps repo
  Step 21: Add GitHub webhook to trigger Jenkins

Phase 6 — Pipeline
  Step 22: Create Jenkins multibranch pipeline for frontend
  Step 23: Create Jenkins multibranch pipeline for backend
  Step 24: Push code → verify full pipeline runs

Phase 7 — Observability
  Step 25: Import Grafana dashboards
  Step 26: Configure Loki datasource in Grafana
  Step 27: Set up Alertmanager email rules
  Step 28: Test rollback
```

---

## 3. EC2 Sizing: What Runs Where

### EC2-A: Jenkins Controller

```
Purpose:        Jenkins Controller only. Never runs builds.
AMI:            Ubuntu 24.04 LTS
Instance type:  t3.medium (2 vCPU, 4 GB RAM) — minimum
                t3.large  (2 vCPU, 8 GB RAM) — recommended
Disk:           50 GB gp3
Subnet:         Private preferred; public with Security Group restriction for teaching
```

**What runs ON EC2-A:**
- Jenkins Controller process (Java)
- Jenkins plugin system
- Job scheduling and agent coordination
- Jenkins credentials vault
- Jenkins build history and logs storage

**What does NOT run on EC2-A:**
- Docker builds
- npm install
- Trivy scans
- Any build stage — all of these must run on EC2-B (agent)

### EC2-B: Jenkins Agent + Minikube

```
Purpose:        Jenkins Agent AND Minikube cluster host
AMI:            Ubuntu 24.04 LTS
Instance type:  t3.xlarge (4 vCPU, 16 GB RAM) — minimum for both agent + Minikube
                t3.2xlarge (8 vCPU, 32 GB RAM) — recommended
Disk:           80–100 GB gp3 (Docker images + Minikube cache consume lots of space)
Subnet:         Private; access via bastion/SSM for production pattern
IAM Role:       jenkins-agent-role (if using ECR)
```

**What runs ON EC2-B:**
- Jenkins Agent process (connected to EC2-A)
- npm install, lint, unit tests (runs directly on EC2 as jenkins user)
- sonar-scanner CLI (sends results to SonarQube server)
- trivy binary (FS scan of workspace, image scan of local Docker images)
- Docker daemon (builds and stores images)
- docker push (pushes to Docker Hub or ECR)
- git clone + yq (GitOps repo update)
- OWASP ZAP — runs as a Docker **container** on EC2-B (ephemeral, auto-removed after scan)
- Minikube (single-node Kubernetes cluster)
- All Kubernetes workloads: frontend, backend, MongoDB, Argo CD, Prometheus, Grafana, Loki

### Optional EC2-C: SonarQube Server

```
Purpose:        SonarQube Community Edition server
AMI:            Ubuntu 24.04 LTS
Instance type:  t3.medium minimum; t3.large recommended
Disk:           30 GB gp3
```

For teaching, SonarQube can also run as Docker Compose on EC2-A or EC2-B if resources allow.

### Security Groups

```
EC2-A (Controller) Inbound:
  TCP 22   from your IP or EC2-B only  (SSH)
  TCP 8080 from your IP only            (Jenkins UI)
  TCP 8080 from EC2-B                   (agent check-in)

EC2-B (Agent) Inbound:
  TCP 22   from EC2-A only              (SSH from controller)
  TCP 22   from your IP                 (your access for debugging)
  TCP 30000-32767 from your IP          (Minikube NodePort services)

EC2-C (SonarQube) Inbound:
  TCP 9000 from EC2-A and EC2-B only   (SonarQube UI and API)
  TCP 22   from your IP only            (SSH)
```

---

## 4. Jenkins Controller EC2 Setup

All commands run on **EC2-A** as your SSH user (ubuntu).

### 4.1 Install Java and Jenkins

```bash
sudo apt update && sudo apt upgrade -y

# Java 21 (Jenkins requires Java 17 or 21)
sudo apt install -y fontconfig openjdk-21-jdk curl gnupg
java -version

# Add Jenkins repo
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins

sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

### 4.2 Get Initial Admin Password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open `http://<EC2-A-Public-IP>:8080` and complete setup wizard.

### 4.3 Install Required Jenkins Plugins

```
Go to: Manage Jenkins → Plugins → Available plugins

Install:
  Pipeline
  Pipeline: Stage View
  Multibranch Pipeline
  Blue Ocean (optional, nice UI)
  SonarQube Scanner
  Credentials Binding
  GitHub Branch Source
  Docker Pipeline
  HTML Publisher
  Email Extension (Mailer)
  AnsiColor
  Timestamper
  Build Discarder
```

### 4.4 Configure Email in Jenkins

```
Manage Jenkins → System → E-mail Notification

SMTP server: smtp.gmail.com
Use SSL: Yes
SMTP Port: 465
Use SMTP Authentication: Yes
  Username: jenkins-notify@yourdomain.com
  Password: (app password, not your real password)
Reply-To Address: jenkins-notify@yourdomain.com

Test by clicking "Test configuration by sending test e-mail"
```

For Gmail: enable 2FA → create App Password at myaccount.google.com/apppasswords.

### 4.5 Configure Extended Email Plugin

```
Manage Jenkins → System → Extended E-mail Notification

SMTP server: smtp.gmail.com
SMTP Port:   465
Use SSL:     Yes
Credentials: (same as above)
Default Recipients: devops-team@company.com
Default Subject:    ${PROJECT_NAME} - Build #${BUILD_NUMBER} - ${BUILD_STATUS}
```

### 4.6 Harden Jenkins

```bash
# Increase Java heap for large pipelines
sudo systemctl edit jenkins
```

Add in the editor:
```ini
[Service]
Environment="JAVA_OPTS=-Xmx2g -Duser.timezone=Asia/Kathmandu"
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

Best practices:
- Disable anonymous access: `Manage Jenkins → Security → Authorization → Logged-in users can do anything` minimum; use matrix-based RBAC for teams
- Disable CLI over remoting: `Manage Jenkins → Security → Disable CLI access`
- Keep Jenkins behind HTTPS for non-teaching environments

---

## 5. Jenkins Agent EC2 Setup

All commands run on **EC2-B**.

### 5.1 Install Java (required for Jenkins Agent)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y fontconfig openjdk-21-jdk git curl wget unzip jq ca-certificates gnupg lsb-release
```

### 5.2 Create Jenkins User

```bash
sudo useradd -m -s /bin/bash jenkins
sudo mkdir -p /home/jenkins/.ssh
sudo chown -R jenkins:jenkins /home/jenkins
sudo chmod 700 /home/jenkins/.ssh
```

### 5.3 Install Docker

```bash
sudo apt remove docker docker-engine docker.io containerd runc -y 2>/dev/null || true

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker jenkins
sudo usermod -aG docker $USER   # also add your own user for testing
sudo systemctl enable docker
sudo systemctl start docker

# Verify
docker --version
```

### 5.4 Install Node.js 20 LTS

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
nvm install 24
node -v
npm -v
```

### 5.5 Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### 5.6 Install kubectl

```bash
# Match the Minikube Kubernetes version
curl -LO "https://dl.k8s.io/release/v1.32.0/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

### 5.7 Install Helm 3

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### 5.8 Install Trivy

```bash
# Check https://github.com/aquasecurity/trivy/releases for latest version
sudo apt-get install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

### 5.9 Install yq (for YAML editing in GitOps stage)

```bash
YQ_VERSION="v4.43.1"
wget "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/yq_linux_amd64" -O yq
chmod +x yq
sudo mv yq /usr/local/bin/
yq --version
```

### 5.10 Install SonarQube Scanner

```bash
SONAR_VERSION="6.2.1.4610"
wget "https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-${SONAR_VERSION}-linux-x64.zip"
unzip sonar-scanner-cli-${SONAR_VERSION}-linux-x64.zip
sudo mv sonar-scanner-${SONAR_VERSION}-linux-x64 /opt/sonar-scanner
sudo ln -s /opt/sonar-scanner/bin/sonar-scanner /usr/local/bin/sonar-scanner
sonar-scanner --version
```

### 5.11 Install Minikube

```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
minikube version
```

---

## 6. Connect Controller to Agent

### On EC2-A (Controller): Generate SSH key pair

```bash
# Switch to jenkins user on controller
sudo su - jenkins

# Generate key pair (no passphrase for automation)
ssh-keygen -t ed25519 -f ~/.ssh/jenkins-agent-key -C "jenkins-agent" -N ""

# Show the public key (copy this)
cat ~/.ssh/jenkins-agent-key.pub
```

### On EC2-B (Agent): Authorize the key

```bash
sudo su - jenkins
mkdir -p ~/.ssh
# Paste the public key from EC2-A here:
echo "PASTE_PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### On EC2-A (Controller): Test connectivity

```bash
sudo su - jenkins
ssh -i ~/.ssh/jenkins-agent-key jenkins@<EC2-B-PRIVATE-IP> hostname
```

You should see EC2-B's hostname without a password prompt.

### In Jenkins UI: Add Agent

```
Manage Jenkins → Nodes → New Node

Name:                   agent-01
Type:                   Permanent Agent
Remote root directory:  /home/jenkins
Labels:                 docker nodejs trivy aws helm kubectl
Usage:                  Use this node as much as possible
Launch method:          Launch agents via SSH

Host:                   <EC2-B-PRIVATE-IP>
Credentials:            → Add → Jenkins
  Kind: SSH Username with private key
  Username: jenkins
  Private Key: Enter directly → paste contents of ~/.ssh/jenkins-agent-key (the private key)

Host Key Verification:  Non verifying (for teaching)
                        OR Known hosts file (for production)

Number of executors: 2
```

Click **Save** → click **Launch agent** and verify it connects.

---

## 7. SonarQube Setup (EC2 or Docker)

### Option A: SonarQube on EC2-A via Docker Compose (easiest for teaching)

Run on **EC2-A**:

```bash
sudo apt install -y docker-compose-plugin

mkdir -p ~/sonarqube
cat > ~/sonarqube/docker-compose.yml << 'EOF'
version: '3'
services:
  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    restart: unless-stopped
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://sonar-db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
    ports:
      - "9000:9000"
    depends_on:
      - sonar-db
    volumes:
      - sonar-data:/opt/sonarqube/data
      - sonar-extensions:/opt/sonarqube/extensions
      - sonar-logs:/opt/sonarqube/logs

  sonar-db:
    image: postgres:15
    container_name: sonar-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: sonar
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
    volumes:
      - sonar-postgres:/var/lib/postgresql/data

volumes:
  sonar-data:
  sonar-extensions:
  sonar-logs:
  sonar-postgres:
EOF

# Required kernel setting for Elasticsearch inside SonarQube
sudo sysctl -w vm.max_map_count=524288
echo "vm.max_map_count=524288" | sudo tee -a /etc/sysctl.conf

cd ~/sonarqube
docker compose up -d
```

Access: `http://<EC2-A-IP>:9000` — default login: `admin` / `admin` (change immediately).

### SonarQube Initial Setup

```
1. Log in → change password
2. Go to Administration → Security → Generate Token
   Name: jenkins-token
   Type: Global Analysis Token
   Copy the token value

3. Create project:
   Manually → Project key: crud-backend, crud-frontend
```

### Jenkins → SonarQube Integration

```
Manage Jenkins → System → SonarQube servers

Name:       sonarqube-server
Server URL: http://<EC2-A-IP>:9000
Token:      Add → Secret text → paste your SonarQube token
            Credential ID: sonarqube-token
```

---

## 8. GitHub Configuration

### 8.1 Create Two Repositories

```
Repo 1: three-tier-crud-app    (application code + Jenkinsfiles)
Repo 2: three-tier-gitops      (Helm charts + Argo CD apps)
```

### 8.2 Personal Access Tokens

```
GitHub → Settings → Developer Settings → Personal Access Tokens → Fine-grained tokens

Token 1 (for Jenkins reading app repo):
  Name: jenkins-app-read
  Permissions: Contents (read), Webhooks (read)

Token 2 (for Jenkins writing GitOps repo):
  Name: jenkins-gitops-write
  Permissions: Contents (read + write)
```

Add both tokens to Jenkins:
```
Manage Jenkins → Credentials → System → Global credentials
Add: Username with Password
  Username: your-github-username
  Password: token-value
  ID: github-gitops-token
```

### 8.3 Webhook

```
GitHub App Repo → Settings → Webhooks → Add webhook

Payload URL:  http://<EC2-A-IP>:8080/github-webhook/
Content type: application/json
Events:       Just the push event + Pull requests
```

### 8.4 Branch Protection (optional for teaching, important for production)

```
Settings → Branches → Add rule for main:
  Require pull request before merging
  Require status checks (select your Jenkins job)
  Do not allow bypassing the above settings
```

---

## 9. Registry Options: Docker Hub vs ECR

### Option A: Docker Hub (recommended for teaching)

#### Setup Steps

```bash
# 1. Create account at hub.docker.com
# 2. Create two repositories:
#    yourusername/crud-backend   (private)
#    yourusername/crud-frontend  (private)

# 3. Add credentials to Jenkins:
Manage Jenkins → Credentials → System → Global
  Kind: Username with password
  Username: yourdockerhubusername
  Password: your-dockerhub-password OR access token (preferred)
  ID: dockerhub-creds
```

#### In Jenkinsfile (Docker Hub mode)

The pipeline uses `withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', ...)])` to log in and push.

### Option B: Amazon ECR (production pattern)

#### Setup Steps

```bash
# 1. Create ECR repositories
aws ecr create-repository \
  --repository-name three-tier/frontend \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --region ap-south-1

aws ecr create-repository \
  --repository-name three-tier/backend \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --region ap-south-1

# 2. Create IAM policy (save as jenkins-ecr-policy.json)
cat > jenkins-ecr-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ecr:GetAuthorizationToken"],
      "Resource": "*"
    },
    {
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
        "arn:aws:ecr:ap-south-1:ACCOUNT:repository/three-tier/frontend",
        "arn:aws:ecr:ap-south-1:ACCOUNT:repository/three-tier/backend"
      ]
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name jenkins-agent-ecr-policy \
  --policy-document file://jenkins-ecr-policy.json

# 3. Attach policy to EC2-B instance role
# In AWS Console: EC2 → EC2-B instance → Actions → Security → Modify IAM role
# Select role or create new role with this policy

# 4. Verify (from EC2-B):
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin \
  ACCOUNT.dkr.ecr.ap-south-1.amazonaws.com
```

#### Switching between Docker Hub and ECR

In both Jenkinsfile-backend and Jenkinsfile-frontend, change:
```groovy
REGISTRY_TYPE = 'dockerhub'   // for Docker Hub
REGISTRY_TYPE = 'ecr'         // for ECR
```

---

## 10. Minikube Setup (Kubernetes for Testing)

All commands on **EC2-B** as your regular user (not jenkins user).

### 10.1 Start Minikube

```bash
# Start with enough resources for all workloads
# driver=docker uses Docker daemon already installed
minikube start \
  --driver=docker \
  --cpus=4 \
  --memory=8192 \
  --disk-size=40g \
  --kubernetes-version=v1.32.0

minikube status
kubectl get nodes
```

### 10.2 Enable Required Addons

```bash
# Ingress controller (nginx) — for routing to frontend/backend
minikube addons enable ingress

# Metrics server — required for HPA
minikube addons enable metrics-server

# Dashboard (optional)
minikube addons enable dashboard

# Verify
minikube addons list
kubectl get pods -n ingress-nginx
```

### 10.3 Create Namespaces

```bash
kubectl create namespace three-tier-staging
kubectl create namespace argocd
kubectl create namespace monitoring
kubectl create namespace logging
```

### 10.4 Allow jenkins User to Use kubectl

```bash
# Copy kubeconfig to jenkins user
sudo mkdir -p /home/jenkins/.kube
sudo cp ~/.kube/config /home/jenkins/.kube/config
sudo chown -R jenkins:jenkins /home/jenkins/.kube
sudo chmod 600 /home/jenkins/.kube/config

# Test as jenkins user
sudo -u jenkins kubectl get nodes
```

### 10.5 Minikube IP and NodePort Access

```bash
# Get Minikube IP (for accessing services from your browser via SSH tunnel)
minikube ip

# Example: access app at http://$(minikube ip):31000
# Or use: minikube service <service-name> -n <namespace> --url
```

---

## 11. Argo CD Setup — Step by Step

All commands on **EC2-B** (where Minikube runs).

### 11.1 Install Argo CD in Minikube

```bash
# Namespace created in step 10.3
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready (takes 2-3 minutes)
kubectl -n argocd rollout status deployment/argocd-server
kubectl get pods -n argocd
```

### 11.2 Get Initial Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo   # print newline
```

### 11.3 Access Argo CD UI

```bash
# Option 1: Port forward (run in background or separate terminal)
kubectl -n argocd port-forward svc/argocd-server 8081:443 --address 0.0.0.0 &

# Then access: http://<EC2-B-IP>:8081
# Accept the self-signed certificate warning
# Login: admin / <password from step 11.2>

# Option 2: NodePort (for easier teaching demo)
kubectl -n argocd patch svc argocd-server \
  -p '{"spec": {"type": "NodePort"}}'

kubectl get svc argocd-server -n argocd
# Note the NodePort — access at http://$(minikube ip):<nodeport>
```

### 11.4 Install Argo CD CLI (optional but useful)

```bash
ARGOCD_VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
curl -sSL -o argocd "https://github.com/argoproj/argo-cd/releases/download/${ARGOCD_VERSION}/argocd-linux-amd64"
chmod +x argocd
sudo mv argocd /usr/local/bin/

# Login via CLI
argocd login localhost:8081 --username admin --password <password> --insecure

# Change password
argocd account update-password
```

### 11.5 Connect Argo CD to Your GitOps Repo

```
In Argo CD UI:
  Settings → Repositories → Connect Repo

  Type: git
  Repository URL: https://github.com/YOUR_ORG/three-tier-gitops.git
  Username: your-github-username
  Password: your-github-token (needs repo read access)
```

Or via CLI:
```bash
argocd repo add https://github.com/YOUR_ORG/three-tier-gitops.git \
  --username your-github-username \
  --password your-github-token
```

### 11.6 Create Argo CD Application (staging)

```bash
cat > argocd-app-staging.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: three-tier-staging
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/YOUR_ORG/three-tier-gitops.git
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
      prune: true      # Remove resources deleted from Git
      selfHeal: true   # Fix drift: revert manual kubectl changes
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
EOF

kubectl apply -f argocd/app-staging.yaml
```

### 11.7 How Argo CD Deployment Flow Works

```
Jenkins pushes image → Docker Hub / ECR
Jenkins clones GitOps repo → updates values-staging.yaml with new image tag
Jenkins pushes commit to GitOps repo

Every 3 minutes (default poll interval), Argo CD:
  → Detects new commit in GitOps repo
  → Compares desired state (Git) vs actual state (Minikube)
  → If different: runs helm upgrade → Kubernetes rolling update
  → Frontend and backend pods replaced with new image version

To manually trigger sync (no waiting):
  argocd app sync three-tier-staging
  OR
  Argo CD UI → Application → Sync
```

### 11.8 Argo CD Verification Commands

```bash
# Check app status
argocd app get three-tier-staging

# Watch sync status
argocd app wait three-tier-staging --sync

# View history
argocd app history three-tier-staging

# Check pod status in Minikube
kubectl get pods -n three-tier-staging
kubectl get svc -n three-tier-staging
```

---

## 12. Prometheus Setup — Step by Step

Prometheus runs as a **pod inside Minikube** on EC2-B. It is not installed directly on EC2.

### 12.1 Install kube-prometheus-stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install in monitoring namespace
helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword='DevSecOps@2024' \
  --set prometheus.prometheusSpec.retention=7d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=5Gi \
  --wait

kubectl get pods -n monitoring
```

### 12.2 Access Prometheus UI

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090 --address 0.0.0.0 &
# Access: http://<EC2-B-IP>:9090
```

### 12.3 Verify Metrics Collection

```
In Prometheus UI → Status → Targets
You should see all Kubernetes targets (nodes, pods, API server, etc.) as UP
```

### 12.4 Configure Backend App to Expose Metrics

In Node.js backend, `prom-client` exposes `/metrics` endpoint. The ServiceMonitor tells Prometheus to scrape it:

```yaml
# Apply this after deploying the backend
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: crud-backend
  namespace: three-tier-staging
  labels:
    release: kube-prometheus-stack   # MUST match the Helm release label
spec:
  selector:
    matchLabels:
      app: three-tier-backend
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
  namespaceSelector:
    matchNames:
      - three-tier-staging
```

```bash
kubectl apply -f servicemonitor-backend.yaml
```

### 12.5 Alert Rules

```yaml
# backend-alerts.yaml
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
        - alert: BackendPodDown
          expr: up{job="crud-backend"} == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Backend pod is down"
            description: "Backend service has been unreachable for 1 minute"

        - alert: HighErrorRate
          expr: rate(http_requests_total{status_code=~"5.."}[5m]) > 0.05
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High backend error rate"
            description: "More than 5% of requests are returning 5xx errors"

        - alert: PodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total[5m]) > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Pod is crash looping"
```

```bash
kubectl apply -f backend-alerts.yaml
```

---

## 13. Grafana Setup — Step by Step

Grafana is installed as part of kube-prometheus-stack. It runs as a **pod in Minikube**.

### 13.1 Access Grafana

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80 --address 0.0.0.0 &
# Access: http://<EC2-B-IP>:3000
# Login: admin / DevSecOps@2024 (set during helm install)
```

### 13.2 Import Dashboards

In Grafana UI → Dashboards → Import → paste Dashboard ID:

| Dashboard                   | ID    | Purpose                          |
| --------------------------- | ----- | -------------------------------- |
| Kubernetes Cluster Overview | 7249  | Cluster health                   |
| Node Exporter Full          | 1860  | EC2/node metrics                 |
| Kubernetes Pod Stats        | 6336  | Pod resource usage               |
| Node.js App Dashboard       | 11159 | Node.js metrics from prom-client |
| Nginx Ingress Controller    | 9614  | Ingress request stats            |

### 13.3 Configure Datasource for Loki (after Loki install)

```
Grafana → connections → Data Sources → Add data source
Type: Loki
URL: http://loki.logging.svc.cluster.local:3100
Access: Server (default)
Save & Test
```

### 13.4 View Logs in Grafana

```
Grafana → Explore → Select Loki datasource
Label filter: namespace = three-tier-staging, app = three-tier-backend
Run query
```

### 13.5 Configure Alertmanager for Email

```bash
# Edit alertmanager config
kubectl -n monitoring get secret alertmanager-kube-prometheus-stack-alertmanager -o yaml

# Create custom config
cat > alertmanager-config.yaml << 'EOF'
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: email-alerts
  namespace: monitoring
spec:
  route:
    receiver: 'email-team'
    groupBy: ['alertname', 'namespace']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 1h
  receivers:
    - name: 'email-team'
      emailConfigs:
        - to: 'devops-team@company.com'
          from: 'alertmanager@company.com'
          smarthost: 'smtp.gmail.com:587'
          authUsername: 'alertmanager@company.com'
          authPassword:
            secret:
              name: alertmanager-email-secret
              key: password
          requireTLS: true
EOF

kubectl apply -f alertmanager-config.yaml
```

---

## 14. Loki Setup — Step by Step

Loki runs as a **pod inside Minikube**. Promtail runs as a DaemonSet on Minikube nodes to collect pod logs.

### 14.1 Install Loki Stack

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki (log storage)
helm upgrade --install loki grafana/loki \
  --namespace logging \
  --set loki.commonConfig.replication_factor=1 \
  --set loki.storage.type=filesystem \
  --set singleBinary.replicas=1 \
  --wait

# Install Promtail (log collector, runs as DaemonSet on each node)
helm upgrade --install promtail grafana/promtail \
  --namespace logging \
  --set config.clients[0].url=http://loki.logging.svc.cluster.local:3100/loki/api/v1/push \
  --wait

kubectl get pods -n logging
```

### 14.2 How Loki Log Collection Works

```
App container writes to stdout/stderr (JSON format recommended)
    ↓
Kubernetes stores logs at /var/log/pods/ on the node
    ↓
Promtail DaemonSet reads log files from node filesystem
    ↓
Promtail ships logs to Loki (HTTP push)
    ↓
Grafana queries Loki (LogQL) and displays logs
```

### 14.3 Why JSON Logging Matters

Promtail can parse JSON log lines and extract fields as Loki labels. This enables filtering by level, service, statusCode, etc. in Grafana.

Good log format (from Node.js backend):
```json
{"level":"info","service":"crud-backend","method":"GET","path":"/api/items","statusCode":200,"durationMs":12,"timestamp":"2024-01-15T10:00:00.000Z"}
```

Bad log format:
```
GET /api/items 200 12ms
```

### 14.4 Verify Logs are Flowing

```bash
# Check Promtail is shipping logs
kubectl logs -n logging daemonset/promtail | grep "level=info"

# In Grafana → Explore → Loki
# Query: {namespace="three-tier-staging"}
# You should see backend and frontend pod logs
```

---

## 15. Repository Structure

### Application Repo

```
three-tier-crud-app/
├── backend/
│   ├── src/
│   │   └── server.js          ← Express app: CRUD routes, health, metrics
│   ├── tests/
│   │   └── items.test.js      ← Jest + supertest + mongodb-memory-server
│   ├── Dockerfile             ← Multi-stage: node:20-alpine, non-root user
│   ├── .dockerignore
│   ├── package.json
│   └── sonar-project.properties
├── frontend/
│   ├── src/
│   │   ├── App.jsx             ← Main CRUD UI with React
│   │   ├── components/
│   │   │   ├── ItemForm.jsx
│   │   │   ├── ItemCard.jsx
│   │   │   └── FilterBar.jsx
│   │   ├── services/
│   │   │   └── api.js          ← Axios API client
│   │   ├── tests/
│   │   │   └── components.test.jsx  ← Vitest + Testing Library
│   │   └── main.jsx
│   ├── Dockerfile              ← Multi-stage: node build + nginx:alpine
│   ├── nginx.conf              ← Non-root nginx, SPA routing, security headers
│   ├── vite.config.js
│   └── package.json
├── docker-compose.yml          ← Local dev: MongoDB + backend + frontend
├── jenkins/
│   ├── Jenkinsfile-frontend    ← Frontend CI pipeline
│   └── Jenkinsfile-backend     ← Backend CI pipeline
└── README.md
```

### GitOps Repo

```
three-tier-gitops/
├── charts/
│   └── three-tier-app/
│       ├── Chart.yaml
│       ├── values.yaml             ← Base values (image placeholders)
│       ├── values-staging.yaml     ← Jenkins updates image tags here
│       ├── values-prod.yaml
│       └── templates/
│           ├── frontend-deployment.yaml
│           ├── frontend-service.yaml
│           ├── backend-deployment.yaml
│           ├── backend-service.yaml
│           ├── mongodb-statefulset.yaml
│           ├── mongodb-service.yaml
│           ├── ingress.yaml
│           ├── hpa-frontend.yaml
│           ├── hpa-backend.yaml
│           ├── servicemonitor.yaml
│           └── secrets.yaml
└── argocd/
    ├── app-staging.yaml
    └── app-prod.yaml
```

---

## 16. Dockerfiles

### Backend Dockerfile

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
# npm ci: reproducible installs, --omit=dev skips devDependencies
RUN npm ci --omit=dev

FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production

# Non-root user: reduces container privilege
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=deps /app/node_modules ./node_modules
COPY src ./src
COPY package*.json ./

USER appuser

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:5000/health || exit 1

CMD ["node", "src/server.js"]
```

### Frontend Dockerfile

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
# Vite builds React app to /app/dist
RUN npm run build

FROM nginx:1.27-alpine AS runner
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html

# nginx permissions for non-root
RUN chown -R appuser:appgroup /var/cache/nginx /var/run /var/log/nginx

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD wget -qO- http://localhost:8080/health || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

### Frontend nginx.conf

```nginx
server {
    listen 8080;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;   # SPA routing
    }

    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # Security headers (complements ZAP passive scan requirements)
    add_header X-Frame-Options "SAMEORIGIN"         always;
    add_header X-Content-Type-Options "nosniff"     always;
    add_header X-XSS-Protection "1; mode=block"     always;
    add_header Referrer-Policy "strict-origin"       always;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    gzip_min_length 1000;

    location ~* \.(js|css|png|jpg|ico|svg|woff2?)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## 17. Jenkins Pipelines: Frontend & Backend Separately

### Why Separate Pipelines?

| Shared pipeline | Separate pipelines |
|---|---|
| One failure stops both services | Frontend failure doesn't block backend deploy |
| One build history for both | Separate history, logs, and trend charts |
| Harder to trace which service failed | Clear ownership and accountability |
| One email for everything | Targeted email: [FRONTEND] Build Failed |
| Can't independently scale build speed | Can tune timeouts per service |

### Jenkins Pipeline Setup

```
Jenkins → New Item

Name: crud-backend-pipeline
Type: Multibranch Pipeline

Branch Sources:
  GitHub
  Credentials: github-app-repo-token
  Repository URL: https://github.com/YOUR_ORG/three-tier-crud-app.git
  Build strategies: All branches

Build Configuration:
  Mode: by Jenkinsfile
  Script Path: jenkins/Jenkinsfile-backend

Scan Multibranch Pipeline Triggers:
  Periodically: 5 minutes (as backup; webhook is primary)
```

Repeat the same for frontend:
```
Name: crud-frontend-pipeline
Script Path: jenkins/Jenkinsfile-frontend
```

### Pipeline Stage Summary

**Both pipelines run on: Jenkins Agent EC2 (EC2-B)**

| Stage | Where it runs | What it does |
|---|---|---|
| Prepare | EC2-B shell | Set image tag, verify tool versions |
| Install Dependencies | EC2-B shell | `npm ci` in app directory |
| Lint | EC2-B shell | ESLint runs on source files |
| Unit Tests | EC2-B shell | Jest/Vitest (no external services needed) |
| SonarQube SAST | EC2-B shell → SonarQube server | sonar-scanner sends analysis |
| Sonar Quality Gate | Jenkins controller polls SonarQube API | Abort if gate fails |
| Trivy Filesystem Scan | EC2-B shell (trivy binary) | Scan workspace files |
| Docker Build | EC2-B Docker daemon | Build and tag image locally |
| Trivy Image Scan | EC2-B shell (trivy binary) | Scan built image |
| Push Image | EC2-B → Docker Hub OR ECR | Upload image |
| Update GitOps Repo | EC2-B shell (git + yq) | Update Helm values |
| OWASP ZAP DAST | Docker container on EC2-B | Passive scan of staging URL |

---

## 18. OWASP ZAP — Complete Guide

### What ZAP Does

ZAP (Zed Attack Proxy) is a DAST (Dynamic Application Security Testing) tool. Unlike Trivy (which scans static code and images), ZAP sends actual HTTP requests to a running application and looks for vulnerabilities.

ZAP checks for:
- Missing security headers (X-Frame-Options, Content-Security-Policy, etc.)
- Clickjacking vulnerabilities
- SQL injection (full scan only)
- XSS (full scan only)
- Path traversal
- Sensitive data exposure
- Cookie security flags
- CORS misconfigurations

### Scan Modes

| Mode | Command | Time | Traffic | Use for |
|---|---|---|---|---|
| Baseline | `zap-baseline.py` | 2-5 min | Passive only (no attacks) | Every build on main/develop |
| Full | `zap-full-scan.py` | 20-60 min | Active attacks sent | Scheduled nightly against staging |
| API | `zap-api-scan.py` | 5-20 min | API-specific tests | Backend REST API testing |

**NEVER run full scan against production.** Active attacks can create records, corrupt data, or crash services.

### ZAP Setup in Jenkins (no separate installation)

ZAP runs as a Docker container on EC2-B. No installation needed — the `ghcr.io/zaproxy/zaproxy:stable` image is pulled automatically.

### Step-by-Step ZAP Flow

```
Step 1: Jenkins agent stage 'OWASP ZAP DAST' begins
Step 2: mkdir zap-reports (on EC2-B filesystem)
Step 3: docker run ghcr.io/zaproxy/zaproxy:stable
         - mounts zap-reports directory
         - --network host (so ZAP can reach localhost services)
Step 4: Inside the container, ZAP starts its embedded server
Step 5: zap-baseline.py spider the target URL
Step 6: Passive scan: analyze all responses for security issues
Step 7: Generate report files (HTML, JSON, MD)
Step 8: Container exits, auto-removed (--rm)
Step 9: Reports are available in zap-reports on EC2-B
Step 10: Jenkins archives reports as build artifacts
Step 11: Jenkins publishes HTML report (viewable in Jenkins UI)
```

### Accessing the Staging URL for ZAP

For Minikube, the app must be accessible from EC2-B. Options:

```bash
# Option 1: Port-forward (simplest for teaching)
# Run BEFORE Jenkins triggers ZAP stage
kubectl port-forward svc/three-tier-backend 5000:5000 -n three-tier-staging &
kubectl port-forward svc/three-tier-frontend 8080:80 -n three-tier-staging &

# Then in Jenkinsfile:
# STAGING_URL = 'http://localhost:5000'           (backend API)
# STAGING_FRONTEND_URL = 'http://localhost:8080'  (frontend)

# Option 2: NodePort (persistent, no port-forward needed)
kubectl patch svc three-tier-backend \
  -n three-tier-staging \
  -p '{"spec":{"type":"NodePort","ports":[{"port":5000,"targetPort":5000,"nodePort":31500}]}}'

MINIKUBE_IP=$(minikube ip)
# STAGING_URL = "http://${MINIKUBE_IP}:31500"

# Option 3: Minikube tunnel (for Ingress-based access)
minikube tunnel &
# Add to /etc/hosts: 127.0.0.1 staging.crud.local
```

### ZAP Full Scan (Staging Nightly) — Example

```bash
# Run manually or from a scheduled Jenkins job
docker run --rm \
  --network host \
  -v $(pwd)/zap-reports:/zap/wrk/:rw \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-full-scan.py \
    -t http://localhost:5000 \
    -r zap-full-report.html \
    -J zap-full-report.json \
    -z "-config scanner.attackStrength=MEDIUM"
```

### Interpreting ZAP Reports

```
FAIL (High risk): Pipeline should fail — fix immediately
  Examples: SQL injection, stored XSS, path traversal

WARN (Medium risk): Pipeline shows warning — fix soon
  Examples: Missing X-Frame-Options, missing X-Content-Type-Options

INFO (Informational): No action required
  Examples: Cookies without Secure flag (acceptable for HTTP dev)
```

To suppress known false positives, create `zap-rules.conf`:
```
10035	IGNORE	(Strict-Transport-Security Header Not Set — ignored for HTTP dev)
10054	IGNORE	(Cookie No HttpOnly Flag — acceptable in dev)
```

Pass it to ZAP:
```bash
zap-baseline.py -t http://localhost:5000 -c zap-rules.conf -r report.html
```

---

## 19. Email Notifications

### Required Jenkins Plugin

Install: **Email Extension Plugin** (emailext)

### Configure SMTP

```
Manage Jenkins → System → Extended E-mail Notification

SMTP Server: smtp.gmail.com
SMTP Port:   587
Use TLS:     Yes
Username:    jenkins-notify@gmail.com
Password:    app-password-from-google-account
Default Recipients: devops-team@company.com
```

### What Each Email Contains

**On Success:**
```
Subject: ✅ [BACKEND] Build #42 PASSED — main

Body:
  Branch, commit hash, image tag, image repository
  List of all stages that passed
  Link to build, console logs, coverage report, ZAP report
  Build duration
```

**On Failure:**
```
Subject: ❌ [BACKEND] Build #42 FAILED — main

Body:
  Branch, commit hash, build duration
  Which stage failed (from console)
  Links to Trivy FS report, Trivy image report, ZAP report
  Guidance on common failure causes
```

### Email is Sent From the Jenkins Pipeline `post {}` Block

Each Jenkinsfile has separate `success` and `failure` email blocks. This means:

- Frontend failure → email subject says `[FRONTEND]`
- Backend failure → email subject says `[BACKEND]`
- Teams can filter emails by subject prefix

---

## 20. Helm Chart Structure

`charts/three-tier-app/Chart.yaml`:
```yaml
apiVersion: v2
name: three-tier-app
description: ReactJS + Node.js + MongoDB CRUD app
type: application
version: 0.1.0
appVersion: "1.0.0"
```

`charts/three-tier-app/values.yaml`:
```yaml
global:
  environment: staging

frontend:
  image:
    repository: ""       # Jenkins sets this
    tag: ""              # Jenkins sets this
  replicaCount: 1
  service:
    port: 80
    targetPort: 8080
  resources:
    requests: { cpu: 100m, memory: 128Mi }
    limits:   { cpu: 500m, memory: 512Mi }

backend:
  image:
    repository: ""
    tag: ""
  replicaCount: 1
  service:
    port: 5000
    targetPort: 5000
  env:
    mongoDatabase: crudapp
  resources:
    requests: { cpu: 200m, memory: 256Mi }
    limits:   { cpu: 500m, memory: 512Mi }

mongodb:
  enabled: true
  image: mongo:7
  storage: 5Gi

ingress:
  enabled: true
  host: staging.crud.local
```

`charts/three-tier-app/values-staging.yaml` (Jenkins updates this file):
```yaml
# Jenkins CI updates frontend.image and backend.image on every build
frontend:
  image:
    repository: yourdockerhubuser/crud-frontend
    tag: "42-abc12345"   # ← Jenkins writes this

backend:
  image:
    repository: yourdockerhubuser/crud-backend
    tag: "42-abc12345"   # ← Jenkins writes this
```

---

## 21. Argo CD Application Manifests

`argocd/app-staging.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: three-tier-staging
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default

  source:
    repoURL: https://github.com/YOUR_ORG/three-tier-gitops.git
    targetRevision: main
    path: charts/three-tier-app
    helm:
      valueFiles:
        - values.yaml
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
      - ServerSideApply=true
      - PruneLast=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        maxDuration: 2m
        factor: 2
```

---

## 22. Kubernetes Manifests (Minikube)

### Backend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: three-tier-backend
  namespace: three-tier-staging
spec:
  replicas: 1            # Use 1 for Minikube to save resources
  selector:
    matchLabels:
      app: three-tier-backend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: three-tier-backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port:   "5000"
        prometheus.io/path:   "/metrics"
    spec:
      containers:
        - name: backend
          image: yourdockerhubuser/crud-backend:latest   # Replaced by Helm
          ports:
            - containerPort: 5000
          env:
            - name: NODE_ENV
              value: production
            - name: MONGO_URI
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: mongo-uri
          readinessProbe:
            httpGet: { path: /health, port: 5000 }
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /health, port: 5000 }
            initialDelaySeconds: 30
            periodSeconds: 20
          resources:
            requests: { cpu: 200m, memory: 256Mi }
            limits:   { cpu: 500m, memory: 512Mi }
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
```

### MongoDB Secret

```bash
# Create MongoDB secret in Kubernetes
kubectl create secret generic mongo-secret \
  --from-literal=mongo-uri="mongodb://root:rootpassword@mongodb:27017/crudapp?authSource=admin" \
  -n three-tier-staging
```

### Ingress for Minikube

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: three-tier-ingress
  namespace: three-tier-staging
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: staging.crud.local
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: three-tier-backend
                port: { number: 5000 }
          - path: /()(.*)
            pathType: Prefix
            backend:
              service:
                name: three-tier-frontend
                port: { number: 80 }
```

Add to `/etc/hosts` on your machine:
```
<MINIKUBE-IP>  staging.crud.local
```

---

## 23. Security Gates Summary

| Stage | Tool | Where it runs | Blocks build? | Catches |
|---|---|---|---|---|
| Lint | ESLint | EC2-B shell | Yes | Code style, obvious bugs |
| Unit Tests | Jest / Vitest | EC2-B shell | Yes | Functional regressions |
| SAST | SonarQube | EC2-B → SonarQube server | Yes | Security hotspots, code smells, bugs |
| Quality Gate | SonarQube | SonarQube API | Yes | Enforces coverage, duplication, rating |
| FS Scan | Trivy | EC2-B shell | Yes | Dependency CVEs, secrets in code, IaC issues |
| Image Scan | Trivy | EC2-B shell | Yes | OS layer + app layer CVEs in built image |
| DAST | OWASP ZAP | Docker container on EC2-B | Yes (configurable) | Runtime web vulnerabilities, missing headers |

---

## 24. End-to-End Flow Diagram

```
Developer pushes code to GitHub
        │
        ▼
GitHub webhook → Jenkins Controller (EC2-A)
        │
        ▼
Jenkins delegates to Agent (EC2-B)
        │
        ├── npm ci
        ├── npm run lint
        ├── npm test (Jest/Vitest + mongodb-memory-server)
        ├── sonar-scanner → SonarQube Quality Gate
        ├── trivy fs (workspace scan)
        ├── docker build (EC2-B Docker daemon)
        ├── trivy image (scans built image)
        ├── docker push → Docker Hub OR ECR
        ├── git push → GitOps repo (updates Helm values)
        └── docker run zaproxy → ZAP baseline scan of staging URL
                                 (container on EC2-B, auto-removed)
        │
        ▼ (pass)
Jenkins sends ✅ email with build info

Jenkins sends ❌ email on any stage failure
        │
        ▼ (GitOps update detected by Argo CD)
Argo CD (pod in Minikube on EC2-B)
  → polls GitOps repo every 3 minutes
  → detects new image tag in values-staging.yaml
  → runs helm upgrade
  → Kubernetes rolling update
        │
        ▼
Minikube (on EC2-B):
  ├── frontend pod: nginx serving React app
  ├── backend pod:  Node.js CRUD API
  └── MongoDB pod:  StatefulSet with PVC
        │
        ▼
Prometheus (Minikube pod) scrapes /metrics from backend
Promtail (Minikube DaemonSet) ships stdout logs to Loki
Grafana (Minikube pod) shows dashboards + logs
Alertmanager sends alert emails on crash loops or high error rates
```

---

## 25. Best Practices Checklist

### Jenkins
- [ ] Builds run on agent EC2, never on controller
- [ ] IAM Role on agent EC2 (no stored AWS keys) for ECR
- [ ] Jenkins credentials store for Docker Hub token and GitHub token
- [ ] Webhook secret configured (not open webhook)
- [ ] Build history limited (`logRotator(numToKeepStr: '20')`)
- [ ] Separate pipelines per service (frontend / backend)
- [ ] Email notification on both success and failure
- [ ] HTML reports published for coverage and ZAP

### Docker
- [ ] Multi-stage builds (smaller final image)
- [ ] Non-root user (`adduser -S appuser`)
- [ ] `.dockerignore` excludes node_modules, .env, test files
- [ ] `npm ci` instead of `npm install`
- [ ] `--omit=dev` in production stage
- [ ] Alpine base image
- [ ] HEALTHCHECK instruction
- [ ] Image tag includes build number + git short SHA

### Kubernetes (Minikube)
- [ ] Resource requests and limits on all containers
- [ ] readinessProbe and livenessProbe on all containers
- [ ] Secrets from Kubernetes Secret (not hardcoded in values)
- [ ] `allowPrivilegeEscalation: false`
- [ ] `capabilities: drop: [ALL]`
- [ ] RollingUpdate strategy with `maxUnavailable: 0`
- [ ] Separate namespace per environment

### Argo CD
- [ ] Auto-sync enabled for staging
- [ ] `selfHeal: true` to prevent manual drift
- [ ] `prune: true` to remove deleted resources
- [ ] Retry policy on sync failures
- [ ] GitOps repo is the single source of truth

### SonarQube
- [ ] Quality gate set to FAIL on: coverage < 60%, critical bugs > 0, security rating < A
- [ ] `waitForQualityGate abortPipeline: true` in Jenkinsfile
- [ ] Coverage report fed from Jest/Vitest lcov.info

### Trivy
- [ ] Filesystem scan with `--scanners vuln,secret,misconfig`
- [ ] Image scan after every build
- [ ] `.trivyignore` file with documented justifications for any ignored CVEs

### OWASP ZAP
- [ ] Baseline scan on every build to main/develop
- [ ] Full scan scheduled nightly against staging
- [ ] Reports archived as Jenkins artifacts
- [ ] HTML report published in Jenkins UI
- [ ] Rules file (`.conf`) to suppress known false positives

### Observability
- [ ] Backend exposes `/metrics` via `prom-client`
- [ ] ServiceMonitor created for Prometheus scraping
- [ ] JSON structured logs to stdout
- [ ] Grafana dashboards imported
- [ ] Alertmanager email configured
- [ ] PrometheusRule for crash loops and error rates

---

## 26. Rollback Procedures

### Method 1: Git Revert (Best Practice — GitOps)

```bash
# In your GitOps repo
git log --oneline charts/three-tier-app/values-staging.yaml

# Revert to previous state
git revert <commit-hash>
git push origin main

# Argo CD auto-syncs within 3 minutes
# Or force immediate sync:
argocd app sync three-tier-staging
```

### Method 2: Argo CD UI Rollback

```
Argo CD UI → three-tier-staging → History and Rollback
Click on a previous revision → Rollback
Confirm

This reverts to that Helm release revision instantly.
```

### Method 3: Kubernetes Rollout Undo (Emergency Only)

```bash
# Emergency — bypasses GitOps (creates drift, use only if Argo CD is down)
kubectl rollout undo deployment/three-tier-backend -n three-tier-staging
kubectl rollout undo deployment/three-tier-frontend -n three-tier-staging

# After emergency, immediately fix and push to GitOps repo
# Then: argocd app sync three-tier-staging
```

---

## 27. Troubleshooting Quick Reference

| Problem | Cause | Fix |
|---|---|---|
| Jenkins agent shows OFFLINE | SSH key mismatch or network SG | Check EC2-B SG allows port 22 from EC2-A; re-generate key pair |
| `npm ci` fails | Missing package-lock.json | Run `npm install` locally and commit the lock file |
| SonarQube quality gate times out | SonarQube server unreachable | Check EC2-C (or EC2-A) SonarQube container is running; check port 9000 SG |
| Trivy scan fails with exit code 5 | No vulnerabilities found (exit 5 = no targets) | Add `--exit-code 1` only for exit on vulnerability found |
| Docker build fails: no space | EC2-B disk full | `docker system prune -af`; increase EBS volume |
| ZAP scan fails: connection refused | Staging URL not accessible | Port-forward the service before ZAP stage runs |
| Argo CD not syncing | Poll interval (default 3 min) | `argocd app sync three-tier-staging` to force sync |
| Minikube out of memory | EC2-B too small | Upgrade to t3.xlarge; or reduce monitoring stack resources |
| Grafana shows no data | Prometheus datasource wrong URL | URL should be: `http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090` |
| Loki shows no logs | Promtail not running | `kubectl get pods -n logging`; check Promtail DaemonSet |
| Email not sending | SMTP auth failure | Use App Password (not account password) for Gmail |
| yq not found in pipeline | yq not installed on EC2-B | Install yq as shown in step 5.9 |

---

*This document covers a complete teaching-grade DevSecOps pipeline using AWS EC2 + Minikube. For production use, replace Minikube with EKS, Docker Hub with ECR, and add WAF, VPC Flow Logs, AWS GuardDuty, and Secrets Manager.*
