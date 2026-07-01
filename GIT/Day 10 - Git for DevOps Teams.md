# Day 10 - Git for DevOps Teams

## Topic

**Git for DevOps Teams**

Focus areas:

- Git hooks
- Pre-commit, pre-push, and commit-msg hooks
- Tagging for releases
- Semantic versioning
- Annotated vs lightweight Git tags
- GitHub repository best practices
- `CODEOWNERS`
- README best practices
- Monorepo vs multirepo considerations
- Branch protection for team workflows

---

## Learning Outcomes

By the end of Day 10, students should be able to:

- Explain why Git is important for DevOps teams.
- Understand how Git hooks help automate quality checks.
- Create a `pre-commit` hook to run linting before commits.
- Understand semantic versioning using `MAJOR.MINOR.PATCH`.
- Create lightweight and annotated Git tags.
- Tag a release such as `v1.0.0`.
- Push tags to GitHub.
- Understand the purpose of `CODEOWNERS`.
- Write a clean and useful `README.md`.
- Compare monorepo and multirepo approaches.
- Set up a Node.js repository with `.gitignore`, `README.md`, `CODEOWNERS`, and branch protection.

---

# 1. Theory: Git for DevOps Teams

---

## 1.1 Why Git Matters in DevOps

Git is not only a tool for developers. In DevOps, Git is also used to manage:

- Application source code
- Infrastructure as Code
- CI/CD pipeline files
- Kubernetes manifests
- Dockerfiles
- Helm charts
- Terraform modules
- Ansible playbooks
- Shell scripts
- Release tags
- Documentation

DevOps teams use Git as a central source of truth.

A common DevOps principle is:

```text
Everything important should be version controlled.
```

This means changes should be visible, reviewable, traceable, and reversible.

---

## 1.2 GitOps Mindset

In GitOps-style workflows, Git becomes the source of truth for infrastructure and deployment configuration.

Example:

```text
Git commit -> CI/CD pipeline -> deployment
```

Benefits:

- Clear change history
- Easy rollback
- Review before deployment
- Audit trail
- Consistent environments
- Better collaboration between developers and operations teams

---

# 2. Git Hooks

---

## 2.1 What are Git Hooks?

**Git hooks** are scripts that Git runs automatically when specific Git events happen.

For example, Git can run a script:

- Before a commit
- Before a push
- After a merge
- When writing a commit message

Hooks are useful for enforcing team standards and preventing bad changes from entering the repository.

---

## 2.2 Where Git Hooks are Stored

Git hooks are stored inside:

```text
.git/hooks/
```

View hooks:

```bash
ls -la .git/hooks
```

When a repository is initialized, Git usually creates sample hook files such as:

```text
pre-commit.sample
pre-push.sample
commit-msg.sample
```

To activate a hook, remove `.sample` and make the file executable.

Example:

```bash
mv .git/hooks/pre-commit.sample .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
```

---

## 2.3 Important Git Hooks

| Hook         | When it Runs                                             | Common Use                             |
| ------------ | -------------------------------------------------------- | -------------------------------------- |
| `pre-commit` | Before a commit is created                               | Run linting, formatting, tests         |
| `pre-push`   | Before code is pushed                                    | Run tests, security checks             |
| `commit-msg` | After commit message is written, before commit completes | Validate commit message format         |
| `post-merge` | After merge completes                                    | Install dependencies or show reminders |
| `pre-rebase` | Before rebase starts                                     | Prevent unsafe rebase operations       |
|              |                                                          |                                        |

---

## 2.4 pre-commit Hook

A `pre-commit` hook runs before Git creates a commit.

Common uses:

- Run linting
- Run formatting checks
- Prevent secrets from being committed
- Prevent large files from being committed
- Run quick tests
- Validate file names

Example workflow:

```text
git commit -> pre-commit hook runs -> if checks pass, commit succeeds
```

If the hook exits with a non-zero exit code, the commit is stopped.

---

## 2.5 pre-push Hook

A `pre-push` hook runs before Git pushes commits to a remote repository.

Common uses:

