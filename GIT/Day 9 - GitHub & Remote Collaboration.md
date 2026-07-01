# Day 9 - GitHub & Remote Collaboration

## Topic

**GitHub & Remote Collaboration**

Focus areas:

- GitHub workflow
- Remote repositories
- `origin` and `upstream`
- `git fetch` vs `git pull`
- Fork vs clone
- Pull request lifecycle
- Draft pull requests
- Code review basics
- Branch protection rules
- Required reviews
- Classroom contribution workflow

---

## Learning Outcomes

By the end of Day 9, students should be able to:

- Explain what a remote Git repository is.
- Understand the difference between local and remote repositories.
- Explain `origin` and `upstream`.
- Use `git remote`, `git fetch`, `git pull`, and `git push`.
- Explain the difference between fork and clone.
- Fork a repository on GitHub.
- Clone a repository to a local machine.
- Create a feature branch and push it to GitHub.
- Open a pull request.
- Understand the pull request lifecycle.
- Explain draft pull requests and when to use them.
- Understand code review basics.
- Set up basic branch protection rules.
- Understand GitHub Actions at a high level.
- Contribute to a sample classroom repository using a proper PR workflow.

---

# 1. Theory: GitHub and Remote Collaboration

---

## 1.1 What is GitHub?

**GitHub** is a cloud-based platform used to host Git repositories.

Git is the version control tool. GitHub is a platform built around Git that helps teams collaborate.

GitHub provides:

- Remote Git repositories
- Pull requests
- Code review
- Issue tracking
- Project boards
- Branch protection
- Access control
- GitHub Actions for automation
- Release management
- Collaboration workflows

Git can be used without GitHub, but GitHub makes team collaboration much easier.

---

## 1.2 Local Repository vs Remote Repository

A **local repository** is the Git repository stored on your own computer.

A **remote repository** is the Git repository stored on a remote server such as GitHub, GitLab, or Bitbucket.

Example:

```text
Local repo:  /home/user/my-project
Remote repo: https://github.com/user/my-project.git
```

Typical workflow:

```text
Local changes -> commit -> push to remote
Remote changes -> fetch/pull -> local repo
```

---

## 1.3 What is a Remote?

A **remote** is a named connection to another Git repository.

Check remotes:

```bash
git remote -v
```

Example output:

```text
origin  https://github.com/username/my-project.git (fetch)
origin  https://github.com/username/my-project.git (push)
```

Here, `origin` is the default name for the remote repository.

---

# 2. origin and upstream

---

## 2.1 What is origin?

`origin` is the default remote name created when you clone a repository.

Example:

```bash
git clone https://github.com/username/my-project.git
```

After cloning, Git automatically creates a remote named `origin`.

Check it:

```bash
git remote -v
```

Usually:

```text
origin -> your fork or the repository you cloned
```

Common commands:

```bash
git push origin main
git pull origin main
git fetch origin
```

---

## 2.2 What is upstream?

`upstream` usually refers to the original repository from which your fork was created.

Example:

```text
Original repo: https://github.com/classroom/sample-repo
Your fork:     https://github.com/yourname/sample-repo
```

In your local repository:

```text
origin   -> your fork
upstream -> original classroom repository
```

Add upstream:

```bash
git remote add upstream https://github.com/classroom/sample-repo.git
```

Check remotes:

```bash
git remote -v
```

Example output:

```text
origin    https://github.com/yourname/sample-repo.git (fetch)
origin    https://github.com/yourname/sample-repo.git (push)
upstream  https://github.com/classroom/sample-repo.git (fetch)
upstream  https://github.com/classroom/sample-repo.git (push)
```

---

## 2.3 Why upstream is Important

When working with forks, the original repository may receive new changes.

To keep your fork updated, you fetch changes from `upstream`.

Example:

```bash
git fetch upstream
git switch main
git merge upstream/main
git push origin main
```

This updates your local `main` branch with changes from the original repository and then pushes those updates to your fork.

---

# 3. git fetch vs git pull

---

## 3.1 git fetch

`git fetch` downloads new data from a remote repository but does not automatically merge it into your current branch.

Example:

```bash
git fetch origin
```

This updates your remote-tracking branches such as:

```text
origin/main
origin/feature/login
```

After fetching, you can inspect changes before merging.

Example:

```bash
git log main..origin/main --oneline
```

or:

```bash
git diff main origin/main
```

---

## 3.2 git pull

`git pull` downloads changes from a remote repository and integrates them into your current branch.

In simple terms:

```bash
git pull = git fetch + git merge
```

Example:

```bash
git pull origin main
```

This fetches changes from `origin/main` and merges them into your current branch.

---

## 3.3 Difference Between fetch and pull

