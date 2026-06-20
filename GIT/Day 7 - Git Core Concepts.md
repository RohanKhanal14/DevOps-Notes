# Day 7 - Git Core Concepts

## Topic

**Git Core Concepts**

Focus areas:

- Version control fundamentals
- Git repository structure
- Working tree, staging area, and repository
- Staging and committing changes
- Viewing history
- Comparing changes
- Undoing changes safely
- Partial staging with `git add -p`
- Creating a Node.js project with meaningful Git commits

---

## Learning Outcomes

By the end of Day 7, students should be able to:

- Explain what version control is and why it is important.
- Understand why Git became the most widely used version control system.
- Explain the purpose of the `.git` directory.
- Understand Git internals such as objects, refs, and `HEAD`.
- Differentiate between the working tree, staging area, and repository.
- Use common Git commands such as `git add`, `git commit`, `git status`, `git log`, `git diff`, and `git show`.
- Initialize a Git repository.
- Make meaningful commits.
- View commit history using `git log --graph`.
- Stage partial changes using `git add -p`.
- Create a Node.js project and track it properly with `.gitignore`.

---

# 1. Theory: Version Control Fundamentals

---

## 1.1 What is Version Control?

**Version control** is a system used to track changes in files over time.

It allows developers and system administrators to:

- Track what changed
- See who changed something
- Understand why something changed
- Go back to an earlier version
- Work safely on new features
- Collaborate with others
- Recover from mistakes

Without version control, people often manage files like this:

```text
project-final.zip
project-final-v2.zip
project-final-new.zip
project-final-latest.zip
project-final-real-final.zip
```

This becomes confusing and unsafe.

With Git, every important change can be stored as a commit with a clear message.

---

## 1.2 Why Version Control is Important

Version control is important because software and infrastructure files change frequently.

Examples of files that should be version controlled:

- Source code
- Shell scripts
- Configuration files
- Infrastructure as Code files
- Documentation
- Dockerfiles
- CI/CD pipeline files
- Kubernetes manifests
- Terraform files

Version control helps teams avoid losing work and makes collaboration easier.

---

## 1.3 Centralized vs Distributed Version Control

There are two common types of version control systems.

| Type | Description | Example |
|---|---|---|
| Centralized Version Control | One central server stores the full history | SVN |
| Distributed Version Control | Every developer has a full copy of the repository history | Git |

In centralized systems, users depend heavily on the central server.

In distributed systems like Git, each developer has the full repository history locally. This makes Git fast and resilient.

---

## 1.4 Why Git Won

Git became popular because it is:

- Fast
- Distributed
- Reliable
- Branch-friendly
- Suitable for both small and large projects
- Good for collaboration
- Widely supported by platforms such as GitHub, GitLab, and Bitbucket

Git allows developers to work offline, create branches cheaply, and merge changes efficiently.

Git is now widely used in software development, DevOps, cloud engineering, infrastructure automation, and documentation workflows.

---

# 2. Git Core Architecture

---

## 2.1 What is a Git Repository?

A **Git repository** is a project folder that Git tracks.

A folder becomes a Git repository when you run:

```bash
git init
```

This creates a hidden directory named:

```text
.git
```

The `.git` directory stores all Git tracking information and history.

---

## 2.2 The `.git` Directory

The `.git` directory is the internal database of Git.

It contains:

- Commit history
- Object database
- Branch references
- Tags
- Configuration
- Staging area information
- `HEAD` pointer

You can view it with:

```bash
ls -la
ls -la .git
```

Example structure:

```text
.git/
├── HEAD
├── config
├── objects/
├── refs/
├── index
├── logs/
└── hooks/
```

### Important Warning

Do not manually edit or delete files inside `.git` unless you fully understand Git internals. Deleting `.git` removes the repository history from that folder.

---

## 2.3 Git Objects

Git stores data as objects inside:

```text
.git/objects
```

Important Git object types:

| Object Type | Purpose |
|---|---|
| Blob | Stores file content |
| Tree | Stores directory structure |
| Commit | Stores snapshot metadata |
| Tag | Stores tag information |

Git does not simply store files as normal copies. Instead, it stores content as objects identified by hashes.

A Git hash looks like this:

```text
a3f5c9e8d7...
```

This hash uniquely identifies content.

---

## 2.4 Blob Object

A **blob** stores the content of a file.

Example:

```text
app.js content -> blob object
```

The blob does not store the file name. It only stores the file content.

File names and directory structure are stored by tree objects.

---

## 2.5 Tree Object

A **tree** stores directory structure.

It connects:

- File names
- Directory names
- Blob objects
- Other tree objects