- Run full test suite
- Run build check
- Run security scan
- Prevent pushing directly to protected branches
- Validate branch naming

Example:

```text
git push -> pre-push hook runs -> if checks pass, push succeeds
```

---

## 2.6 commit-msg Hook

A `commit-msg` hook checks the commit message before the commit is finalized.

It is commonly used to enforce commit message standards.

Example valid commit messages:

```text
feat: add login endpoint
fix: correct healthcheck status code
docs: update README setup steps
chore: update dependencies
```

Example invalid commit messages:

```text
update
final
fix
changes
```

---

## 2.7 Local Hooks vs Shared Hooks

Hooks inside `.git/hooks/` are local to each developer's machine. They are not automatically committed to the repository.

For team-wide hook management, teams often use tools such as:

- Husky for Node.js projects
- pre-commit framework
- Lefthook
- CI/CD checks in GitHub Actions

For beginners, it is still useful to understand raw Git hooks first.

---

# 3. Semantic Versioning

---

## 3.1 What is Semantic Versioning?

**Semantic Versioning**, often called **SemVer**, is a version numbering system.

Format:

```text
MAJOR.MINOR.PATCH
```

Example:

```text
1.4.2
```

Meaning:

| Part | Example | Meaning |
|---|---|---|
| `MAJOR` | `1` | Breaking changes |
| `MINOR` | `4` | New features without breaking existing behavior |
| `PATCH` | `2` | Bug fixes without new features or breaking changes |

---

## 3.2 Version Example

Version:

```text
2.5.9
```

Means:

```text
MAJOR = 2
MINOR = 5
PATCH = 9
```

If you fix a bug:

```text
2.5.9 -> 2.5.10
```

If you add a new backward-compatible feature:

```text
2.5.9 -> 2.6.0
```

If you introduce breaking changes:

```text
2.5.9 -> 3.0.0
```

---

## 3.3 When to Increase MAJOR, MINOR, PATCH

### PATCH

Increase PATCH for bug fixes.

Example:

```text
1.0.0 -> 1.0.1
```

Use when:

- Fixing a bug
- Updating documentation typo
- Fixing small behavior without breaking compatibility

### MINOR

Increase MINOR for new backward-compatible features.

Example:

```text
1.0.1 -> 1.1.0
```

Use when:

- Adding a new endpoint
- Adding a new optional config
- Adding a new feature that does not break existing users

### MAJOR

Increase MAJOR for breaking changes.

Example:

```text
1.1.0 -> 2.0.0
```

Use when:

- Removing an API
- Changing required config
- Changing behavior in a way that breaks existing users
- Changing database schema in a non-backward-compatible way

---

# 4. Git Tags for Releases

---

## 4.1 What is a Git Tag?

A **Git tag** is a named pointer to a specific commit.

Tags are commonly used to mark releases.

Example:

```text
v1.0.0
v1.1.0
v2.0.0
```

A tag usually points to a commit that represents a stable release.

---

## 4.2 Why Tags are Used in DevOps

Tags are useful because they provide stable release references.

DevOps teams use tags to:

- Mark production releases
- Trigger CI/CD deployment pipelines
- Build Docker images with release versions
- Roll back to previous versions
- Create GitHub releases
- Track what version is running in production

Example Docker image tag:

```text
myapp:v1.0.0
```

---

## 4.3 Lightweight Tags

A lightweight tag is a simple pointer to a commit.

Create a lightweight tag:

```bash
git tag v1.0.0
```

View tags:

```bash
git tag
```

Push tag:

```bash
git push origin v1.0.0
```

Lightweight tags are simple but contain less metadata.

---

## 4.4 Annotated Tags

An annotated tag stores extra metadata such as:

- Tagger name
- Tagger email
- Date
- Message
- Optional GPG signature

Create annotated tag:

```bash
git tag -a v1.0.0 -m "Release version 1.0.0"
```

Show tag details:

```bash
git show v1.0.0
```

Push tag:

```bash
git push origin v1.0.0
```

Annotated tags are recommended for official releases.

---

## 4.5 Annotated vs Lightweight Tags

