# Day 8 - Git Branching & Merging

## Topic

**Git Branching & Merging**

Focus areas:

- Branch strategies
- Branches and `HEAD` as pointers
- Fast-forward merge vs 3-way merge
- Merge vs rebase
- Interactive rebase for clean history
- Merge conflict markers
- Conflict resolution strategies
- Git flow vs trunk-based development
- Team workflow simulation

---

## Learning Outcomes

By the end of Day 8, students should be able to:

- Explain how Git branches work internally.
- Understand `HEAD` and branches as pointers.
- Create and switch branches.
- Merge a feature branch back into `main`.
- Explain the difference between fast-forward and 3-way merges.
- Understand when to use merge and when to use rebase.
- Use interactive rebase to clean commit history.
- Identify Git merge conflict markers.
- Resolve merge conflicts safely.
- Compare Git flow and trunk-based development.
- Simulate a team workflow with multiple branches modifying the same file.

---

# 1. Theory: Git Branching Fundamentals

---

## 1.1 What is a Git Branch?

A **branch** is an independent line of development.

Branches allow developers to work on new features, bug fixes, experiments, or releases without directly changing the main working version of the project.

Example branch names:

```text
main
feature/login
bugfix/navbar-error
hotfix/payment-timeout
release/v1.0.0
```

The default branch is commonly called:

```text
main
```

In older repositories, it may be called:

```text
master
```

---

## 1.2 Why Branches Are Important

Branches are important because they allow teams to work safely.

Without branches, everyone would directly change the same main codebase, which increases the risk of breaking the project.

Branches help with:

- Feature development
- Bug fixing
- Code review
- Testing changes before release
- Experimenting safely
- Separating stable code from work in progress

Typical workflow:

```text
main -> create feature branch -> make changes -> test -> merge back to main
```

---

## 1.3 Branches as Pointers

In Git, a branch is not a full copy of the project.

A branch is simply a lightweight pointer to a commit.

Example:

```text
A---B---C  main
```

Here, `main` points to commit `C`.

When you create a new branch:

```bash
git branch feature-login
```

Git creates a new pointer at the current commit:

```text
A---B---C  main
                   feature-login
```

Both `main` and `feature-login` initially point to the same commit.

When you commit on the feature branch:

```text
A---B---C  main
         \
          D---E  feature-login
```

Now the feature branch has moved forward, but `main` remains at commit `C`.

---

## 1.4 HEAD

`HEAD` is a special pointer that tells Git where you currently are.

Usually, `HEAD` points to the current branch.

Check `HEAD`:

```bash
cat .git/HEAD
```

Example output:

```text
ref: refs/heads/main
```

This means:

- You are currently on the `main` branch.
- `HEAD` points to `main`.
- `main` points to the latest commit on that branch.

When you switch branches:

```bash
git switch feature-login
```

`HEAD` points to `feature-login`.

---

## 1.5 Detached HEAD

A **detached HEAD** happens when `HEAD` points directly to a commit instead of a branch.

Example:

```bash
git checkout <commit-hash>
```

In this state, you are not on a branch.

This is useful for inspecting old commits, but new commits made in detached HEAD can be lost if not saved to a branch.

Create a branch from detached HEAD:

```bash
git switch -c recovered-work
```

---

# 2. Branch Commands

---

## 2.1 Check Current Branch

```bash
git branch
```

Show all branches:

```bash
git branch -a
```

Show branch with last commit:

```bash
git branch -v
```

---

## 2.2 Create a Branch

```bash
git branch feature-login
```

This creates a branch but does not switch to it.

Create and switch in one command:

```bash
git switch -c feature-login
```

Older syntax:

```bash
git checkout -b feature-login
```

---

## 2.3 Switch Branches

```bash
git switch main
git switch feature-login
```

Older syntax:

```bash
git checkout main
```

---

## 2.4 Delete a Branch

Delete a merged branch:

```bash
git branch -d feature-login
```

Force delete an unmerged branch:

```bash
git branch -D feature-login
```

Use `-D` carefully because it can delete work that has not been merged.

---

# 3. Merge Concepts

---

## 3.1 What is a Merge?

A **merge** combines changes from one branch into another branch.

Usually, you merge a feature branch into `main`.

Example:

```bash
git switch main
git merge feature-login
```

This brings the changes from `feature-login` into `main`.

---

## 3.2 Fast-Forward Merge

A **fast-forward merge** happens when the target branch has not changed since the feature branch was created.

Example:

```text
Before merge:

A---B---C  main
         \
          D---E  feature-login
```

If `main` has not moved forward, Git can simply move the `main` pointer forward:

```text
After fast-forward merge:

A---B---C---D---E  main, feature-login
```

No new merge commit is created.

Command:

```bash
git switch main
git merge feature-login
```

Force no fast-forward and create a merge commit:

```bash
git merge --no-ff feature-login
```

---

## 3.3 3-Way Merge

A **3-way merge** happens when both branches have new commits after they split.

Example:

```text
A---B---C---F  main
         \
          D---E  feature-login
```

Here:

- `main` has commit `F`
- `feature-login` has commits `D` and `E`
- Both branches changed after commit `C`

Git uses three points:

1. Common ancestor commit
2. Latest commit on current branch
3. Latest commit on branch being merged

Result:

```text
A---B---C---F---M  main
         \     /
          D---E
```

`M` is a merge commit.

---

## 3.4 Merge Commit

A merge commit is a commit with more than one parent.

It records that two branches were combined.

Merge commits are useful because they preserve the true branch history.

However, too many merge commits can make history harder to read.

---

# 4. Rebase Concepts

---

## 4.1 What is Rebase?

`git rebase` moves commits from one base commit to another base commit.

It rewrites commit history to make it look as if your branch was created from the latest version of another branch.

Example before rebase:

```text
A---B---C---F  main
         \
          D---E  feature-login
```

After rebasing `feature-login` onto `main`:

```text
A---B---C---F  main
             \
              D'---E'  feature-login
```

The commits `D` and `E` are recreated as new commits `D'` and `E'`.

---

## 4.2 Why Use Rebase?

Rebase is used to create a cleaner, linear history.

Benefits:

- Avoids unnecessary merge commits
- Makes history easier to read
- Makes feature branch look updated with latest `main`
- Useful before opening or updating a pull request

Common command:

```bash
git switch feature-login
git rebase main
```

---

## 4.3 Merge vs Rebase

| Feature                    | Merge                        | Rebase                         |
| -------------------------- | ---------------------------- | ------------------------------ |
| Preserves original history | Yes                          | No, rewrites commits           |
| Creates merge commit       | Sometimes                    | No                             |
| History style              | Branching history            | Linear history                 |
| Safer for shared branches  | Yes                          | Be careful                     |
| Best used for              | Combining completed branches | Cleaning local feature history |

---

## 4.4 Golden Rule of Rebase

Do not rebase public/shared branches that other people are using.

Safe to rebase:

```text
Your local feature branch
```

Avoid rebasing:

```text
main
shared team branches
branches already pushed and used by others
```

Reason: rebase rewrites commit hashes. This can confuse collaborators and create duplicate commits.

---

## 4.5 Interactive Rebase

Interactive rebase lets you clean up commit history.

Common use cases:

- Rename commit messages
- Squash multiple commits into one
- Reorder commits
- Delete unnecessary commits
- Edit old commits

Start interactive rebase for last 3 commits:

```bash
git rebase -i HEAD~3
```

Git opens an editor with something like:

```text
pick a1b2c3d Add app file
pick b2c3d4e Fix typo
pick c3d4e5f Update README
```

Common actions:

| Action   | Meaning                                          |
| -------- | ------------------------------------------------ |
| `pick`   | Use commit as-is                                 |
| `reword` | Change commit message                            |
| `edit`   | Stop and edit commit                             |
| `squash` | Combine commit with previous commit              |
| `fixup`  | Combine with previous commit and discard message |
| `drop`   | Remove commit                                    |

Example:

```text
pick a1b2c3d Add app file
squash b2c3d4e Fix typo
pick c3d4e5f Update README
```

This combines `Fix typo` into `Add app file`.

---

# 5. Merge Conflicts

---

## 5.1 What is a Merge Conflict?

A **merge conflict** happens when Git cannot automatically combine changes.

This usually occurs when two branches modify the same line of the same file.

Example:

- Branch `main` changes line 5 in `README.md`
- Branch `feature` also changes line 5 in `README.md`

Git does not know which version should be kept, so it asks the user to resolve the conflict manually.

---

## 5.2 Conflict Markers

When a conflict happens, Git adds conflict markers inside the file.

Example:

```text
<<<<<<< HEAD
This is the version from current branch.
=======
This is the version from the branch being merged.
>>>>>>> feature-branch
```

Meaning:

| Marker | Meaning |
|---|---|
| `<<<<<<< HEAD` | Start of current branch version |
| `=======` | Separator between versions |
| `>>>>>>> branch-name` | End of incoming branch version |

You must edit the file and remove these markers.

---

## 5.3 Conflict Resolution Steps

Basic conflict resolution workflow:

```bash
git status
nano conflicted-file.txt
git add conflicted-file.txt
git commit
```

Steps:

1. Check which files are conflicted.
2. Open the file.
3. Choose the correct final content.
4. Remove conflict markers.
5. Save the file.
6. Stage the resolved file.
7. Complete the merge commit.

---

## 5.4 Conflict Resolution Strategies

Common strategies:

| Strategy              | Description                                        |
| --------------------- | -------------------------------------------------- |
| Keep current version  | Use content from your current branch               |
| Keep incoming version | Use content from the branch being merged           |
| Combine both changes  | Manually merge both ideas                          |
| Rewrite the section   | Replace both versions with a cleaner final version |
| Ask teammate          | Discuss if the correct result is unclear           |

---

## 5.5 Useful Conflict Commands

Show conflict status:

```bash
git status
```

See conflicted files:

```bash
git diff --name-only --diff-filter=U
```

Keep current branch version:

```bash
git checkout --ours file.txt
git add file.txt
```

Keep incoming branch version:

```bash
git checkout --theirs file.txt
git add file.txt
```

Abort merge:

```bash
git merge --abort
```

Abort rebase:

```bash
git rebase --abort
```

Continue rebase after resolving conflict:

```bash
git add file.txt
git rebase --continue
```

---

# 6. Branching Strategies

---

## 6.1 What is a Branching Strategy?

A **branching strategy** defines how a team creates, names, merges, and releases branches.

It helps teams avoid confusion and maintain stable code.

A good branching strategy answers:

- Where should new work happen?
- Which branch is production-ready?
- How are releases prepared?
- How are hotfixes handled?
- When should changes be merged?

---

## 6.2 Git Flow

**Git flow** is a branching model with multiple long-lived branches.

Common branches:

| Branch      | Purpose                                 |
| ----------- | --------------------------------------- |
| `main`      | Production-ready code                   |
| `develop`   | Integration branch for upcoming release |
| `feature/*` | New features                            |
| `release/*` | Release preparation                     |
| `hotfix/*`  | Emergency production fixes              |

Example flow:

```text
main
develop
feature/login
release/v1.0
hotfix/payment-bug
```

### Advantages

- Clear release structure
- Good for scheduled releases
- Separates development and production code
- Useful for larger release cycles

### Disadvantages

- More complex
- More long-lived branches
- More merging overhead
- Slower feedback loop

---

## 6.3 Trunk-Based Development

**Trunk-based development** is a strategy where developers integrate small changes into the main branch frequently.

The main branch is often called:

```text
main
```

or:

```text
trunk
```

Feature branches are short-lived.

Typical flow:

```text
main -> short feature branch -> pull request -> merge quickly
```

### Advantages

- Simple workflow
- Faster integration
- Fewer long-lived branches
- Reduces large merge conflicts
- Works well with CI/CD

### Disadvantages

- Requires strong automated testing
- Requires discipline with small commits
- Incomplete features may need feature flags

---

## 6.4 Git Flow vs Trunk-Based Development

| Topic              | Git Flow           | Trunk-Based Development     |
| ------------------ | ------------------ | --------------------------- |
| Main branch        | Production code    | Main integration branch     |
| Development branch | Uses `develop`     | Usually no `develop`        |
| Feature branches   | Can live longer    | Short-lived                 |
| Release process    | More structured    | Continuous                  |
| Complexity         | Higher             | Lower                       |
| Best for           | Scheduled releases | CI/CD and frequent releases |
|                    |                    |                             |

---

# 7. Day 8 Practical Lab

---

## Lab 1: Create a Repository

```bash
mkdir day8-branching-lab
cd day8-branching-lab
git init
```

Configure Git if needed:

```bash
git config user.name "Your Name"
git config user.email "you@example.com"
```

Create first file:

```bash
cat > app.txt <<'EOF'
Application Name: Git Branching Lab
Version: 1.0.0
Status: Initial
EOF
```

Commit:

```bash
git add app.txt
git commit -m "Add initial app file"
```

Check branch:

```bash
git branch
```

If your branch is not `main`, rename it:

```bash
git branch -M main
```

---

## Lab 2: Create a Feature Branch

```bash
git switch -c feature/update-status
```

Modify file:

```bash
cat > app.txt <<'EOF'
Application Name: Git Branching Lab
Version: 1.0.0
Status: Feature branch update
EOF
```

Commit:

```bash
git add app.txt
git commit -m "Update app status in feature branch"
```

View history:

```bash
git log --oneline --graph --decorate --all
```

---

## Lab 3: Merge Feature Branch Back to Main

Switch to main:

```bash
git switch main
```

Merge:

```bash
git merge feature/update-status
```

View graph:

```bash
git log --oneline --graph --decorate --all
```

Delete feature branch:

```bash
git branch -d feature/update-status
```

---

## Lab 4: Create a 3-Way Merge Example

Create a new feature branch:

```bash
git switch -c feature/add-description
```

Add a description:

```bash
cat >> app.txt <<'EOF'
Description: This project is used to practice Git branching.
EOF
```

Commit:

```bash
git add app.txt
git commit -m "Add app description"
```

Switch back to main:

```bash
git switch main
```

Make a different change on main:

```bash
echo "Maintainer: DevOps Team" >> app.txt
git add app.txt
git commit -m "Add maintainer information"
```

Merge feature branch:

```bash
git merge feature/add-description
```

This should create a 3-way merge if both branches moved forward.

View graph:

```bash
git log --oneline --graph --decorate --all
```

---

## Lab 5: Intentionally Create a Merge Conflict

Start from main:

```bash
git switch main
```

Create a clean file:

```bash
cat > config.txt <<'EOF'
APP_ENV=development
APP_PORT=3000
LOG_LEVEL=info
EOF

git add config.txt
git commit -m "Add config file"
```

Create branch A:

```bash
git switch -c feature/change-port
```

Edit same line:

```bash
cat > config.txt <<'EOF'
APP_ENV=development
APP_PORT=4000
LOG_LEVEL=info
EOF

git add config.txt
git commit -m "Change app port to 4000"
```

Switch back to main:

```bash
git switch main
```

Create branch B:

```bash
git switch -c feature/change-port-main
```

Edit the same line differently:

```bash
cat > config.txt <<'EOF'
APP_ENV=development
APP_PORT=5000
LOG_LEVEL=info
EOF

git add config.txt
git commit -m "Change app port to 5000"
```

Switch to main and merge one branch:

```bash
git switch main
git merge feature/change-port
```

Now merge the second branch:

```bash
git merge feature/change-port-main
```

Git should show a merge conflict.

---

## Lab 6: Resolve the Merge Conflict

Check status:

```bash
git status
```

Open conflicted file:

```bash
nano config.txt
```

You may see:

```text
APP_ENV=development
<<<<<<< HEAD
APP_PORT=4000
=======
APP_PORT=5000
>>>>>>> feature/change-port-main
LOG_LEVEL=info
```

Resolve it by choosing the final value:

```text
APP_ENV=development
APP_PORT=4000
LOG_LEVEL=info
```

or combine according to your decision.

Stage resolved file:

```bash
git add config.txt
```

Finish merge:

```bash
git commit
```

View final history:

```bash
git log --oneline --graph --decorate --all
```

---

## Lab 7: Practice Rebase

Create a branch:

```bash
git switch -c feature/rebase-demo
```

Make a commit:

```bash
echo "Rebase demo line" >> app.txt
git add app.txt
git commit -m "Add rebase demo line"
```

Switch to main and add another commit:

```bash
git switch main
echo "Main branch update" >> app.txt
git add app.txt
git commit -m "Update app file on main"
```

Rebase feature branch onto main:

```bash
git switch feature/rebase-demo
git rebase main
```

If conflicts happen, resolve them and run:

```bash
git add app.txt
git rebase --continue
```

View history:

```bash
git log --oneline --graph --decorate --all
```

---

## Lab 8: Interactive Rebase Demo

Make three small commits:

```bash
echo "Line A" >> notes.txt
git add notes.txt
git commit -m "Add line A"

echo "Line B" >> notes.txt
git add notes.txt
git commit -m "Add line B"

echo "Fix typo" >> notes.txt
git add notes.txt
git commit -m "fix"
```

Start interactive rebase:

```bash
git rebase -i HEAD~3
```

Change:

```text
pick commit1 Add line A
pick commit2 Add line B
pick commit3 fix
```

To:

```text
pick commit1 Add line A
pick commit2 Add line B
squash commit3 fix
```

Save and close the editor.

Then edit the final commit message if Git asks.

---

# 8. Assignment 8

---

## Assignment: Simulate a Team Workflow

### Objective

Simulate two developers working on two different branches. Both branches must modify the same file and create a conflict. Then resolve the conflict correctly.

---

## Requirements

Your assignment must include:

1. A Git repository.
2. A `main` branch.
3. Two feature branches.
4. Both branches modifying the same file.
5. A merge conflict.
6. A resolved final version.
7. A final Git history graph.

---

## Suggested Scenario

File:

```text
team-notes.md
```

Initial content:

```markdown
# Team Notes

## Deployment Plan

Deployment will happen on Friday.

## Owner

Owner: TBD
```

Branch 1:

```text
feature/backend-plan
```

Change deployment plan to:

```markdown
Deployment will happen on Friday after backend testing.
```

Branch 2:

```text
feature/frontend-plan
```

Change the same line to:

```markdown
Deployment will happen on Friday after frontend testing.
```

When both branches are merged, Git should produce a conflict.

Final resolved version could be:

```markdown
Deployment will happen on Friday after both backend and frontend testing.
```

---

## Step-by-Step Assignment Commands

Create repository:

```bash
mkdir day8-team-workflow
cd day8-team-workflow
git init
git branch -M main
```

Create file:

```bash
cat > team-notes.md <<'EOF'
# Team Notes

## Deployment Plan

Deployment will happen on Friday.

## Owner

Owner: TBD
EOF

git add team-notes.md
git commit -m "Add initial team notes"
```

Create first branch:

```bash
git switch -c feature/backend-plan
```

Edit file:

```bash
cat > team-notes.md <<'EOF'
# Team Notes

## Deployment Plan

Deployment will happen on Friday after backend testing.

## Owner

Owner: Backend Team
EOF

git add team-notes.md
git commit -m "Update deployment plan for backend"
```

Create second branch from main:

```bash
git switch main
git switch -c feature/frontend-plan
```

Edit file:

```bash
cat > team-notes.md <<'EOF'
# Team Notes

## Deployment Plan

Deployment will happen on Friday after frontend testing.

## Owner

Owner: Frontend Team
EOF

git add team-notes.md
git commit -m "Update deployment plan for frontend"
```

Merge first branch:

```bash
git switch main
git merge feature/backend-plan
```

Merge second branch:

```bash
git merge feature/frontend-plan
```

Resolve conflict in `team-notes.md`:

```markdown
# Team Notes

## Deployment Plan

Deployment will happen on Friday after both backend and frontend testing.

## Owner

Owner: Backend and Frontend Team
```

Finish conflict resolution:

```bash
git add team-notes.md
git commit -m "Resolve deployment plan conflict"
```

View final graph:

```bash
git log --oneline --graph --decorate --all
```

---

## Assignment Submission Format

Students should submit:

```text
Repository name:
Branch names:
Conflict file:
Final resolved content:
Commands used:
git log --oneline --graph --decorate --all output:
Screenshot or terminal output:
```

---

# 9. Quick Command Summary

## Branching

```bash
git branch
git branch -a
git switch -c feature/name
git switch main
git branch -d feature/name
```

## Merging

```bash
git switch main
git merge feature/name
git merge --no-ff feature/name
```

## Rebase

```bash
git switch feature/name
git rebase main
git rebase -i HEAD~3
```

## Conflict Resolution

```bash
git status
git diff --name-only --diff-filter=U
nano conflicted-file.txt
git add conflicted-file.txt
git commit
```

## Abort Operations

```bash
git merge --abort
git rebase --abort
```

## History

```bash
git log --oneline --graph --decorate --all
```

---

# 10. Common Beginner Mistakes

- Forgetting which branch they are currently on.
- Creating a feature branch from the wrong branch.
- Merging into the wrong branch.
- Thinking a branch is a full copy of the project.
- Forgetting that a branch is just a pointer to a commit.
- Leaving conflict markers inside a file.
- Running `git commit` before resolving all conflicts.
- Using `git rebase` on a shared branch.
- Using `git reset --hard` during conflict resolution and losing work.
- Deleting a branch before confirming that it has been merged.
- Not checking history with `git log --graph`.

---

# 11. End-of-Day Review Questions

1. What is a Git branch?
2. Why is a branch called a pointer?
3. What does `HEAD` point to?
4. What is a detached HEAD?
5. What is a fast-forward merge?
6. What is a 3-way merge?
7. What is a merge commit?
8. What is the difference between merge and rebase?
9. Why should you avoid rebasing shared branches?
10. What does interactive rebase do?
11. What do conflict markers look like?
12. How do you resolve a merge conflict?
13. What is Git flow?
14. What is trunk-based development?
15. Which workflow is better for frequent CI/CD releases?

---

# 12. Expected Practical Output

By the end of the lab, students should have:

- Created and switched branches.
- Made commits on a feature branch.
- Merged a feature branch into `main`.
- Viewed branch history with graph output.
- Created an intentional merge conflict.
- Resolved the conflict manually.
- Practiced basic rebase.
- Understood the difference between Git flow and trunk-based development.
- Completed the team workflow assignment with two branches modifying the same file.