Example:

```text
project/
├── app.js
├── package.json
└── README.md
```

This directory structure is represented internally by a tree object.

---

## 2.6 Commit Object

A **commit** stores a snapshot of the project at a point in time.

A commit contains:

- Reference to a tree object
- Parent commit hash
- Author information
- Committer information
- Timestamp
- Commit message

A commit is like a saved checkpoint.

Example:

```bash
git commit -m "Add initial project files"
```

---

## 2.7 refs

`refs` are references to commits.

They are stored inside:

```text
.git/refs
```

Common refs:

| Ref Type | Example |
|---|---|
| Branch | `refs/heads/main` |
| Tag | `refs/tags/v1.0.0` |
| Remote branch | `refs/remotes/origin/main` |

A branch is simply a movable pointer to a commit.

---

## 2.8 HEAD

`HEAD` points to the current branch or current commit.

View `HEAD`:

```bash
cat .git/HEAD
```

Example:

```text
ref: refs/heads/main
```

This means your current branch is `main`.

When you make a new commit, the branch pointer moves forward, and `HEAD` still points to that branch.

---

# 3. Git Working Areas

---

## 3.1 The Three Main Areas of Git

Git has three important working areas:

| Area | Meaning |
|---|---|
| Working tree | Your actual project files |
| Staging area | Prepared changes for next commit |
| Repository | Saved commit history |

---

## 3.2 Working Tree

The **working tree** is the project directory where you edit files.

Example:

```text
my-project/
├── app.js
├── package.json
└── README.md
```

When you create, edit, or delete files, those changes happen in the working tree.

Check working tree status:

```bash
git status
```

---

## 3.3 Staging Area

The **staging area** is where you prepare changes before committing them.

It is also called the **index**.

You add changes to the staging area using:

```bash
git add filename
```

or:

```bash
git add .
```

The staging area lets you decide exactly what should go into the next commit.

---

## 3.4 Repository

The **repository** stores committed history.

When you run:

```bash
git commit -m "message"
```

Git saves the staged changes into the repository as a commit.

---

## 3.5 Git Flow Example

Typical Git workflow:

```bash
edit files
git status
git add file
git commit -m "Write meaningful message"
git log
```

Flow:

```text
Working Tree -> Staging Area -> Repository
```

Example:

```bash
echo "Hello Git" > hello.txt
git status
git add hello.txt
git commit -m "Add hello text file"
```

---

# 4. Core Git Commands

---

## 4.1 git init

`git init` creates a new Git repository.

```bash
mkdir git-demo
cd git-demo
git init
```

Check hidden `.git` directory:

```bash
ls -la
```

---

## 4.2 git status

`git status` shows the current state of your working tree and staging area.

```bash
git status
```

It shows:

- Current branch
- Untracked files
- Modified files
- Staged files
- Files ready to commit

Use `git status` often. It is one of the safest Git commands.

---

## 4.3 git add

`git add` stages files for commit.

Stage one file:

```bash
git add README.md
```

Stage all changes:

```bash
git add .
```

Stage all tracked and untracked changes:

```bash
git add -A
```

Stage interactively:

```bash
git add -p
```

---

## 4.4 git commit

`git commit` saves staged changes to repository history.

```bash
git commit -m "Add README file"
```

A good commit message should clearly describe what changed.

Good examples:

```text
Add initial Express server
Configure environment variables
Add healthcheck endpoint
Update README with setup steps
Ignore node_modules directory
```

Poor examples:

```text
update
changes
fix
final
done
```

---

## 4.5 git log

`git log` shows commit history.

```bash
git log
```

Compact one-line history:

```bash
git log --oneline
```

Graph view:

```bash
git log --oneline --graph --decorate --all
```

This is useful for visualizing branches and commit history.

---

## 4.6 git diff

`git diff` shows changes in the working tree that are not yet staged.

```bash
git diff
```

Show staged changes:

```bash
git diff --staged
```

or:

```bash
git diff --cached
```

Compare two commits:

```bash
git diff commit1 commit2
```

---

## 4.7 git show

`git show` displays details of a commit.

```bash
git show
```

Show a specific commit:

```bash
git show <commit-hash>
```

It displays:

- Commit hash
- Author
- Date
- Commit message
- Code changes

---

## 4.8 git restore

`git restore` is used to undo working tree changes.

Restore one file:

```bash
git restore filename
```

Restore all modified files:

```bash
git restore .
```

Unstage a staged file:

```bash
git restore --staged filename
```

Important: `git restore filename` discards uncommitted changes in that file.

---

## 4.9 git rm