| Topic | Lightweight Tag | Annotated Tag |
|---|---|---|
| Metadata | Minimal | Stores tagger, date, message |
| Use case | Temporary/local marker | Official release |
| Command | `git tag v1.0.0` | `git tag -a v1.0.0 -m "message"` |
| Recommended for releases | No | Yes |

---

## 4.6 Push All Tags

Push all local tags:

```bash
git push origin --tags
```

Use carefully, because this pushes every local tag.

---

## 4.7 Delete a Tag

Delete local tag:

```bash
git tag -d v1.0.0
```

Delete remote tag:

```bash
git push origin --delete v1.0.0
```

---

# 5. GitHub Repository Best Practices

---

## 5.1 Good Repository Structure

A clean repository is easier to understand, review, and maintain.

Example Node.js repository structure:

```text
my-node-app/
├── .github/
│   └── workflows/
│       └── ci.yml
├── src/
│   └── index.js
├── tests/
│   └── app.test.js
├── .gitignore
├── CODEOWNERS
├── README.md
├── package.json
└── package-lock.json
```

---

## 5.2 README Best Practices

A `README.md` is usually the first file people read.

A good README should include:

- Project name
- Project purpose
- Requirements
- Installation steps
- How to run locally
- How to test
- Environment variables
- Deployment notes
- Contribution rules
- License information, if applicable

---

## 5.3 Example README Structure

```markdown
# Project Name

## Overview

Short description of what the project does.

## Requirements

- Node.js 20+
- npm

## Installation

```bash
npm install
```

## Run Locally

```bash
npm start
```

## Test

```bash
npm test
```

## Environment Variables

```text
PORT=3000
NODE_ENV=development
```

## Deployment

Explain deployment steps or link to deployment documentation.

## Contributing

Use feature branches and open a pull request into `main`.
```

---

## 5.4 .gitignore Best Practices

A `.gitignore` file prevents unnecessary or sensitive files from being committed.

Example for Node.js:

```gitignore
node_modules/
.env
.env.local
npm-debug.log*
yarn-debug.log*
yarn-error.log*
coverage/
dist/
build/
.DS_Store
```

Never commit:

- `node_modules/`
- `.env`
- Secret keys
- Build output
- Logs
- Temporary files

---

# 6. CODEOWNERS

---

## 6.1 What is CODEOWNERS?

`CODEOWNERS` is a GitHub file that defines who owns specific files or directories in a repository.

When a pull request changes owned files, GitHub can automatically request reviews from the correct people or teams.

---

## 6.2 CODEOWNERS File Location

The `CODEOWNERS` file can be placed in:

```text
.github/CODEOWNERS
CODEOWNERS
docs/CODEOWNERS
```

A common location is:

```text
.github/CODEOWNERS
```

---

## 6.3 CODEOWNERS Syntax

Example:

```text
# Global owners
* @devops-team

# Application code owners
/src/ @backend-team

# Documentation owners
/docs/ @docs-team

# GitHub Actions owners
.github/workflows/ @devops-team

# Package files
package.json @nodejs-maintainers
```

---

## 6.4 Classroom CODEOWNERS Example

For a simple classroom repository:

```text
# Default owner for all files
* @teacher-github-username

# Student profile files
/students/ @teacher-github-username
```

For a company repository:

```text
# DevOps owns CI/CD and infrastructure
.github/workflows/ @company/devops
Dockerfile @company/devops
terraform/ @company/platform

# Backend team owns source code
src/ @company/backend
```

---

## 6.5 Benefits of CODEOWNERS

Benefits:

- Automatic reviewer assignment
- Clear ownership
- Faster review routing
- Safer changes to critical files
- Better accountability

Useful for:

- Infrastructure repositories
- Monorepos
- Production services
- Security-sensitive files
- CI/CD configuration

---

# 7. Monorepo vs Multirepo

---

## 7.1 What is a Monorepo?

A **monorepo** is a single repository that contains multiple projects or services.

Example:

```text
company-platform/
├── services/
│   ├── auth-service/
│   ├── payment-service/
│   └── notification-service/
├── infrastructure/
├── shared-libraries/
└── docs/
```