| Command             | What it Does                        | Safer For Beginners?                           |
| ------------------- | ----------------------------------- | ---------------------------------------------- |
| `git fetch`         | Downloads changes only              | Yes, because it does not modify working branch |
| `git pull`          | Downloads and merges changes        | Useful, but can create unexpected merges       |
| `git pull --rebase` | Downloads and rebases local commits | Useful for cleaner history                     |


Recommended safe workflow:

```bash
git fetch origin
git status
git log --oneline --graph --decorate --all
git merge origin/main
```

or for feature branches:

```bash
git fetch origin
git rebase origin/main
```

---

# 4. Fork vs Clone

---

## 4.1 What is Clone?

A **clone** is a local copy of a remote Git repository.

Command:

```bash
git clone https://github.com/user/repo.git
```

This downloads the repository to your computer.

Clone is done from the terminal.

After cloning, you can edit files, commit changes, and push if you have permission.

---

## 4.2 What is Fork?

A **fork** is your own copy of someone else’s repository on GitHub.

Forking happens on GitHub, usually by clicking the **Fork** button.

Example:

```text
Original repository: github.com/classroom/sample-repo
Your fork:           github.com/yourname/sample-repo
```

A fork is useful when you do not have direct write permission to the original repository.

You make changes in your fork and then open a pull request to the original repository.

---

## 4.3 Fork vs Clone

| Topic | Fork | Clone |
|---|---|---|
| Where it happens | GitHub website | Local terminal |
| What it creates | A copy under your GitHub account | A copy on your computer |
| Used for | Contributing without direct access | Working locally |
| Command needed? | No, usually done in browser | Yes, `git clone` |
| Example | Fork classroom repo | Clone your fork locally |

Typical open-source workflow:

```text
Fork original repo -> clone your fork -> create branch -> make changes -> push -> open PR
```

---

# 5. Pull Requests

---

## 5.1 What is a Pull Request?

A **pull request**, often called a **PR**, is a request to merge changes from one branch into another branch.

Usually, a PR is used to merge:

```text
feature branch -> main branch
```

or:

```text
your fork branch -> original repository main branch
```

A pull request allows team members to:

- Review code
- Discuss changes
- Request improvements
- Run automated checks
- Approve or reject changes
- Merge only after review

---

## 5.2 Pull Request Lifecycle

Typical PR lifecycle:

```text
Create branch
Make changes
Commit changes
Push branch
Open pull request
Review changes
Fix requested changes
Pass checks
Approval
Merge
Delete branch
```

---

## 5.3 Pull Request States

| State             | Meaning                                       |
| ----------------- | --------------------------------------------- |
| Open              | PR is active and waiting for review or checks |
| Draft             | PR is not ready to merge yet                  |
| Approved          | Reviewer has approved the changes             |
| Changes requested | Reviewer wants updates before merge           |
| Merged            | PR changes have been merged                   |
| Closed            | PR was closed without merging                 |

---

## 5.4 Draft Pull Requests

A **draft pull request** is used when work is not ready for final review or merge.

Use draft PRs when:

- You want early feedback
- Work is still incomplete
- Tests are not finished
- You want to show progress
- You do not want the PR merged accidentally

Example use case:

```text
Feature is 60% complete, but you want feedback on the approach.
```

When ready, convert the draft PR to a normal PR.

---

## 5.5 Good Pull Request Description

A good PR description should explain:

- What changed
- Why it changed
- How it was tested
- Any risks or known issues
- Related issue or ticket number

Example:

```markdown
## Summary

Added a healthcheck endpoint for the Node.js application.

## Changes

- Added `/health` route
- Updated README with endpoint details
- Added basic test command

## Testing

- Ran `npm start`
- Tested endpoint with `curl http://localhost:3000/health`

## Notes