`git rm` removes a file and stages the deletion.

```bash
git rm oldfile.txt
git commit -m "Remove old file"
```

Remove from Git but keep file locally:

```bash
git rm --cached .env
```

This is commonly used when a file was accidentally tracked but should be ignored.

---

# 5. Undoing Changes Safely

---

## 5.1 Undo Unstaged Changes

If you edited a file but have not staged it:

```bash
git restore file.txt
```

This discards local changes in that file.

---

## 5.2 Unstage a File

If you staged a file but do not want it in the next commit:

```bash
git restore --staged file.txt
```

The changes remain in your working tree but are removed from the staging area.

---

## 5.3 Amend the Last Commit

If you forgot to add something to the last commit:

```bash
git add forgotten-file.txt
git commit --amend
```

Or update the message only:

```bash
git commit --amend -m "New commit message"
```

Use amend carefully if you already pushed the commit to a shared remote.

---

## 5.4 Revert a Commit

`git revert` creates a new commit that undoes an older commit.

```bash
git revert <commit-hash>
```

This is safer for shared branches because it does not rewrite history.

---

## 5.5 Reset Warning

`git reset` can move branch history and may delete work depending on the option.

Examples:

```bash
git reset --soft HEAD~1
git reset --mixed HEAD~1
git reset --hard HEAD~1
```

For beginners, avoid `git reset --hard` unless you are sure. It can permanently discard uncommitted work.

---

# 6. `.gitignore`

---

## 6.1 What is `.gitignore`?

`.gitignore` tells Git which files or folders should not be tracked.

Common examples:

```text
node_modules/
.env
dist/
coverage/
*.log
```

Create `.gitignore`:

```bash
nano .gitignore
```

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
.DS_Store
```

---

## 6.2 Why `.gitignore` is Important

You should not commit:

- Dependency folders such as `node_modules/`
- Secret files such as `.env`
- Build output such as `dist/`
- Log files
- Temporary files
- OS-specific files

This keeps the repository clean and secure.

---

# 7. Day 7 Practical Lab

---

## Lab 1: Initialize a Git Repository

Create a demo directory:

```bash
mkdir day7-git-lab
cd day7-git-lab
git init
```

Check repository:

```bash
ls -la
git status
```

---

## Lab 2: Configure Git Identity

Set your name and email:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

Check configuration:

```bash
git config --list
```

For one repository only:

```bash
git config user.name "Your Name"
git config user.email "you@example.com"
```

---

## Lab 3: Make the First Commit

Create a file:

```bash
echo "# Day 7 Git Lab" > README.md
```

Check status:

```bash
git status
```

Stage and commit:

```bash
git add README.md
git commit -m "Add README"
```

View history:

```bash
git log --oneline
```

---

## Lab 4: Make Multiple Commits

Create an app file:

```bash
echo 'console.log("Hello Git");' > app.js
git add app.js
git commit -m "Add basic app file"
```

Update README:

```bash
echo "This repository is for Git practice." >> README.md
git add README.md
git commit -m "Update README with project purpose"
```

Add `.gitignore`:

```bash
echo "node_modules/" > .gitignore
echo ".env" >> .gitignore
git add .gitignore
git commit -m "Add Git ignore rules"
```

---

## Lab 5: View History with Graph

```bash
git log --oneline --graph --decorate --all
```

More detailed view:

```bash
git log --graph --pretty=format:'%h - %an, %ar : %s'
```

---

## Lab 6: Use git diff

Modify `app.js`:

```bash
echo 'console.log("Second line");' >> app.js
```

Show unstaged diff:

```bash
git diff
```

Stage it:

```bash
git add app.js
```

Show staged diff:

```bash
git diff --staged
```

Commit:

```bash
git commit -m "Add second console message"
```

---

## Lab 7: Use git show

Show latest commit:

```bash
git show
```

Show a specific commit:

```bash
git log --oneline
git show <commit-hash>
```

---

## Lab 8: Practice Partial Staging with `git add -p`

Partial staging allows you to stage only selected parts of a file.

Edit `README.md` and add two separate sections:

```bash
cat >> README.md <<'EOF'

## Setup

Run the application using Node.js.

## Notes