---

## 7.2 Advantages of Monorepo

Advantages:

- One place for all code
- Easier cross-project changes
- Shared tooling
- Consistent standards
- Easier dependency visibility
- Centralized CI/CD patterns

---

## 7.3 Disadvantages of Monorepo

Disadvantages:

- Repository can become very large
- CI/CD can become slower if not optimized
- Access control can be harder
- More complex ownership rules
- Requires good structure and automation

---

## 7.4 What is Multirepo?

A **multirepo** approach uses separate repositories for separate projects or services.

Example:

```text
auth-service
payment-service
notification-service
terraform-infra
frontend-app
```

Each service has its own repository.

---

## 7.5 Advantages of Multirepo

Advantages:

- Clear separation of services
- Smaller repositories
- Easier access control
- Independent CI/CD pipelines
- Teams can work independently
- Simpler ownership per repository

---

## 7.6 Disadvantages of Multirepo

Disadvantages:

- Cross-service changes are harder
- More repositories to manage
- Duplicate CI/CD configuration
- Dependency updates can be scattered
- Harder to maintain consistent standards

---

## 7.7 Monorepo vs Multirepo Comparison

| Topic | Monorepo | Multirepo |
|---|---|---|
| Repository count | One large repository | Many smaller repositories |
| Ownership | Needs strong CODEOWNERS | Easier per repo |
| CI/CD | Needs optimization | Independent pipelines |
| Cross-project changes | Easier | Harder |
| Access control | More complex | Easier |
| Best for | Shared platform, tightly connected services | Independent services and teams |

---

# 8. Branch Protection for DevOps Teams

---

## 8.1 Why Branch Protection Matters

DevOps teams often use the `main` branch as the source for production deployments.

If anyone can push directly to `main`, a mistake can trigger a bad deployment.

Branch protection helps enforce safe changes.

---

## 8.2 Recommended Branch Protection Rules

Recommended rules for `main`:

```text
Require a pull request before merging
Require at least 1 approval
Require status checks to pass
Require conversation resolution
Prevent force pushes
Prevent branch deletion
Require branches to be up to date before merging
```

---

## 8.3 Example DevOps Protection Policy

Example:

```text
Branch: main

Rules:
- No direct pushes
- Pull request required
- Minimum 1 approval
- CI must pass
- CODEOWNERS review required
- Force push disabled
- Branch deletion disabled
```

This is a strong baseline for production repositories.

---

# 9. Day 10 Practical Lab

---

## Lab 1: Create a Node.js Repository

Create project:

```bash
mkdir day10-devops-git
cd day10-devops-git
git init
git branch -M main
npm init -y
```

Create app file:

```bash
mkdir src
cat > src/index.js <<'EOF'
function main() {
  console.log("Day 10 Git for DevOps Teams");
}

main();
EOF
```

---

## Lab 2: Add .gitignore

Create `.gitignore`:

```bash
cat > .gitignore <<'EOF'
node_modules/
.env
.env.local
npm-debug.log*
coverage/
dist/
build/
.DS_Store
EOF
```

---

## Lab 3: Add README

Create `README.md`:

```bash
cat > README.md <<'EOF'
# Day 10 DevOps Git Project

## Overview

This is a sample Node.js repository for practicing Git hooks, release tags, CODEOWNERS, and GitHub repository best practices.

## Requirements

- Node.js 20+
- npm

## Installation

```bash
npm install
```

## Run

```bash
node src/index.js
```

## Git Workflow

- Create a feature branch.
- Commit meaningful changes.
- Open a pull request into `main`.
- Merge only after review.

## Release

Releases are tagged using semantic versioning, for example `v1.0.0`.
EOF
```

---

## Lab 4: Add CODEOWNERS

Create GitHub directory:

```bash
mkdir -p .github
```

Create `CODEOWNERS`:

```bash
cat > .github/CODEOWNERS <<'EOF'
# Default owner for all files
* @your-github-username

# GitHub Actions and repository config
.github/ @your-github-username

# Source code
src/ @your-github-username

# Package files
package.json @your-github-username
EOF
```