No database changes required.
```

---

# 6. Code Review

---

## 6.1 What is Code Review?

**Code review** is the process where another developer checks your changes before they are merged.

The goal is not to criticize the person. The goal is to improve the code and reduce mistakes.

Code review helps catch:

- Bugs
- Security issues
- Style problems
- Missing tests
- Unclear logic
- Poor documentation
- Risky changes

---

## 6.2 What Reviewers Should Check

Reviewers should check:

- Does the code solve the problem?
- Is the code readable?
- Are there unnecessary changes?
- Are secrets or credentials committed?
- Are tests included if needed?
- Does the change break existing behavior?
- Is documentation updated?
- Are naming and formatting consistent?

---

## 6.3 How to Respond to Review Comments

Good response style:

```text
Thanks, fixed in the latest commit.
```

or:

```text
Good point. I updated the function name and added a comment.
```

If you disagree:

```text
I see your point. I kept the current approach because this function is reused in two places. Let me know if you prefer a separate helper.
```

Keep review discussions professional and focused on the code.

---

# 7. Branch Protection Rules

---

## 7.1 What are Branch Protection Rules?

**Branch protection rules** are repository settings that protect important branches such as `main`.

They prevent direct or unsafe changes to production or stable branches.

Common protection rules include:

- Require pull request before merging
- Require approvals
- Require status checks to pass
- Require conversation resolution
- Restrict who can push
- Prevent force pushes
- Prevent branch deletion
- Require signed commits

---

## 7.2 Why Protect main?

The `main` branch usually represents stable or production-ready code.

If anyone can push directly to `main`, mistakes can quickly affect the whole team.

Branch protection helps enforce a safe workflow:

```text
feature branch -> pull request -> review -> checks -> merge to main
```

---

## 7.3 Required Reviews

A common rule is:

```text
Require at least 1 approval before merging
```

This means at least one reviewer must approve the pull request before it can be merged.

This prevents unreviewed code from entering `main`.

---

## 7.4 Required Status Checks

Status checks are automated checks that must pass before merging.

Examples:

- Unit tests
- Lint checks
- Build checks
- Security scans
- GitHub Actions workflows

Example:

```text
PR can merge only if test workflow passes.
```

---

## 7.5 Example Branch Protection Rule for Class

For a classroom repository, use:

```text
Branch name pattern: main
Require a pull request before merging: enabled
Required approvals: 1
Require conversation resolution before merging: enabled
Prevent force pushes: enabled
Prevent deletion: enabled
```

Optional:

```text
Require status checks to pass before merging
```

---

# 8. GitHub Actions Introduction

---

## 8.1 What is GitHub Actions?

**GitHub Actions** is GitHub's built-in automation platform.

It can run workflows when events happen in a repository.

Examples of events:

- Push to a branch
- Pull request opened
- Pull request updated
- Issue opened
- Manual workflow trigger
- Scheduled time

GitHub Actions is commonly used for CI/CD.

---

## 8.2 What Can GitHub Actions Do?

GitHub Actions can:

- Run tests
- Check code formatting
- Build applications
- Run security scans
- Build Docker images
- Deploy applications
- Generate documentation
- Notify teams

---

## 8.3 Workflow File Location

GitHub Actions workflows are stored in:

```text
.github/workflows/
```

Example file:

```text
.github/workflows/ci.yml
```

---

## 8.4 Simple GitHub Actions Example

Example workflow for a Node.js project:

```yaml
name: Node.js CI

on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm install

      - name: Run tests
        run: npm test
```

This workflow runs when code is pushed to `main` or when a pull request targets `main`.

---

# 9. Day 9 Practical Lab

---

## Lab 1: Fork a Repository

1. Open the classroom repository on GitHub.
2. Click **Fork**.
3. Select your GitHub account.
4. Create the fork.

Result:

```text
Original repo: github.com/classroom/sample-repo
Your fork:     github.com/your-username/sample-repo
```

---

## Lab 2: Clone Your Fork

Copy your fork URL from GitHub.

Clone it:

```bash
git clone https://github.com/your-username/sample-repo.git
cd sample-repo
```

Check remotes:

```bash
git remote -v
```

Expected:

```text
origin  https://github.com/your-username/sample-repo.git (fetch)
origin  https://github.com/your-username/sample-repo.git (push)
```

---

## Lab 3: Add Upstream Remote

Add the original classroom repository as upstream:

```bash
git remote add upstream https://github.com/classroom/sample-repo.git
```

Check:

```bash
git remote -v
```

Expected:

```text
origin    https://github.com/your-username/sample-repo.git
upstream  https://github.com/classroom/sample-repo.git
```

---

## Lab 4: Sync with Upstream

```bash
git fetch upstream
git switch main
git merge upstream/main
git push origin main
```

This keeps your fork updated with the classroom repository.

---

## Lab 5: Create a Feature Branch

Create a branch:

```bash
git switch -c feature/add-student-profile
```

Make a change:

```bash
mkdir -p students
cat > students/your-name.md <<'EOF'
# Student Profile

Name: Your Name
Topic: GitHub Collaboration
Learning Goal: Practice pull request workflow
EOF
```

Check status:

```bash
git status
```

Commit:

```bash
git add students/your-name.md
git commit -m "Add student profile"
```

---

## Lab 6: Push Feature Branch to GitHub

```bash
git push origin feature/add-student-profile
```

GitHub will usually show a link to create a pull request.

---

## Lab 7: Open a Pull Request

1. Open your fork on GitHub.
2. Click **Compare & pull request**.
3. Confirm base repository is the classroom repository.
4. Confirm base branch is `main`.
5. Confirm compare branch is your feature branch.
6. Add a clear title and description.
7. Create the pull request.

Example PR title:

```text
Add student profile for Your Name
```

Example PR description:

```markdown
## Summary