This section is for practice notes.
EOF
```

Now run:

```bash
git diff
git add -p README.md
```

Git will ask what to do with each change.

Common options:

| Option | Meaning |
|---|---|
| `y` | Stage this hunk |
| `n` | Do not stage this hunk |
| `s` | Split hunk into smaller parts |
| `e` | Manually edit hunk |
| `q` | Quit |
| `?` | Show help |

After staging selected parts:

```bash
git diff
git diff --staged
git status
git commit -m "Add README setup section"
```

Then stage and commit remaining changes:

```bash
git add README.md
git commit -m "Add README notes section"
```

---

# 8. Assignment 7

---

## Assignment: Create a Node.js Project with 5 Meaningful Commits

### Objective

Create a simple Node.js project, add a proper `.gitignore`, and make at least 5 meaningful commits.

---

## 8.1 Step 1: Create Project Directory

```bash
mkdir day7-node-git-project
cd day7-node-git-project
git init
```

---

## 8.2 Step 2: Initialize Node.js Project

```bash
npm init -y
```

This creates:

```text
package.json
```

---

## 8.3 Step 3: Add `.gitignore`

Create `.gitignore`:

```bash
cat > .gitignore <<'EOF'
node_modules/
.env
.env.local
npm-debug.log*
coverage/
dist/
.DS_Store
EOF
```

Commit:

```bash
git add .gitignore
git commit -m "Add Git ignore rules"
```

---

## 8.4 Step 4: Create Basic Application

Create `index.js`:

```bash
cat > index.js <<'EOF'
console.log("Node.js Git project started");
EOF
```

Commit:

```bash
git add index.js package.json
git commit -m "Add initial Node.js application"
```

---

## 8.5 Step 5: Add Start Script

Edit `package.json` and add:

```json
"scripts": {
  "start": "node index.js"
}
```

Commit:

```bash
git add package.json
git commit -m "Add start script"
```

Test:

```bash
npm start
```

---

## 8.6 Step 6: Add README

Create `README.md`:

```bash
cat > README.md <<'EOF'
# Day 7 Node Git Project

This is a simple Node.js project created for Git practice.

## Run

```bash
npm start
```
EOF
```

Commit:

```bash
git add README.md
git commit -m "Add project README"
```

---

## 8.7 Step 7: Add a Healthcheck Function

Update `index.js`:

```bash
cat > index.js <<'EOF'
function healthcheck() {
  return {
    status: "ok",
    service: "day7-node-git-project"
  };
}

console.log("Node.js Git project started");
console.log(healthcheck());
EOF
```

Commit:

```bash
git add index.js
git commit -m "Add healthcheck function"
```

---

## 8.8 Verify 5 Meaningful Commits

Check history:

```bash
git log --oneline --graph --decorate
```

Expected commits:

```text
Add Git ignore rules
Add initial Node.js application
Add start script
Add project README
Add healthcheck function
```

---

## Assignment Submission Format

Students should submit:

```text
Repository name:
Total commits:
git log --oneline output:
.gitignore content:
Screenshot or terminal output:
```

---

# 9. Quick Command Summary

## Repository Setup

```bash
git init
git status
git add .
git commit -m "message"
```

## History

```bash
git log
git log --oneline
git log --oneline --graph --decorate --all
git show
```

## Difference

```bash
git diff
git diff --staged
```

## Undo

```bash
git restore file.txt
git restore --staged file.txt
git commit --amend
git revert <commit-hash>
```

## Partial Staging

```bash
git add -p
```

---

# 10. Common Beginner Mistakes

- Forgetting to run `git init`.
- Forgetting to configure Git username and email.
- Using unclear commit messages such as `update` or `final`.
- Running `git add .` without checking `git status`.
- Committing `node_modules/`.
- Committing `.env` files with secrets.
- Confusing working tree and staging area.
- Thinking `git add` means uploading to GitHub.
- Thinking `git commit` means uploading to GitHub.
- Using `git reset --hard` without understanding that it can delete work.
- Forgetting to check `git diff` before committing.

---

# 11. End-of-Day Review Questions

1. What is version control?
2. Why is Git called a distributed version control system?
3. What is stored inside the `.git` directory?
4. What is the difference between a blob, tree, and commit object?
5. What does `HEAD` point to?
6. What is the difference between working tree, staging area, and repository?
7. What does `git status` show?
8. What does `git add` do?
9. What does `git commit` do?
10. What is the purpose of `git diff`?
11. What is the purpose of `git show`?
12. How do you view Git history as a graph?
13. What does `git add -p` do?
14. Why should `node_modules/` be added to `.gitignore`?
15. What makes a commit message meaningful?

---

# 12. Expected Practical Output

By the end of the lab, students should have:

- Initialized a Git repository.
- Created multiple commits.
- Viewed history using `git log --graph`.
- Practiced `git diff` and `git show`.
- Practiced partial staging using `git add -p`.
- Created a Node.js project.
- Added a `.gitignore`.
- Made at least 5 meaningful commits.