Replace `@your-github-username` with your real GitHub username or team.

---

## Lab 5: Add a Simple Lint Script

Install ESLint:

```bash
npm install --save-dev eslint
```

Initialize ESLint if required:

```bash
npx eslint --init
```

For a simple classroom demo, add this script in `package.json`:

```json
"scripts": {
  "start": "node src/index.js",
  "lint": "node --check src/index.js"
}
```

Test lint:

```bash
npm run lint
```

The command `node --check src/index.js` checks JavaScript syntax without running the file.

---

## Lab 6: Write a pre-commit Hook to Run Linting

Create `pre-commit` hook:

```bash
cat > .git/hooks/pre-commit <<'EOF'
#!/bin/bash

echo "Running pre-commit checks..."

npm run lint

if [ $? -ne 0 ]; then
  echo "Lint failed. Commit blocked."
  exit 1
fi

echo "Pre-commit checks passed."
exit 0
EOF
```

Make it executable:

```bash
chmod +x .git/hooks/pre-commit
```

Test by committing:

```bash
git add .
git commit -m "Set up Node.js repo with Git best practices"
```

If lint passes, the commit succeeds.

If lint fails, the commit is blocked.

---

## Lab 7: Write a commit-msg Hook

Create a simple commit message validator:

```bash
cat > .git/hooks/commit-msg <<'EOF'
#!/bin/bash

COMMIT_MSG_FILE="$1"
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

if [[ ! "$COMMIT_MSG" =~ ^(feat|fix|docs|chore|test|refactor): ]]; then
  echo "Invalid commit message."
  echo "Use format: type: message"
  echo "Allowed types: feat, fix, docs, chore, test, refactor"
  exit 1
fi

exit 0
EOF
```

Make executable:

```bash
chmod +x .git/hooks/commit-msg
```

Test invalid message:

```bash
git commit --allow-empty -m "update"
```

Expected: commit should fail.

Test valid message:

```bash
git commit --allow-empty -m "chore: test commit message hook"
```

Expected: commit should pass.

---

## Lab 8: Tag a Release v1.0.0

Create an annotated release tag:

```bash
git tag -a v1.0.0 -m "Release version 1.0.0"
```

View tags:

```bash
git tag
```

Show tag details:

```bash
git show v1.0.0
```

---

## Lab 9: Push Repository and Tag to GitHub

Create a GitHub repository first, then connect local repo:

```bash
git remote add origin https://github.com/your-username/day10-devops-git.git
git push -u origin main
```

Push tag:

```bash
git push origin v1.0.0
```

Or push all tags:

```bash
git push origin --tags
```

---

## Lab 10: Set Branch Protection on main

Repository owner/admin should complete this part on GitHub.

Steps:

1. Open the repository on GitHub.
2. Go to **Settings**.
3. Open **Branches** or **Rulesets**.
4. Create a rule for `main`.
5. Require pull request before merging.
6. Require at least 1 approval.
7. Require status checks if GitHub Actions is configured.
8. Require conversation resolution.
9. Disable force pushes.
10. Disable branch deletion.
11. Save the rule.

Recommended rule:

```text
Branch: main
Require PR: enabled
Required approvals: 1
Require CODEOWNERS review: enabled if CODEOWNERS exists
Force push: disabled
Deletion: disabled
```

---

# 10. Assignment 10

---

## Assignment: Set Up a Node.js Repo with DevOps Git Best Practices

### Objective

Create a Node.js repository configured with `.gitignore`, `README.md`, `CODEOWNERS`, Git hooks, release tag, and branch protection.

---

## Requirements

Your repository must include:

1. Node.js project initialized with `npm init -y`.
2. `.gitignore` for Node.js.
3. `README.md` with setup and run instructions.
4. `.github/CODEOWNERS`.
5. `pre-commit` hook to run linting.
6. Release tag `v1.0.0`.
7. GitHub remote repository.
8. Branch protection on `main`.
9. At least one pull request merged after review.

---

## Suggested Commands

