# Day 17 : What is Jenkins? Jenkins Installation Options, Docker Setup & Freestyle Job

---
## ⭐ Support the Project  
If this **repository** helps you, give it a ⭐ to show your support and help others discover it! 

---

## Table of Contents

* [Introduction](#introduction)  
* [What is Jenkins](#what-is-jenkins)  
* [Jenkins Installation Options (Local to Enterprise)](#jenkins-installation-options-local-to-enterprise)  
  * [1) Docker Container](#1-docker-container)  
  * [2) VM / Bare Metal](#2-vm--bare-metal)  
  * [3) Kubernetes (Jenkins on K8s)](#3-kubernetes-jenkins-on-k8s)  
  * [4) Cloud Marketplace Images (AWS/Azure/GCP)](#4-cloud-marketplace-images-awsazuregcp)  
  * [5) CloudBees Jenkins (Enterprise CI)](#5-cloudbees-jenkins-enterprise-ci)  
* [Quick chooser (when to pick what)](#quick-chooser-when-to-pick-what). 
* [Lab setup: Jenkins in Docker](#lab-setup-jenkins-in-docker)  
  * [Understanding `/var/jenkins_home` (your Jenkins state)](#understanding-varjenkins_home-your-jenkins-state)  
* [Who uses Jenkins: roles and responsibilities](#who-uses-jenkins-roles-and-responsibilities)  
* [First steps with Jenkins jobs (start with Freestyle)](#first-steps-with-jenkins-jobs-start-with-freestyle)  
  * [Step 0 — Create the job](#step-0--create-the-job)  
  * [Step 1 — Where do commands run? (Execute Shell)](#step-1--where-do-commands-run-execute-shell)  
  * [Step 2 — Produce and archive an artifact](#step-2--produce-and-archive-an-artifact)  
  * [Step 3 — SCM checkout (Git) + tool requirements](#step-3--scm-checkout-git--tool-requirements)  
  * [Using Maven (two beginner-friendly approaches)](#using-maven-two-beginner-friendly-approaches)  
    * [A) Manage Jenkins → Tools (preferred)](#a-manage-jenkins--tools-preferred-for-freestylepipeline-no-os-install-needed)  
    * [B) OS-level install on the node](#b-os-level-install-on-the-node-quick-lab-hack)  
  * [Which to pick (simple rule)](#which-to-pick-simple-rule)  
  * [Key points to remember](#key-points-to-remember)  
* [Conclusion](#conclusion)  
* [References](#references)  

---

## Introduction

Today we move from “why CI/CD” to **hands-on with Jenkins**: we’ll set up Jenkins (starting with a **Docker lab**), explain **installation options** (Docker, VM/Bare Metal, Kubernetes, Cloud Marketplace, CloudBees), and build your **first Freestyle job** to learn where commands run, how artifacts work, and how tools like Git/Maven are provided. By the end, you’ll have a working Jenkins you can reuse in later sessions.

---

## What is Jenkins

![Alt text](3b.png)

**Open-source automation server**
Jenkins is a free, community-maintained automation server that runs your build, test, and delivery steps. It centralizes these tasks so teams don’t rely on ad-hoc scripts on developer machines, improving repeatability and visibility.

**CI/CD orchestrator**
At its core, Jenkins automates the path from **code → build → test → package → deploy**. You define the steps; Jenkins schedules and executes them on available workers, surfaces logs and results, and gates promotions with required checks.

**Pipelines as code (`Jenkinsfile`)**
A `Jenkinsfile` stored in your repo describes the pipeline (stages, steps, conditions). Because it lives with the code, it’s versioned, code-reviewed, and reproducible. Moving a repo between environments brings its pipeline logic along with it.

**Rich plugin ecosystem**
Jenkins integrates with GitHub/GitLab/Bitbucket, Docker/OCI registries, cloud providers, scanners (SAST/DAST), test reporters, notifications, and more. Most real-world needs are covered by well-known plugins, so you compose capabilities instead of reinventing them. You can integrate **cloud services wherever a plugin exists**.

* **AWS (examples):** Amazon ECR, EC2 Agents, Secrets Manager Credentials.
* **Azure (examples):** Azure VM Agents, Key Vault Credentials, Azure Container Registry.
* **GCP (examples):** Google Compute Engine, Google Cloud Storage, Google OAuth Credentials.


**Works with your Git workflow**
Jenkins reacts to **webhooks** from your Git host (push/PR/tag events); those events **trigger** the matching jobs/pipelines in Jenkins (or a multibranch scan). If webhooks aren’t available, it falls back to **SCM polling**. Jenkins runs CI on feature branches, sets **status checks** on PRs, enforces branch protections, and on approval can publish artifacts/images and promote by environment. Fits both **GitFlow** and **Trunk-Based** models with path filters and auto-cancel of superseded builds.

Flow (crisp):
Developer **pushes to feature** → **opens PR to `dev`** → Git host **sends webhook** → Jenkins **triggers pipeline** → **checks green + reviews** → **merge to `dev`**.

>Note: Your **Git host must reach your Jenkins webhook endpoint** (public URL or reachable over VPN/VPC/ingress) for this to work; otherwise use **SCM polling** as a fallback.



**Runs almost anywhere**
You can host Jenkins on a laptop (Docker), on a VM/bare-metal server, or inside Kubernetes. This flexibility is why you’ll see multiple installation methods—each trades off **speed, control, and scale** depending on your environment.

**Controller and agents (simple view)**
Think **Jenkins controller** as the conductor (UI, scheduling, job metadata) and **agents** as workers that run jobs. In **local/lab setups**, a **single machine often runs both** controller and builds (control + data plane on one box). In **enterprise setups**, these are **separated**: the controller stays lightweight and **agents scale out** on VMs, containers, or Kubernetes Pods based on load—without redesigning your pipelines.


**Credentials and policies (practical basics)**
Jenkins stores credentials (tokens, SSH keys, cloud creds) centrally and injects them only when needed. For stronger secret hygiene, integrate with external managers—e.g., **HashiCorp Vault**, **AWS Secrets Manager**, **Azure Key Vault**, **Google Secret Manager**, or **Kubernetes Secrets** (ideally via **External Secrets Operator**). Combine this with **branch protection** and **required status checks** to prevent unreviewed or failing code from being promoted.


**Free core, enterprise option (CloudBees):**
Jenkins is a **community-governed** project under the **Continuous Delivery Foundation (CDF)**, with public governance and Linux Foundation stewardship. CloudBees is the **largest corporate backer** and offers the commercial distribution **CloudBees CI** with support, governance, and scale features—but it does **not own** Jenkins OSS. You’ll also see CloudBees content featured on jenkins.io alongside community materials, reflecting sponsorship and contributor overlap.

---

## **Jenkins Installation Options (Local to Enterprise)**

Jenkins can be set up in multiple ways—**from laptop labs to enterprise clusters**—depending on your needs for **speed, control, and scale**. The sections below outline each method (Docker, VM/Bare Metal, Kubernetes, Cloud Marketplace, CloudBees), when to use it, and key trade-offs.

![Alt text](3c.png)
---

### 1) Docker Container

**Fast setup:** Pull the official `jenkins/jenkins:lts` image and you’re running in minutes; ideal for labs, POCs, and local demos without provisioning VMs.
**Portable:** The container runs consistently across macOS, Linux, and Windows hosts (Docker Desktop), reducing “works on my machine” drift.
**Lightweight:** Minimal infrastructure footprint; add-on tools (Git, Docker CLI) can be layered via Dockerfile or bind-mounts.
**Con:** Single-host by default; persistence and true scale-out require extra design (external agents, external storage).

**Scaling note:** Plain Docker (and **Docker Compose**) is **single-host orchestration**—good for convenience, **not** cluster scaling. **Docker Swarm** can cluster, but adoption is limited and features are basic. In practice, teams either:

* Keep the **controller in Docker** and add **external agents** (separate VMs/containers) via SSH/JNLP, or
* Move to **Kubernetes** for elastic, pod-based agents and proper autoscaling.

---

### 2) VM / Bare Metal

**Traditional install:** Install via `apt/yum` packages or run the WAR as a service; predictable for enterprises with locked-down hosts and CM tools (Ansible/Chef).
**Full control:** You manage OS packages, JVM flags, file descriptors, and disk layouts; helpful for compliance audits and tuning.
**Enterprise fit:** Common on-prem where security teams require hardened base images, restricted egress, and approved artifacts.
**Con:** Scaling and upgrades are manual; horizontal growth requires more hosts and coordinated maintenance windows.

**Example (Debian/Ubuntu):**

```bash
# Jenkins repo & install
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc >/dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list
sudo apt-get update && sudo apt-get install -y openjdk-17-jre-headless jenkins
sudo systemctl enable --now jenkins
```

**Notes:**

* Place `JENKINS_HOME` on resilient storage (LVM, RAID, snapshotting).
* Use systemd overrides for JVM memory (`-Xms/-Xmx`) and time zone.

---

### 3) Kubernetes (Jenkins on K8s)

**Cloud-native:** Run the controller as a Deployment/StatefulSet; ingress, storage, and configs are standard K8s resources, making ops uniform with other apps.
**Scalable:** Spin up **ephemeral agent Pods** per build via the Kubernetes plugin; scale to zero idle agents and pay only for active builds.
**Enterprise-ready:** Best where clusters already exist; integrates with secrets managers, network policies, and cloud storage classes.
**Con:** Operationally complex; requires K8s expertise (storage, ingress, RBAC, autoscaling) and careful chart/values management.

>Not every organization runs Kubernetes. If your applications deploy to **VMs or serverless** (and not to K8s), running Jenkins **on Kubernetes adds no value**. In such cases, prefer a **VM/bare-metal Jenkins controller** with VM-based agents (or cloud agents) and keep CD/CDp targeting the same platforms you deploy to.


**Example (Helm, community chart):**

```bash
helm repo add jenkins https://charts.jenkins.io && helm repo update
helm upgrade --install jenkins jenkins/jenkins \
  --namespace ci --create-namespace \
  --set controller.serviceType=ClusterIP \
  --set controller.jenkinsUriPrefix="/" \
  --set controller.resources.requests.cpu=500m \
  --set controller.resources.requests.memory=1Gi \
  --set controller.jenkinsAdminPassword=admin123 \
  --set persistence.size=20Gi
```

**Notes:**

* Use a **StorageClass** with snapshots; back up PVCs and configuration as code (JCasc).
* Prefer ingress + TLS; lock down with NetworkPolicies; run agents as non-root.

---

## 4) Cloud Marketplace Images (AWS/Azure/GCP)

**One-click deploy:** Prebuilt VM images/templates spin up Jenkins fast—great for pilots, PoCs, or lift-and-shift.
**Cloud-integrated:** Easier wiring to managed disks, load balancers, logging/monitoring; many offers include hardened baselines.
**Enterprise usage:** Used where teams want **BYOL or managed Jenkins** from vendors/MSPs (including **CloudBees CI** listings). Common as a sanctioned **starting point**, not always the long-term scale platform.
**Con:** Vendor coupling; less flexibility than DIY VM or Kubernetes.

**Notes:**

* If you need **supported Jenkins**, **CloudBees CI** is available via cloud marketplaces (including government regions), typically **BYOL with enterprise support**; pricing is usually **quote-based/private offer**.
* Many enterprises still standardize on **VM or Kubernetes** installs for long-term governance and scale; marketplace images excel at **speed to value**, not necessarily as the final architecture.
* **Starting fresh on cloud?** Check the marketplace to accelerate provisioning, then harden and govern as you scale.

---

### Quick chooser (when to pick what)

* **Docker:** Fast start for labs/dev laptops/small teams; single-host by default (Compose ≠ cluster). Scale via external agents or move to K8s later.
* **VM / Bare Metal:** Regulated/on-prem with full OS control and stable scale; predictable, but scaling/upgrades are manual.
* **Kubernetes:** Cloud-native elasticity with ephemeral agent Pods and GitOps; best if your org already runs K8s—otherwise unnecessary complexity.
* **Cloud Marketplace:** Rapid cloud start with managed integrations and hardened images; great for pilots/PoCs and BYOL, but tighter vendor coupling—often not the final long-term platform.
* **CloudBees (Enterprise Jenkins):** Governance, RBAC at scale, curated plugins, SLAs, Ops Center; ideal for large/regulated estates. Can add support to existing OSS Jenkins now and migrate to CloudBees CI later.

---


## Lab setup: Jenkins in Docker

We’ll learn the **basics on a single-node Jenkins running in Docker**—fast to spin up, easy to reset.
When we switch to **projects and production patterns**, we’ll use **VM-based** and **Kubernetes-based** installs.

### Run Jenkins (Docker)

```bash
docker run -d \
  --name jenkins \
  --restart unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -e TZ=Asia/Kathmandu \
  jenkins/jenkins:lts
```

**Flag breakdown (quick)**

* `-d` : background container.
* `--restart unless-stopped` : auto-start on Docker restart.
* `--name jenkins` : easy to reference.
* `-p 8080:8080` : UI at `http://localhost:8080`.
* `-p 50000:50000` : inbound agent port (JNLP).
* `-v jenkins_home:/var/jenkins_home` : **persistent data** (jobs, plugins, config).
* `-e TZ` : set container timezone.
* `jenkins/jenkins:lts` : stable LTS image.

> Note: We **don’t** set `-u root` here—Jenkins runs as the default non-root user

### First-time setup

1. `docker logs jenkins` → copy **initialAdminPassword**.
2. Open `http://localhost:8080` → install **recommended plugins**.
3. Create admin user → you’re ready to create jobs/pipelines.

### Handy commands

* Exec shell: `docker exec -it jenkins bash`
* Restart: `docker restart jenkins`
* Stop/Start: `docker stop jenkins && docker start jenkins`

---

### Understanding `/var/jenkins_home` (your Jenkins **state**)

This path holds the **controller’s persisted state**—not the build “data plane”. Everything the controller needs lives here: **jobs/pipelines, build history/logs, plugins, global config, credentials, node definitions, UI settings**, etc. Keep it **outside the container FS** so a container crash/recreate doesn’t lose data.

In our lab, we mount a **Docker named volume**:

```bash
-v jenkins_home:/var/jenkins_home
```

If the container dies or is replaced, reattach the same volume and Jenkins restores instantly. In production, consider **bind mounts or network/block storage** with snapshots and encryption.

**Tips**

* Treat as **sensitive** (contains secrets). Back up and restrict access.
* Keep it **lean** by offloading heavy workspaces to agents; prune old builds.
* Test **backup/restore** regularly.

> **Note:** If `/var/jenkins_home` is on a persistent volume and you reuse that same volume when recreating the container, **all prior state returns**—jobs, build history, credentials, configs, and **plugins**. Keep the Jenkins image/version consistent, ensure ownership (`jenkins:jenkins`), and back up before changes.

---

## Who uses Jenkins: roles and responsibilities

![Alt text](3d.png)

**Jenkins Administrators (DevOps):**

* Own the platform: install/upgrade Jenkins, manage plugins, backups/DR, HA, credentials stores, security (RBAC/SSO), agents, and integrations.
* Use **Manage Jenkins** for global config, toolchains (JDK/Git/Docker), nodes/agents, system logs, and configuration-as-code.
* Typically part of the **DevOps/Platform** team; ensure reliability, governance, and compliance.

**Jenkins Users (Developers/DevOps/QA):**

* Consume the platform to **automate builds, tests, packaging, and deployments**.
* Create jobs and pipelines via **New Item** (Freestyle or **Pipeline/Jenkinsfile**), manage stages, credentials usage, and post-build actions.
* Treat pipelines as code, integrate status checks with PRs, and own application-level automation.

---

## First steps with Jenkins jobs (start with Freestyle)

When you click **New Item**, Jenkins offers several job types (Pipeline, Multibranch Pipeline, Freestyle, Folder, etc.). We’ll start with the **simplest, most beginner-friendly option: Freestyle**. In this demo, we’ll create **one Freestyle job** and progressively edit it to learn the essentials: where commands run, how workspaces and environment variables behave, saving **artifacts**, connecting **SCM (Git)**, understanding **tool requirements** (Git/Maven/Helm), and adding **parameters** to make the job configurable.


---

### Step 0 — Create the job

* **New Item → Freestyle project → Name:** `intro-freestyle` → **OK**.
* For now, leave SCM = *None*.

---

### Step 1 — Where do commands run? (Execute Shell)

**Build → Add build step → Execute shell:**

```bash
echo "Node info:"
uname -a

echo "Workspace: $WORKSPACE"

echo "Build: $JOB_NAME #$BUILD_NUMBER"

echo "Node: $NODE_NAME"
```

**Build Now → Console Output:**
You’ll see details from the **Jenkins node** this job ran on. In our lab, that’s the **Jenkins container** (controller).

> **Takeaway:** Freestyle steps run **on the node** assigned to the job. Today it’s the controller; later we’ll use **agents**.

**Built-in variables (quick peek):**
Common ones you’ll use: `WORKSPACE` (workspace path), `JOB_NAME`, `BUILD_NUMBER`, `BUILD_ID`, `BUILD_URL`, `NODE_NAME`.
With Git SCM configured: `GIT_COMMIT`, `GIT_BRANCH`, `GIT_URL`.

**See all built-in env vars (UI):**
Open **`<JENKINS_URL>/env-vars.html`** (e.g., `http://localhost:8080/env-vars.html`) for Jenkins’ reference list of core environment variables.

**Dump all env vars (from a build):**

```bash
echo "---- ALL ENV VARS ----"
printenv | sort
```
> **Why print build details?**
>
> * **Traceability:** Link logs/artifacts to the exact run (`JOB_NAME`, `BUILD_NUMBER`, `GIT_COMMIT`).
> * **Debugging:** Show execution context (`NODE_NAME`, `WORKSPACE`, OS) to speed root-cause.
> * **Reproducibility/Audit:** Capture versions, params, and env to re-run or prove what shipped.


---

### Step 2 — Produce and archive an artifact

* **Edit job → Build → Execute shell (append):**

  ```bash
  echo "Report for build #$BUILD_NUMBER" > build-report.txt
  ls -l
  ```
* **Post-build Actions → Archive the artifacts:** `build-report.txt`
* **Build Now → Open the build → “Artifacts” tab:** download **build-report.txt**.

  > Takeaway: **Artifacts** are files saved from `$WORKSPACE` with the build record.

---

### Step 3 — SCM checkout (Git) + tool requirements

* **Edit job → Source Code Management → Git**

  * **Repository URL:** `<your repo>` (HTTPS works).
  * **Credentials:** Add if private.

* **Build → Execute shell (replace commands):**

  ```bash
  echo "Repo status:"
  git --version
  git log --oneline -n 1 || true
  ```

* **Build Now → Console Output:**
  If you see `git: command not found`, the **node** running this job doesn’t have Git.

**Fix options for Git (same pattern applies to any CLI):**

* **Simple (lab):** install on the node

  ```bash
  # inside the Jenkins container (controller) for our lab
  apt-get update && apt-get install -y git
  ```
* **Proper (admin):** use **Manage Jenkins → Tools** to define **Git** versions, or run the job on an **agent** image/VM that already includes Git.

> **Takeaway:** Jenkins can only run CLIs that exist **on the node** (or are provided via Tools for that build). No Maven → no `mvn`; no Helm → no `helm`.

---

## Using **Maven** (two beginner-friendly approaches)

#### A) **Manage Jenkins → Tools** (preferred for Freestyle/Pipeline; no OS install needed)

* **Configure Maven tool once:**
  **Manage Jenkins → Tools → Maven installations → Add Maven**

  * **Name:** `M3`
  * **Install automatically:** ✅ pick a version
* **Freestyle job:**
  **Add build step → Invoke top-level Maven targets**

  * **Maven Version:** `M3`
  * **Goals:** `-B clean test package` *(or `-v` to print version)*
    → Jenkins wires Maven into PATH **for this build**, even if the node doesn’t have `mvn` installed.
* **Pipeline (for later):**

  ```groovy
  pipeline {
    agent any
    tools { maven 'M3' }
    stages { stage('Check') { steps { sh 'mvn -v' } } }
  }
  ```

**Pros:** Versioned/reproducible, no root access, works on clean nodes.
**Gotcha:** A plain **Execute shell** in Freestyle won’t see `mvn` unless you either (1) use the **Maven build step** as above, or (2) install a small bridge like the **Tool Environment** plugin to expose tools to PATH.

---

#### B) **OS-level install on the node** (quick lab hack)

* Install Maven directly where the job runs (controller or agent):

  ```bash
  # inside the Jenkins container (controller) for our lab
  apt-get update && apt-get install -y maven
  mvn -v
  ```
* Then your Freestyle **Execute shell** can run:

  ```bash
  mvn -B clean test package
  ```

**Pros:** Works with any plain shell step immediately.
**Cons:** You manage versions manually; can drift across nodes; less reproducible.

---

### Which to pick (simple rule)

* **Learning/labs or teams seeking reproducibility:** **Use Tools** (A). Pick versions per job; no OS tinkering.
* **Quick local demo or a one-off node:** **OS install** (B) is fine—just know it doesn’t scale as cleanly.

> **Tip:** To *only* print Maven’s version:
>
> * Tools way (Freestyle): “Invoke top-level Maven targets” with **Goals:** `-v`
> * OS way: add **Execute shell** with `mvn -v` after installing Maven on the node.


---

### Key points to remember

* **Node of execution:** Freestyle steps run on the assigned node (today = controller). For real projects, attach **agents** (VMs/containers/K8s Pods) with the required toolchain.
* **Tools live on nodes:** If the node lacks **Git/Maven/Helm**, commands will fail. Use **Manage Jenkins → Tools** or provide an agent image/VM that has them.
* **Artifacts:** Archive outputs you need to keep (reports, binaries); they’re attached to the build record.
* **Parameters:** Make jobs reusable; values show up as env vars.
* **Security & scaling:** Avoid piling every tool into the controller. Prefer **agents** and keep the controller lean and stable.

This single job, edited in stages, gives beginners a clean tour of **execution context**, **workspaces/env vars**, **artifacts**, **SCM**, **tooling**, and **parameters**—without overwhelming them.

---

## Conclusion

You now have a **running Jenkins** (Docker lab), understand **when to choose** each install method, and have created a **Freestyle job** that prints environment details, archives artifacts, and checks out code—plus you know the difference between **OS-level tools** and **Tools-managed** integrations. Next up: **Day 4**—introducing **Pipelines (Jenkinsfile)**, agents, and promoting artifacts/images by **immutable digest**.

---