Added my student profile file for the classroom GitHub collaboration lab.

## Changes

- Created `students/your-name.md`
- Added name, topic, and learning goal

## Testing

- Checked file locally
- Verified Git status before commit
```

---

## Lab 8: Update Pull Request After Review

If reviewer asks for changes:

Edit the file:

```bash
nano students/your-name.md
```

Commit update:

```bash
git add students/your-name.md
git commit -m "Update student profile details"
```

Push again:

```bash
git push origin feature/add-student-profile
```

The pull request updates automatically.

---

## Lab 9: Set Up Branch Protection on main

Repository owner/admin should do this part.

Steps:

1. Open repository on GitHub.
2. Go to **Settings**.
3. Go to **Branches** or **Rulesets**.
4. Create a rule for `main`.
5. Enable requiring a pull request before merging.
6. Require at least **1 approval**.
7. Enable conversation resolution if available.
8. Disable force pushes.
9. Save the rule.

Recommended class rule:

```text
Protect branch: main
Require PR before merge: Yes
Required approvals: 1
Allow force push: No
Allow deletion: No
```

---

# 10. Assignment 9

---

## Assignment: Contribute to a Sample Classroom Repo Using PR Workflow

### Objective

Students must contribute to a sample classroom repository using the proper GitHub pull request workflow.

---

## Assignment Requirements

Each student must:

1. Fork the classroom repository.
2. Clone their fork locally.
3. Add the classroom repository as `upstream`.
4. Create a feature branch.
5. Add or edit a file.
6. Commit with a meaningful message.
7. Push the feature branch to their fork.
8. Open a pull request to the classroom repository.
9. Respond to review feedback if requested.
10. Merge only after approval, if they have permission.

---

## Suggested Contribution

Create a student profile file:

```text
students/your-name.md
```

Example content:

```markdown
# Student Profile

Name: Your Name
GitHub Username: your-username
Favorite Linux Command: grep
Learning Goal: Become confident with Git and GitHub collaboration.
```

---

## Required Commands

```bash
git clone https://github.com/your-username/sample-repo.git
cd sample-repo

git remote add upstream https://github.com/classroom/sample-repo.git
git fetch upstream

git switch main
git merge upstream/main

git switch -c feature/add-your-profile

mkdir -p students
nano students/your-name.md

git status
git add students/your-name.md
git commit -m "Add student profile for Your Name"
git push origin feature/add-your-profile
```

Then open a pull request on GitHub.

---

## Assignment Submission Format

Students should submit:

```text
GitHub username:
Fork URL:
Branch name:
Pull request URL:
Commit message:
Reviewer name:
PR status:
Screenshot or terminal output:
```

---

# 11. Quick Command Summary

## Remote Commands

```bash
git remote -v
git remote add upstream <repo-url>
git fetch origin
git fetch upstream
git pull origin main
git push origin main
```

## Branch and PR Workflow

```bash
git switch main
git pull origin main
git switch -c feature/my-change
git add .
git commit -m "Meaningful message"
git push origin feature/my-change
```

## Sync Fork with Upstream

```bash
git fetch upstream
git switch main
git merge upstream/main
git push origin main
```

---

# 12. Common Beginner Mistakes

- Cloning the original repository instead of the fork.
- Forgetting to create a feature branch.
- Committing directly to `main`.
- Forgetting to add `upstream`.
- Opening PR from the wrong branch.
- Opening PR to the wrong base repository.
- Using unclear commit messages.
- Not pulling latest changes before starting work.
- Ignoring reviewer comments.
- Force-pushing without understanding the impact.
- Merging a PR without required approval.
- Forgetting that `git fetch` does not update the current branch automatically.

---

# 13. End-of-Day Review Questions

1. What is GitHub?
2. What is a remote repository?
3. What is `origin`?
4. What is `upstream`?
5. What is the difference between `git fetch` and `git pull`?
6. What is the difference between fork and clone?
7. Why do open-source contributors usually fork first?
8. What is a pull request?
9. What is a draft pull request?
10. What should be included in a good PR description?
11. What is code review?
12. Why should `main` be protected?
13. What does requiring one review mean?
14. What are GitHub Actions used for?
15. What is the correct workflow for contributing to a classroom repository?

---

# 14. Expected Practical Output

By the end of the lab, students should have:

- Forked a classroom repository.
- Cloned their fork locally.
- Added an `upstream` remote.
- Created a feature branch.
- Made a meaningful commit.
- Pushed the branch to GitHub.
- Opened a pull request.
- Understood how reviews and branch protection work.
- Understood GitHub Actions at a basic awareness level.