```bash
mkdir day10-node-devops-repo
cd day10-node-devops-repo
git init
git branch -M main
npm init -y

mkdir src .github

cat > src/index.js <<'EOF'
console.log("Hello from Day 10 DevOps Git repo");
EOF

cat > .gitignore <<'EOF'
node_modules/
.env
.env.local
npm-debug.log*
coverage/
dist/
build/
.DS_Store
EOF

cat > README.md <<'EOF'
# Day 10 Node DevOps Repo

## Overview

This repository demonstrates Git best practices for DevOps teams.

## Requirements

- Node.js 20+
- npm

## Install

```bash
npm install
```

## Run

```bash
node src/index.js
```

## Workflow

Use feature branches and pull requests before merging into main.
EOF

cat > .github/CODEOWNERS <<'EOF'
* @your-github-username
.github/ @your-github-username
src/ @your-github-username
package.json @your-github-username
EOF
```

Add lint script in `package.json`:

```json
"scripts": {
  "start": "node src/index.js",
  "lint": "node --check src/index.js"
}
```

Create pre-commit hook:

```bash
cat > .git/hooks/pre-commit <<'EOF'
#!/bin/bash

echo "Running lint before commit..."
npm run lint

if [ $? -ne 0 ]; then
  echo "Lint failed. Commit blocked."
  exit 1
fi

echo "Lint passed."
exit 0
EOF

chmod +x .git/hooks/pre-commit
```

Commit:

```bash
git add .
git commit -m "chore: set up Node.js DevOps repository"
```

Create tag:

```bash
git tag -a v1.0.0 -m "Release version 1.0.0"
```

Push:

```bash
git remote add origin https://github.com/your-username/day10-node-devops-repo.git
git push -u origin main
git push origin v1.0.0
```

---

## Assignment Submission Format

Students should submit:

```text
Repository URL:
Branch protection screenshot:
CODEOWNERS file content:
README file content:
Git tag output:
Pre-commit hook test output:
Pull request URL:
Reviewer name:
Final status:
```

---

# 11. Quick Command Summary

## Hooks

```bash
ls -la .git/hooks
nano .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
```

## Tags

```bash
git tag
git tag v1.0.0
git tag -a v1.0.0 -m "Release version 1.0.0"
git show v1.0.0
git push origin v1.0.0
git push origin --tags
```

## CODEOWNERS

```bash
mkdir -p .github
nano .github/CODEOWNERS
```

## GitHub Remote

```bash
git remote -v
git remote add origin <repo-url>
git push -u origin main
```

---

# 12. Common Beginner Mistakes

- Forgetting to make Git hooks executable.
- Expecting `.git/hooks` to be automatically shared with other developers.
- Using lightweight tags for official releases when annotated tags are better.
- Forgetting to push tags to GitHub.
- Tagging the wrong commit.
- Using unclear version numbers.
- Committing `node_modules/`.
- Forgetting to replace `@your-github-username` in `CODEOWNERS`.
- Writing a README with no setup instructions.
- Not protecting the `main` branch.
- Allowing direct pushes to production branches.
- Forgetting that branch protection is configured on GitHub, not local Git.

---

# 13. End-of-Day Review Questions

1. What is a Git hook?
2. Where are local Git hooks stored?
3. What does a `pre-commit` hook do?
4. What does a `pre-push` hook do?
5. What does a `commit-msg` hook do?
6. What does semantic versioning mean?
7. What type of change increases `MAJOR`?
8. What type of change increases `MINOR`?
9. What type of change increases `PATCH`?
10. What is the difference between annotated and lightweight tags?
11. Why are tags useful in DevOps release workflows?
12. What is `CODEOWNERS` used for?
13. What should a good README include?
14. What is the difference between monorepo and multirepo?
15. Why should `main` be protected?

---

# 14. Expected Practical Output

By the end of the lab, students should have:

- Created a Node.js Git repository.
- Added `.gitignore`.
- Added `README.md`.
- Added `.github/CODEOWNERS`.
- Created a working `pre-commit` hook.
- Created a release tag `v1.0.0`.
- Pushed the repository and tag to GitHub.
- Understood branch protection rules.
- Completed the assignment using a PR-based workflow.
