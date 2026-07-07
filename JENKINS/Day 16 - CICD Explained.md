# Day 16 - CICD Explained | How Pipelines Work & Branching Strategies

---
## ⭐ Support the Project  
If this **repository** helps you, give it a ⭐ to show your support and help others discover it! 

---

## Table of Contents

- [Introduction](#introduction)
- [Git & Branching: The Essentials for CI/CD](#git--branching-the-essentials-for-cicd)
  - [What is Git?](#what-is-git)
  - [Where to host your Git repo (remotes)](#where-to-host-your-git-repo-remotes)
  - [What are branches?](#what-are-branches)
  - [What is a PR/MR?](#what-is-a-prmr)
  - [Protected branches & status checks](#protected-branches--status-checks)
  - [How Git fits CI/CD (quick mental model)](#how-git-fits-cicd-quick-mental-model)
  - [Quick glossary](#quick-glossary)
- [Continuous Practices in DevOps](#continuous-practices-in-devops)
  - [What is CI/CD?](#what-is-cicd)
  - [The core “continuous” practices (10,000-ft view)](#the-core-continuous-practices-10000-ft-view)
    - [1. Continuous Integration (CI)](#1-continuous-integration-ci)
    - [2. Continuous Testing (CT)](#2-continuous-testing-ct)
    - [3. Continuous Delivery (CD)](#3-continuous-delivery-cd)
    - [4. Continuous Deployment (CDp)](#4-continuous-deployment-cdp)
    - [5. Continuous Monitoring (CM)](#5-continuous-monitoring-cm)
- [Continuous Integration (CI)](#continuous-integration-ci-1)
  - [Where CI fits in your branching strategy](#where-ci-fits-in-your-branching-strategy)
- [GitFlow-Based CI/CD: Environment Branches](#gitflow-based-cicd-environment-branches)
  - [Step-by-step (numbers match the diagram)](#step-by-step-numbers-match-the-diagram)
- [Promotion PR: `dev → stage` (Deploy-Before-Merge Flow)](#promotion-pr-dev--stage-deploy-before-merge-flow)
  - [What happened earlier on `dev` (input to promotion)](#what-happened-earlier-on-dev-input-to-promotion)
  - [Stage PR gates (repo rules)](#stage-pr-gates-repo-rules)
  - [1 Open PR: `dev → stage`](#1-open-pr-dev--stage)
  - [2 Stage Promotion Pipeline (runs on the PR)](#2-stage-promotion-pipeline-runs-on-the-pr)
  - [3 Stage environment (what’s running)](#3-stage-environment-whats-running)
  - [4 PR status & merge decision](#4-pr-status--merge-decision)
  - [Why the strategies differ: speed vs. safety](#why-the-strategies-differ-speed-vs-safety)
- [Promotion PR: `stage → prod` (Production Gate)](#promotion-pr-stage--prod-production-gate)
- [Trunk-Based CI/CD: Environment Promotions](#trunk-based-cicd-environment-promotions)
  - [Diagram at a glance](#diagram-at-a-glance)
  - [Guardrails on `main` (protected branch)](#guardrails-on-main-protected-branch)
  - [End-to-end flow (numbers match the diagram)](#end-to-end-flow-numbers-match-the-diagram)
- [Jenkins in this picture (why & what)](#jenkins-in-this-picture-why--what)
- [Conclusion](#conclusion)
- [References](#references)

---

## Introduction

Welcome back to **Jenkins: Basics to Production 
In previous lecture, we answered *why* reliable builds matter. Today we connect that to the bigger picture: **how code travels from a laptop to Production** using **Git, CI/CD, and environment promotions**, and **where Jenkins fits**.

**By the end of this lecture you’ll be able to:**

* Explain the core “continuous” terms: **CI, CT, CD, CDp, CM** (and how they differ).
* Compare **GitFlow** vs **Trunk-based** workflows and know when each is used.
* Read the **promotion diagrams** and narrate the steps (Feature → Dev → Stage → Prod).
* Defend the principle **“build once, promote the same digest.”**
* Describe **branch protection** and **PR status checks** that gate merges.
* Place **Jenkins** in the flow: what it triggers, builds, publishes, and promotes.


---


## Git & Branching: The Essentials for CI/CD

![Alt text](2a.png)

### What is Git?

**Git** is a **distributed version control system (DVCS)**. It lets many people work on the same codebase, track every change, and safely move back and forth in history. Because it’s *distributed*, every developer’s laptop has the full repo history—so you can branch, compare, and commit offline, then sync to a shared **remote** later.

* A **commit** is a saved snapshot of your changes with a message and unique ID (SHA).
* **Push** sends your local commits to the remote so others can review them.
* **Pull** brings down new commits from the remote so your laptop is up to date.

---

### Where to host your Git repo (remotes)

You use `git` locally, then push to a hosting service for collaboration, reviews, and CI/CD.

* **Cloud hosts:** **GitHub**, **GitLab**, **Bitbucket**
* **Provider suites:** **Azure Repos** *(some teams also use AWS CodeCommit / the former GCP CSR; most new teams standardize on the hosts above)*
* **Self-hosted:** **GitLab CE/EE**, **Gitea**, **Gerrit**, **SourceHut**

These add PR/MR reviews, permissions, issue tracking, runners, and CI/CD integrations (e.g., Jenkins).

---

### What are branches?

A **branch** is a lightweight pointer (a moving label) to a line of work.

* **`main`** (or `master`): the trunk, source of truth
* **Feature branches:** short-lived work, e.g., `feature/topup-loan`
* **Environment/release branches (GitFlow):** `dev`, `stage`, `prod` (or `release/x.y`)

You commit on your branch, then propose merging into a **protected** branch.

---

### What is a PR/MR?

A **Pull Request (PR)** / **Merge Request (MR)** is a **proposed change** from one branch into another with **review + automated checks**.

* Open PR → reviewers comment/approve → CI runs (build/tests/scans) → **status** posts (green/red)
* **Green** enables the merge; **red** blocks it until you fix and push again
* Names vary: GitHub/Azure/Bitbucket use **PR**; GitLab uses **MR** (same idea)

---

### Protected branches & status checks

Protect important branches (`main`, `stage`, `prod`) so they stay clean and deployable:

* **PRs only + required reviews:** You can’t push straight to `main`/`prod`; you **open a PR** and get **approvals** (e.g., **1** for `main`, **2** for `prod`).
  *Example:* Varun opens a PR → Seema approves → only then can it merge.

* **CI must pass (green checks):** The PR won’t merge unless **build**, **unit tests**, and a **quick security scan** are green.
  *Example:* “build”, “tests”, “sec-scan” all show ✅; if any is ❌, fix and push again.

* **Up-to-date with base:** The PR must include the **latest `main`/`prod`** before merging.
  *Example:* `main` moved while your PR was open → click **Update branch** (or rebase), CI re-runs, then merge.


---

### How Git fits CI/CD (quick mental model)

* **Git** stores changes (commits/branches).
* **CI** runs on pushes/PRs to **build & test automatically** and post a status that **gates merging**.
* **CD** promotes the **built artifact** through **Dev → Staging → Prod**, with approvals/rollouts depending on your branching strategy (GitFlow vs Trunk-Based).

---

### Quick glossary

* **Commit:** snapshot + message; identified by a **SHA** (e.g., `9f3c2e7`)
* **Tag/Release:** human label for a commit (e.g., `v1.4.3`) used for packaging/changelogs
* **Merge vs Rebase:** both integrate history; *merge* creates a merge commit, *rebase* rewrites your branch onto a new base (cleaner history—use carefully)
* **Clone / Pull / Push:** copy a remote repo; fetch/merge updates; publish your commits
* **Fork:** your own copy of someone else’s repo (common in open source) → PR back



---

## Continuous Practices in DevOps

Modern DevOps isn’t just about writing code—it’s about moving changes **from laptop → production → monitoring** in a **continuous loop**, learning from what’s running, and repeating.

### What is CI/CD?

* **Continuous Integration (CI):** automatically **builds and tests every change** as soon as it hits Git, then posts a green/red status that **gates the merge**.
* **Continuous Delivery/Deployment (CD):** automatically **promotes and deploys the CI-built, immutable artifact** across environments (Dev → Staging → Prod) using environment-specific config, gates, and rollout strategies.
  **CD** typically requires a human approval for Prod; **CDp** deploys to Prod automatically once checks pass.

> Note: Some teams add “release metadata” in CD (tagging, signing, SBOM attach), but they still **don’t rebuild**—the artifact digest stays the same.

--- 

### The core “continuous” practices (10,000-ft view)

![Alt text](2b.png)

#### 1. Continuous Integration (CI)

* **Build & test every push/PR automatically.** Each commit/PR kicks off a pipeline that checks out code, installs deps, and runs scripted steps without human action.
* **Fast checks enforce merge protection on the branch.** CI posts green/red status on the PR; branch protection enables Merge only when CI is green, keeping `dev/main` clean.
* **Compile, unit test, quick security/dependency scans.** Keep it lean (\~10–15 min): compile/package, unit tests, lint, and quick SAST/SCA or dep checks to catch obvious breakages early.
* **Tools:** Jenkins, GitHub Actions, GitLab CI.

#### 2. Continuous Testing (CT)

* **Testing across the pipeline, not just CI.** CT is the umbrella: it includes CI’s quick checks plus broader tests run before deploy and after deploy.
    >**Pipeline:** an automated, repeatable sequence of steps—**checkout → build → test → package → (publish/deploy)**—triggered by a push/PR to validate and deliver changes.
* **Add integration, API/UI, performance, security suites.** Contract/API tests, UI end-to-end flows, load/perf, and security (DAST) validate behavior beyond unit scope.
* **Run pre-merge, pre-deploy, and post-deploy smoke/synthetics.** Gate releases with deeper suites and verify live systems with smoke/synthetic monitors.
* **Tools:** JUnit/TestNG, Cypress/Playwright, Postman/Newman, JMeter/k6, OWASP ZAP.

#### 3. Continuous Delivery (CD)

* **Keep `main` always deployable with versioned artifacts.** Every green build is a shippable artifact tagged and traceable (build once, run everywhere).
* **Promote the same artifact across environments.** Move the exact image/JAR through Dev → Staging with config only; no rebuilds, fewer surprises.
* **Use human approvals/gates before production.** Releases become a decision (not a project) with checks, sign-offs, and change records.
* **Tools:** Jenkins Pipelines, GitHub Actions Environments, GitLab CI/CD.

#### 4. Continuous Deployment (CDp)

* **Auto-deploy to production when all checks pass.** No manual approval—pipelines push straight to prod after green signals.
* **Use progressive delivery for safety.** Canary/blue-green rollouts and feature flags reduce blast radius and enable quick enable/disable.
* **Rollback fast on bad signals.** Health checks and KPIs/SLOs trigger automated rollback or flag-off to restore service quickly.
* **Tools:** Argo CD, Flux, Spinnaker (feature flags: LaunchDarkly/Flagsmith optional).

#### 5. Continuous Monitoring (CM)

* **Observability:** The three pillars are **metrics, logs, and traces**—your raw **telemetry/signals**. Observability is useful only when you **apply rules** (alerts, SLOs, dashboards) to turn those signals into action.

    * **Collect telemetry (metrics, logs, traces) continuously—and make it actionable with rules (alerts/SLOs), not just raw data.** Instrument apps/infra to see behavior in real time.

* **Visualize health and manage SLOs/error budgets.** Dashboards + alerting highlight regressions and user impact early.
* **Close the loop: findings feed back into CI/CT.** Incidents and trends create tests, rules, and guardrails for the next change.
* **Tools:** Prometheus/Grafana, Elastic Stack (ELK), Datadog, New Relic.


> **Mental model:** CI runs the **fast, merge-gating subset** of tests/scans; CT is the **lifecycle umbrella** for all testing. CD keeps releases **always ready**; CDp **ships automatically**; CM **watches everything** and informs the next change.

> **Mental model:** **CI proves a change is safe to merge. CT widens the safety net. CD keeps releases always ready. CDp ships automatically. CM watches everything and informs the next change.**

---

## Continuous Integration (CI)

**Continuous Integration (CI)** means **every code change is automatically built and tested as soon as it’s pushed to Git**.
A CI server (e.g., Jenkins) checks out the exact commit, installs dependencies, compiles/packages the app, and runs fast automated checks (unit tests + basic static scans).
It then **posts a status back to your Git platform** (green = pass, red = fail). With **branch protection** on, **red checks block** and **green checks allow** the merge—keeping `main`/`dev` clean and deployable.
We integrate **small, frequent changes** via **pull requests** into protected branches; **CI gates the merge** so only healthy code lands.

> **Note:** “Pull Request (PR)” and “Merge Request (MR)” mean the **same thing**—a proposed change from one branch into another with review + CI checks.
> **Examples:** GitHub/Azure Repos/Bitbucket say **PR**; GitLab says **MR**. We’ll use **PR** generically.

---

### Where CI fits in your **branching strategy**

**GitFlow-based** and **Trunk-based** are **branching strategies**—i.e., version-control workflows that define **how teams use Git branches** to integrate code and ship releases. CI (build → test → status) works in both; what changes is **where** CI triggers and **how** code/artifacts move between environments.

![Alt text](2c.png)

1. **GitFlow-Based CI/CD: Environment Branches**
    * **Model:** Multiple long-lived branches mirror environments (**`dev` → `stage` → `prod`**).
    * **Flow:** Devs merge feature branches into `dev`, then raise **promotion PRs** `dev→stage` and `stage→prod`.
    * **CI:** Runs on **every PR** (feature→dev and env→env) to gate merges with green/red checks.
    * **Build:** Build once when the change is approved into `dev`; publish image + digest.
    * **Promotion:** Use PRs `dev→stage` and `stage→prod` that **deploy the same digest**; CI on those PRs gates the merge.
    * **Governance:** Strong branch protection, required reviews/approvals, clear audit trail at each hop.
    * **Trade-offs / Fit:** More merges and coordination; risk of branch drift. Common in **regulated/enterprise** and **release-train** setups.

2. **Trunk-Based CI/CD: Environment Promotions**
    * **Model:** Single long-lived **trunk (`main`)** as source of truth; **short-lived feature branches**.
    * **Flow:** Small PRs → **CI on PR into `main`** (tests the merge result) → **build once after merge** → **promote same artifact** through **Dev → Staging → Prod** (no env-branch merges).
    * **Build:** Build once on merge to `main`; publish image + **immutable digest**.
    * **Promotion:** Update the **same digest** in Dev → Staging → Prod via pipeline stages **or** GitOps PRs in an env-config repo (**no rebuild; config only**).
      > Key point: **Both strategies should “build once, promote many.”** Trunk-based differs in *branching* (single `main`, no env branches in the app repo), not in the artifact policy.
    * **Governance:** Lean process with protected `main`, required checks; Prod uses approvals, progressive rollout, and rollback.
    * **Trade-offs / Fit:** Requires discipline (small PRs, solid tests); rewards with **fewer merges and faster feedback**, ideal for **microservices/SaaS**.

Great point—here’s a cleaned-up mental model that reflects that:

> **Mental model:** In **both** styles you **build once and promote the same artifact**.
> **GitFlow:** uses **environment branches**; promotions are **PRs between branches** (dev→stage→prod) that deploy that artifact.
> **Trunk-Based:** keeps all code in **`main`**; promotions **update env config** (pipeline/GitOps) to deploy that artifact—**no env-branch merges**.

> We’ll dive into both next and show exactly where CI and CD plug in.

**Also seen in the wild (beyond these two):**

* **GitHub Flow (lightweight):** `main` + short feature branches; frequent deploys from `main` (common in SaaS/web).
* **Release Branching / Release Trains:** cut `release/x.y` to stabilize while `main` advances; cherry-pick hotfixes (mobile/desktop, SDKs).
* **Forking Workflow:** contributors work in personal forks → PRs back (open-source).
* **Monorepo variants:** trunk-based in one repo with CODEOWNERS and env promotions (often GitOps).

**What you’ll likely see in practice:**

* **Microservices (cloud/SaaS):** mostly **Trunk-Based CI/CD** with **feature flags** and **environment promotions** (sometimes GitOps).
* **Standalone web/app tiers or regulated/legacy systems:** more often **GitFlow-style environment branches** (dev → stage → prod) for auditability and gated promotions.


---

## GitFlow-Based CI/CD: Environment Branches

**Why this name:** Branches represent **environments** (`dev`, `stage`, `prod`). Code **moves by merging between branches**, and each hop is guarded by PR checks and approvals. This model is easy to visualize and audit, which is why it’s common in beginner and regulated setups.
**Flow:** **Feature → `dev` → `stage` → `prod` (Beginner Flow)**
**Tagline:** *Environment branches with merges between them; **PRs gate each hop***.
**Notes:** Clear release windows and change records; more merge overhead and longer-lived branches.

![Alt text](2d.png)

**What the diagram shows:** the journey of one change from a **feature branch** to a protected **`dev`** branch using a **PR (pull/merge request)**.
>A **protected branch** is a repository rule-set that **blocks direct pushes/force-pushes** and **allows merges only via PRs** that meet guardrails—typically **green CI status checks** and **required reviews** (often also “up to date with base,” signed commits, and linear history).

### Step-by-step (numbers match the diagram)

1. **Commit & push (feature branch)**
   You work on `feature/topup-loan`, commit locally, and **push** to Git. This publishes your change so it can be reviewed and tested. *Output:* a feature branch with new commits.

2. **Open PR to `dev` (protected)**
   The developer **manually opens a PR** from **feature → `dev`**. Because `dev` is protected: **no direct commits**, **PRs only**, and **passing status checks + required reviews**. The dotted arrow means a PR **proposes** a merge; `dev` hasn’t changed yet.
   > *Note:* A **feature branch contains only your feature work**—not the full, up-to-date app. To validate your change against everything else, you **must raise a PR** so CI can test the **merge result (`dev + feature`)** before it lands.
   *Output:* an open PR.


3. **CI Pipeline (Checkout • Build • Test)**
   Opening the PR **triggers CI**. CI builds the **merge result** of `dev + feature` in a clean workspace (so `dev` itself stays unchanged), then runs quick scripted steps—checkout, build/compile, unit tests, and basic lint/security checks. Target runtime: **≈10–15 min**. *Output:* a pass/fail result.

4. **PR Status (green/red)**
   CI posts status back to the PR:

   * **Green → Merge enabled.** All checks passed; the Merge button unlocks.
   * **Red → Merge blocked.** Something failed; you **push a fix**, CI **reruns**, and the status updates. *Output:* a decision to merge or iterate.

5. **Merge PR into `dev`**
   With green checks (and any required reviews), the PR merges. `dev` advances to a **known-good merge SHA**. Many teams tie the built artifact (image/JAR) to that SHA for traceability. *Output:* clean `dev` pointing at the merge commit.

6. **Auto-deploy to Dev (CD)**
   After the merge, the pipeline may **automatically deploy `dev`’s new version to the Dev environment** (Kubernetes/VM/Serverless) for quick smoke checks. This is **Continuous Delivery (CD)** because it’s an automated **non-prod** promotion/deploy.
   >*Clarifier:* If you **automatically deploy to Production with no human approval**, that’s **Continuous Deployment (CDp)**. If Production still requires a manual approval/gate, it remains **CD**.

   *Output:* the change running in Dev, ready for further testing.


> **Key idea:** CI runs **on the PR** to test the **merge result** *before* anything lands in `dev`. **Green enables merge; red blocks it** until you push a fix. After the merge, you can **optionally** auto-deploy to the Dev environment.

> **CI runs on the PR** to test the tentative merge (`dev + feature`). **If it’s green, you merge.** **After the merge**, a pipeline can **optionally auto-deploy the updated `dev`** to the **Dev environment**—that’s **Continuous Delivery (CD)**. (Auto-deploying to **Production** without approval would be **Continuous Deployment (CDp)**, which we’re *not* doing here.)

---

## Promotion PR: `dev → stage` (Deploy-Before-Merge Flow)

This flow promotes a **release candidate** from `dev` to **Stage**. The promotion happens **during the PR**: CI does **Preflight → Deploy → Post-deploy**. The PR turns **green only if Stage is healthy**, and **then** you merge. That keeps the `stage` history clean and auditable.

---


### What happened earlier on `dev` (input to promotion)

* **Build once & push (single image).** CI builds **one image** and may attach **human-friendly tags** (e.g., `:v1.4.3`, `:9f3c2e7`, `:9f3c2e7-v1.4.3`).
  *All of these tags point to the **same immutable digest**.*

* **Record the immutable digest (source of truth).** The registry returns a content address like `sha256:abc123…`; this **never changes** and is what you **deploy**.

* **Publish a promotion manifest.** CI writes `manifest.json` (repo, **digest**, commit, version, built\_at, optional tags) to a **read-only location** Stage/Prod can read (e.g., GitHub Release, CI artifact, S3).

* **Deploy by digest (not tags).** Dev/Stage/Prod jobs read the manifest, **verify the digest**, and **deploy that digest** with env-specific config—**no rebuilds** (digest preferred even if tags are immutable).

**Example `manifest.json`:**

```json
{
  "image": "ghcr.io/acme/banking-loans",
  "digest": "sha256:abc123...",
  "version": "v1.4.3",
  "commit_sha": "9f3c2e7",
  "tags": ["v1.4.3", "9f3c2e7", "9f3c2e7-v1.4.3"],
  "built_at": "2025-09-15T10:22:00Z"
}
```

**Why keep multiple tags (even though you deploy by digest)?**

* **`:v1.4.3` (release):** easy for Product/QA/Support to map incidents and read changelogs/dashboards.
* **`:9f3c2e7` (commit):** engineers/SREs can jump to the exact source for bisects/hotfixes.
* **`:9f3c2e7-v1.4.3` (combined):** best of both at a glance in registry UIs.

> **TL;DR:** **Build once**, **tag for humans**, **deploy by digest** everywhere.


---

### Stage PR gates (repo rules)

Configure these on the `stage` branch so a PR **cannot merge** until the right things are true:

* **PRs only** (block direct/force pushes).
* **Up-to-date with base** (PR includes the latest `stage` tip; use *Update branch*).
* **Recent Dev run green** for the promoted `commit_sha` (status like `dev-build-publish`).
* **Require these statuses to pass** (produced by the pipeline below):
  `preflight-manifest`, `preflight-config`, `deploy-to-stage`, `stage-smoke`, `stage-integration`.
* **Required review (≥1)** (TL/QA/owner).

> These are *repo gates*. They’re cheap checks and required statuses; if any are red, the PR **can’t merge**.

---

![Alt text](2e.png)

### 1 Open PR: `dev → stage`

Open a **promotion PR** that references the **exact release candidate** by its **immutable digest** (tags are optional, for humans). Stage CI will read the manifest and **deploy by digest**—**no rebuilds**.

**Good PR description template:**

```
Promote to Stage

image: ghcr.io/acme/banking-loans
digest: sha256:abc123…          # deployments use this (immutable)
version: v1.4.3                 # optional, human-friendly
commit_sha: 9f3c2e7             # source traceability
tags: [v1.4.3, 9f3c2e7, 9f3c2e7-v1.4.3]  # optional, for discoverability
manifest: <read-only link>      # where Stage reads image+digest
```

*Output:* PR is open; the **`stage` branch remains unchanged**. (Promotion pipeline will validate and deploy **by digest** during this PR; merge only when all checks are green.)


---

### 2 Stage Promotion Pipeline (runs on the PR)

**2.1 Preflight (fast fail in seconds)**
CI proves the release *can* be deployed before touching Stage:

* **Fetch `manifest.json`** (S3 / repo / PR attachment) and validate schema (image, **digest**, commit).
* **Resolve & verify digest is pullable** (`repo@sha256:…`).
* **Render & validate Stage config**:
  `helm lint` → `helm template | kubectl apply --dry-run=server -f -`.

**2.2 Deploy (no rebuild; by digest with Stage config only)**

```bash
IMAGE=$(jq -r '.image'  manifest.json)
DIGEST=$(jq -r '.digest' manifest.json)

helm upgrade --install loans ./charts/loans \
  --set image.repository="$IMAGE" \
  --set image.digest="$DIGEST" \
  -f charts/loans/values-stage.yaml
# Chart should build: image: {{ .Values.image.repository }}@{{ .Values.image.digest }}
```

*Deployed image:* `ghcr.io/acme/banking-loans@sha256:abc123…` (immutable; **not** a tag)

**2.3 Post-deploy (only what needs a live Stage)**

* **Smoke/health:** readiness/liveness; `/version` shows `v1.4.3 (9f3c2e7)`.
* **Small integration/e2e:** 1–2 happy paths (e.g., *create loan → fetch*).

*Output:* CI posts the required commit statuses back to the PR:
`preflight-manifest`, `preflight-config`, `deploy-to-stage`, `stage-smoke`, `stage-integration`.

> If all statuses are green and review is approved, the PR can merge. If any fail, the PR stays red; fix in `dev`, produce a new digest, and re-promote.

---

### 3 Stage environment (what’s running)

The **exact digest** from the manifest is now running in Stage, configured with Stage values/secrets. The **`stage` branch is still unchanged**; this is a dress rehearsal of the release.

---

### 4 PR status & merge decision

* **Green → Merge enabled.**
  All required checks passed and ≥1 reviewer approved. Merge to record the promotion in `stage`. (No new deploy is needed; Stage is already on that digest.)

* **Red → Merge blocked.**
  Something failed. **Roll back Stage** (e.g., `helm rollback loans <rev>`), **fix on `dev`**, produce a new digest, and **re-promote** with a fresh PR or an updated one.

> **Required checks to configure:** `preflight-manifest`, `preflight-config`, `deploy-to-stage`, `stage-smoke`, `stage-integration`, plus “Up-to-date with base” and “Required review”.

---

### Why the strategies differ: **speed vs. safety**

#### 1) Feature → **dev** = **Deploy *after* merge** (speed)

* **Goal:** fast, frequent integration. Dev should always equal **`dev@HEAD`**.
* **Why not pre-merge deploy?** Many PRs are open; pre-merge deploys would overwrite each other and Dev would no longer reflect the branch.
* **Failure model:** if the Dev deploy fails (even due to infra), fix fast or roll back; don’t block all other PRs. Dev is low-risk and shared.

#### 2) **dev** → Stage/Prod = **Merge *after* deploy** (safety)

* **Goal:** controlled promotion. Treat Stage/Prod as gates.
* **How:** deploy the **exact artifact** during the PR, run Stage/Prod checks, **merge only if green**—so the branch history contains only proven promotions.
* **Benefit:** clean audit trail and high confidence the thing that ran in Stage is exactly what you ship.

---

#### Analogy

* **Feature → dev = daily stand-up.** Quick sync: you share progress, integrate fast, and handle issues without holding up the team.
* **dev → Stage/Prod = dress rehearsal.** You perform the whole show first; only when everyone’s satisfied do you sign off and publish.

---

## Promotion PR: `stage → prod` (Production Gate)

![Alt text](2f.png)

This flow mirrors **dev → stage**: you **promote the same artifact by digest**, run checks **during the PR**, and **merge only when the environment is healthy**. The differences at Production are (1) **stricter pre-deploy gates** (approvals/change windows, prod config validation, flags default OFF) and (2) **focused post-deploy verification** (SLO guardrails + short synthetics, not Stage’s heavy suites). Below are the key deltas to highlight.

* **Open a promotion PR that references the exact Stage-proven artifact.** Include the **image** and immutable **digest**, the **commit/version**, and a link to the manifest. Production branch rules are stricter: **PRs only**, **up-to-date with base**, **manifest reachable**, **config renders cleanly**, **Stage run is green for this digest**, and **more reviewers** (e.g., TL + on-call/QA or change approver/maintenance window if required).

* **CI runs Prod preflight and promotes the same artifact—no rebuild.** The pipeline re-validates the manifest and resolves the **digest**, renders **Prod config**, checks that **feature flags default OFF**, confirms any **change window/approval**, and only then begins rollout of that **same digest**. (Production emphasizes **stricter pre-deploy gates**.)



* **Deploy with cautious rollout and instant rollback wired.** Use the **same mechanism** in Stage and Prod, but **different policy**: Stage can do a quick canary (**10% → 100%**) to rehearse the flow; Prod uses a stricter canary (**5% → 20% → 50% → 100%**) or blue-green with **SLO guardrails**, **flags OFF** during ramp, and **auto-rollback** to the last good digest. *“Flags OFF” = new code paths are **disabled by default** for users; you **gradually enable** them only after health looks good.*


* **Post-deploy verification in Prod is focused (lighter than Stage).** Don’t re-run Stage’s heavy test suites; rely on **health/SLO guardrails** (error rate, latency, saturation), short **synthetic/smoke tests** for critical journeys, and a brief **dwell period** with alerts quiet. (Stage already carried the deepest post-deploy checks.)

* **Merge only when Production is green, then record the release.** Approvals + green checks enable merge; merging **records the promotion** (Prod is already on that digest). Add a **release tag** (e.g., `v1.4.3`) annotated with **digest/commit**, and update your change log/ticket.

* **If anything fails, roll back and re-promote.** Auto-**rollback** (or **flag-off**), keep the PR **unmerged/red**, fix in `dev`, build a new digest, and repeat **dev → stage → prod**. **Key idea:** promotions **deploy during the PR**, merge **only when Prod is healthy**—with **Prod prioritizing strict pre-deploy checks** and **Stage carrying heavier post-deploy testing**.


---

## Trunk-Based CI/CD: Environment Promotions

**Overview (why / how):**

* **Single trunk** (**`main`**) is the source of truth; developers branch off it and send **PR → `main`**.
* **CI** tests the **tentative merge** on the PR; when **green**, you merge.
* A **single immutable artifact** is **built once on `main`** and then **promoted (not rebuilt)** through **Dev → Staging → Prod**.
* **Heavier checks** happen in Staging; Production ships via **CD (manual approval)** or **CDp (auto)** with guarded rollout.
* This cuts merge pain and speeds feedback; it works best with **small PRs**, strong tests, and **feature flags**.

---

### Diagram at a glance

![Alt text](2g.png)

One **source of truth (`main`)**; **Feature → `main` → Dev → Staging → Prod**. CI validates the PR’s **merge result**; the **post-merge pipeline builds once** and publishes an **image + digest** + `manifest.json`. Promotions read the manifest and **deploy by digest** to each env.

---

### Guardrails on `main` (protected branch)

* **PRs only**: no direct pushes to `main`.
* **Required status checks** gate the Merge button: **build**, **unit tests**, **lint/format**, **quick security scans**, and **“up-to-date with `main`.”**
* *(Many teams also require ≥1 review.)* Keep reviews **lightweight**—**1 CODEOWNERS approval**, **auto-merge** via **merge queue** when checks pass. Allow **trivial/docs-only** PRs to auto-merge on green CI and treat **pair/mob programming** as the approval to preserve speed.

---

### End-to-end flow (numbers match the diagram)

![Alt text](2g.png)

#### 1) Commit & push (feature branch)

Work on `feature/topup-loan`, commit locally, **push** to publish your change.

#### 2) Open PR → `main`

Raise a PR from feature → `main`. Protection rules require passing checks; opening the PR **triggers CI**.

#### 3) PR CI (Checkout • Build • Test)

CI tests a **tentative merge**: `main + feature` (**`main` unchanged**). Keep it fast (\~10–15 min):

* **Checkout & build/package** (Maven/Gradle/npm).
* **Fast tests** (unit + a couple light integration).
* **Lint/format** + **quick static scans** (SAST/SCA).

#### 4) PR status gates merge

* **Green → merge enabled.**
* **Red → merge blocked**; push a fix and CI re-runs.

#### 5) Merge PR into `main`

With green checks (and any required review), merge. `main` now points at a **known-good merge commit**.

#### 6) Post-merge on `main`: **build once & publish**

The trunk pipeline creates the **release candidate**:

* **Build/package once**, **push the image** to the registry.
* Capture the **immutable digest** (e.g., `ghcr.io/acme/loans@sha256:abc123…`).
* Publish a small **`manifest.json`** (e.g., GH Release / S3) with:
  `image`, **`digest`**, `commit_sha`, `version`, `built_at`.
  *(You can add human-friendly tags like `:v1.4.3`, `:9f3c2e7`, or `:9f3c2e7-v1.4.3` for discoverability, but **deployments should use the digest**.)*

#### 7) **Dev**: auto-deploy (CD) — **same digest, no rebuild**

A Dev job reads `manifest.json`, **verifies** the digest is pullable, **renders** Dev config (Helm/Kustomize), **deploys by digest**, and runs **smoke/health checks**.
**If red:** roll back Dev to the last good digest; fix via a new PR → repeat.

#### 8) **Stage**: promotion job / approval — **same digest**

When Dev is healthy, promote with an approval gate (often TL or QA):

* **Deploy the same digest** with Stage values/secrets (no rebuild).
* Run **heavier checks**: integration/contract, **API/UI e2e** (1–2 key flows), **perf/security smoke**.
  **If anything fails:** roll back Stage; fix on `main` via a new PR (produces a new digest in Step 6) and re-promote.

#### 9) **Prod**: approval or CDp — **same digest**

With Stage green, promote to Prod:

* **Cautious rollout**—**canary** (5% → 20% → 50% → 100%) or **blue-green**.
* **SLO guardrails** (error rate, latency, saturation) + quick synthetics; **ready rollback** to the last good digest if guardrails trip.
* Optionally **tag the release** (e.g., `v1.4.3`) and annotate with the **digest + commit**.

---

## Jenkins in this picture (why & what)

**What it is:**
Jenkins is an **open-source automation server**. It watches Git for changes and **runs pipelines** (as code) to **build, test, package, and promote** your software. It’s the glue between your **Git host, build tools, scanners, registries, and environments**.

**Why teams use it:**

* **Portable & extensible:** Runs on a laptop, VM, or Kubernetes; 1,800+ plugins for build tools, scanners, clouds, and chat.
* **Pipeline-as-Code:** A `Jenkinsfile` lives in the repo so CI/CD is versioned, reviewed, and reproducible.
* **Fits both strategies:**

  * *GitFlow:* multibranch pipelines run CI on **feature→dev** PRs, then run **promotion PR** jobs (**dev→stage→prod**) that deploy the **same digest**.
  * *Trunk-based:* CI runs on **PR→main**; a post-merge pipeline **builds once** and publishes image + `manifest.json`; downstream stages (or GitOps PRs) **promote that digest** to **Dev → Stage → Prod**.
* **Orchestration end-to-end:** One place to gate merges, sign artifacts, attach SBOMs, publish to a registry, create release notes/tags, request approvals, kick off canaries, and post status back to Git/Slack.

**How it usually runs:**

* **Controller + ephemeral agents** (Kubernetes or cloud VMs) for isolation and scale.
* **Webhooks** from your Git host trigger PR/merge builds; Jenkins posts **green/red** status checks that gate protected branches.
* **Credentials & secrets** via Jenkins Credentials (or external managers like Vault).
* **Minimal, curated plugins** + **shared libraries** for common steps (build once, write `manifest.json`, deploy by digest, open GitOps PR, etc.).

**What it isn’t:**
Not a Git server, registry, or monitoring stack. Jenkins **calls** those systems; it doesn’t replace them.

**Alternatives / complements:**
GitHub Actions or GitLab CI can do similar CI/CD. Argo CD/Flux excel at **GitOps promotions** (Jenkins can open the PRs they sync). Jenkins remains popular when you want **maximum portability and customization** across many tools and environments.

---

## Conclusion

**Key takeaways**

* **CI** runs on every push/PR to **build + test** and **gate the merge**; keep it fast.
* **CD/CDp** promote the **same immutable artifact (by digest)** across **Dev → Stage → Prod**—no rebuilds.
* **Branch protection** (PRs only, required checks, up-to-date with base, reviews) keeps mainlines deployable.
* **GitFlow vs Trunk-based:** both should *build once & promote*; they differ in how code/branches are organized and how promotions are expressed.
* **Stage vs Prod:** Stage carries **heavier post-deploy tests**; Prod enforces **stricter pre-deploy gates** and **guarded rollout** (canary/blue-green, SLO guardrails, ready rollback).
* **Jenkins** is the **orchestrator**: pipelines as code to **test, package, publish, and promote**; it posts statuses back to Git and drives the same-digest promotions.

---
