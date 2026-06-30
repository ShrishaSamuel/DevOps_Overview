# Git — Zero to Hero

> A complete, beginner-to-advanced guide to Git version control for developers and DevOps engineers.

---

## Table of Contents

1. [Introduction & Theory](#1-introduction--theory)
2. [Installation & Setup](#2-installation--setup)
3. [Core Concepts](#3-core-concepts)
4. [Hands-on Tasks](#4-hands-on-tasks)
5. [Projects](#5-projects)
6. [Best Practices & Common Pitfalls](#6-best-practices--common-pitfalls)
7. [Interview Questions](#7-interview-questions)
8. [Quizzes](#8-quizzes)
9. [Further Resources](#9-further-resources)

---

## 1. Introduction & Theory

### 1.1 What is Git?

**Git** is a **distributed version control system (DVCS)** created by Linus Torvalds in 2005 to manage the development of the Linux kernel. It tracks changes to files over time, enabling multiple people to collaborate, experiment safely, and recover any previous state of a project.

"Version control" answers questions like:
- What changed, when, and by whom?
- How do I go back to a working version?
- How do multiple people edit the same codebase without overwriting each other?

### 1.2 Centralized vs. Distributed VCS

| Aspect | Centralized (SVN, CVS) | Distributed (Git) |
|--------|------------------------|-------------------|
| Repository | One central server | Every clone is a full repo |
| Offline work | Limited | Full history available offline |
| Speed | Network-bound | Local operations are instant |
| Backup | Single point of failure | Every clone is a backup |
| Branching | Expensive | Cheap and fast |

In Git, **every developer has the complete history** of the project. You can commit, branch, and view history without a network connection.

### 1.3 Why Git matters for DevOps

- **Source of truth** for code and Infrastructure as Code (Terraform, Ansible, Kubernetes manifests).
- **Foundation of CI/CD** — pipelines trigger on commits, tags, and merge requests.
- **GitOps** — declarative infrastructure where Git is the single source of truth and the desired state.
- **Collaboration** — pull/merge requests, code review, and history auditing.

### 1.4 How Git works internally (the theory)

Git is fundamentally a **content-addressable filesystem** with a version control UI on top. The key idea: Git stores **snapshots**, not diffs.

Core object types in the `.git` database:

| Object | Purpose |
|--------|---------|
| **blob** | The content of a file (no name, no metadata) |
| **tree** | A directory listing: names → blobs/trees + permissions |
| **commit** | A snapshot pointer: root tree + parent(s) + author + message |
| **tag** | A named, often annotated pointer to a commit |

Every object is identified by a **SHA-1 (or SHA-256) hash** of its content. Identical content produces identical hashes, which Git uses for integrity and deduplication.

```
commit  ──► tree (root)
                ├── blob (README.md)
                ├── tree (src/)
                │      ├── blob (main.py)
                │      └── blob (utils.py)
                └── blob (.gitignore)
```

A commit points to its parent(s), forming a **Directed Acyclic Graph (DAG)** of history.

### 1.5 The three areas / states

```
 Working Directory      Staging Area (Index)        Repository (.git)
  (your files)   ──git add──►   (snapshot to be   ──git commit──►  (permanent
                                  committed)                          history)
       ▲                                                                 │
       └──────────────────── git checkout / restore ────────────────────┘
```

A file in Git is in one of these states:
- **Untracked** — Git doesn't know about it yet.
- **Modified** — changed but not staged.
- **Staged** — marked to go into the next commit.
- **Committed** — safely stored in the local repository.

### 1.6 When to use Git

- Any software project, big or small.
- Configuration files, infrastructure code, documentation.
- Writing/research where you want history (e.g., LaTeX, Markdown books).
Essentially: **any text-based work that evolves over time.**

---

## 2. Installation & Setup

### 2.1 Install Git

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install -y git

# RHEL/Fedora
sudo dnf install -y git

# macOS (Homebrew)
brew install git

# Windows
# Download from https://git-scm.com/download/win  (includes Git Bash)

# Verify
git --version    # e.g., git version 2.45.2
```

### 2.2 First-time configuration

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global core.editor "vim"     # or "code --wait"
git config --global pull.rebase false      # merge on pull (default)
git config --global color.ui auto

# View configuration
git config --list
git config --global --edit                 # open the global config file
```

Configuration scopes (most specific wins):
- `--system` — `/etc/gitconfig` (all users)
- `--global` — `~/.gitconfig` (current user)
- `--local` — `.git/config` (current repo, default)

### 2.3 Set up SSH keys for GitHub/GitLab

```bash
ssh-keygen -t ed25519 -C "you@example.com"
cat ~/.ssh/id_ed25519.pub          # copy this into GitHub/GitLab > SSH Keys
ssh -T git@github.com              # test the connection
```

### 2.4 Useful aliases

```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --decorate --all"
```

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Creating and cloning repositories

```bash
git init                          # initialize a new repo in current dir
git init myproject                # create dir + init
git clone <url>                   # clone a remote repo
git clone <url> myfolder          # clone into a named folder
git clone --depth 1 <url>         # shallow clone (latest commit only)
```

#### 3.1.2 The basic workflow

```bash
git status                        # see current state
git add file.txt                  # stage one file
git add .                         # stage everything in current dir
git add -A                        # stage all changes (incl. deletions)
git commit -m "Add feature X"     # commit staged changes
git commit -am "Quick fix"        # stage tracked files + commit
git log                           # view history
git log --oneline                 # compact history
```

A good commit cycle:

```bash
# 1. edit files
vim app.py
# 2. review what changed
git diff
# 3. stage
git add app.py
# 4. confirm what will be committed
git diff --staged
# 5. commit
git commit -m "Fix null pointer in login handler"
```

#### 3.1.3 Inspecting changes and history

```bash
git diff                          # unstaged changes
git diff --staged                 # staged changes
git diff HEAD~1 HEAD              # between two commits
git log --oneline --graph --all   # visual history
git log -p file.txt               # history with patches for a file
git log --author="Alice"          # filter by author
git log --since="2 weeks ago"
git show <commit>                 # details of a specific commit
git blame file.txt                # who last changed each line
```

#### 3.1.4 .gitignore

Create a `.gitignore` file to exclude files from tracking:

```gitignore
# Dependencies
node_modules/
venv/
__pycache__/

# Build artifacts
dist/
build/
*.o
*.class

# Secrets & env
.env
*.pem
credentials.json

# OS / editor
.DS_Store
.idea/
*.swp
```

> Note: `.gitignore` only affects **untracked** files. To stop tracking an already-committed file: `git rm --cached file`.

### 3.2 INTERMEDIATE

#### 3.2.1 Branching

A **branch** is a lightweight, movable pointer to a commit. The default is `main`.

```bash
git branch                        # list branches
git branch feature-login          # create a branch
git checkout feature-login        # switch to it
git switch feature-login          # modern equivalent
git checkout -b feature-login     # create + switch
git switch -c feature-login       # modern create + switch
git branch -d feature-login       # delete (safe)
git branch -D feature-login       # force delete
git branch -m old new             # rename
```

#### 3.2.2 Merging

```bash
git switch main
git merge feature-login           # merge feature into main
```

Two merge types:
- **Fast-forward:** main simply moves forward (no divergence).
- **Three-way merge:** creates a merge commit when histories diverged.

```bash
git merge --no-ff feature-login   # always create a merge commit
git merge --squash feature-login  # combine all changes into one staged set
```

#### 3.2.3 Resolving merge conflicts

When two branches change the same lines, Git inserts conflict markers:

```
<<<<<<< HEAD
current branch content
=======
incoming branch content
>>>>>>> feature-login
```

Resolve by editing the file to the desired result, removing markers, then:

```bash
git add conflicted_file.txt
git commit                        # completes the merge
# or, to abort:
git merge --abort
```

#### 3.2.4 Working with remotes

```bash
git remote -v                     # list remotes
git remote add origin <url>       # add a remote
git remote remove origin
git remote rename origin upstream

git fetch origin                  # download objects, don't merge
git pull origin main              # fetch + merge
git pull --rebase origin main     # fetch + rebase
git push origin main              # upload commits
git push -u origin feature        # push + set upstream tracking
git push --tags                   # push tags
git push origin --delete feature  # delete remote branch
```

The relationship:

```
git fetch  = download remote changes (no merge)
git merge  = combine branches
git pull   = fetch + merge
git push   = upload your commits
```

#### 3.2.5 Undoing changes

```bash
# Discard unstaged changes in a file
git restore file.txt              # (or: git checkout -- file.txt)

# Unstage a file (keep changes)
git restore --staged file.txt     # (or: git reset HEAD file.txt)

# Amend the last commit (message or content)
git commit --amend -m "Better message"

# Revert a commit (creates a new inverse commit — safe for shared history)
git revert <commit>

# Reset (moves branch pointer)
git reset --soft HEAD~1           # undo commit, keep changes staged
git reset --mixed HEAD~1          # undo commit, keep changes unstaged (default)
git reset --hard HEAD~1           # undo commit AND discard changes (DANGEROUS)
```

#### 3.2.6 Stashing

```bash
git stash                         # save uncommitted changes, clean working dir
git stash push -m "WIP login"     # named stash
git stash list                    # view stashes
git stash apply                   # re-apply most recent (keep in stash)
git stash pop                     # re-apply and remove from stash
git stash drop stash@{0}          # delete a stash
git stash clear                   # delete all stashes
git stash -u                      # include untracked files
```

#### 3.2.7 Tagging

```bash
git tag v1.0.0                    # lightweight tag
git tag -a v1.0.0 -m "Release 1.0.0"   # annotated tag (recommended)
git tag                           # list tags
git show v1.0.0
git push origin v1.0.0            # push a tag
git push origin --tags            # push all tags
git tag -d v1.0.0                 # delete local tag
git push origin --delete v1.0.0   # delete remote tag
```

### 3.3 ADVANCED

#### 3.3.1 Rebasing

Rebase **replays** your commits on top of another base, producing a linear history.

```bash
git switch feature
git rebase main                   # move feature's commits on top of main
```

Visualization:

```
Before:            After rebase onto main:
  A---B---C main      A---B---C main
       \                       \
        D---E feature           D'---E' feature
```

**Golden rule:** Never rebase commits that have been pushed and shared, because it rewrites history.

#### 3.3.2 Interactive rebase (cleaning up history)

```bash
git rebase -i HEAD~4              # edit the last 4 commits
```

In the editor you can:
- `pick` — keep the commit
- `reword` — change the message
- `edit` — pause to amend
- `squash` — combine into previous (keep both messages)
- `fixup` — combine into previous (discard this message)
- `drop` — remove the commit
- reorder lines to reorder commits

#### 3.3.3 Cherry-picking

```bash
git cherry-pick <commit>          # apply a specific commit onto current branch
git cherry-pick A^..B             # a range
git cherry-pick --no-commit <c>   # apply without committing
```

Useful for backporting a fix from `main` to a release branch.

#### 3.3.4 Reflog — your safety net

`git reflog` records where HEAD has been, even after resets/rebases. It lets you recover "lost" commits.

```bash
git reflog                        # list recent HEAD positions
git reset --hard HEAD@{2}         # jump back to a previous state
git checkout -b rescue HEAD@{5}   # branch off a lost commit
```

#### 3.3.5 Bisect — finding the bad commit

`git bisect` performs a binary search to find the commit that introduced a bug.

```bash
git bisect start
git bisect bad                    # current commit is broken
git bisect good v1.0.0            # this older commit worked
# Git checks out a midpoint; test, then mark:
git bisect good   # or  git bisect bad
# ... repeat until Git identifies the culprit
git bisect reset                  # return to original state
```

You can automate it: `git bisect run ./test.sh`.

#### 3.3.6 Submodules and worktrees

```bash
# Submodules: a repo inside a repo
git submodule add <url> libs/foo
git submodule update --init --recursive
git submodule update --remote

# Worktrees: multiple working directories from one repo
git worktree add ../hotfix main
git worktree list
git worktree remove ../hotfix
```

#### 3.3.7 Hooks and Git internals

Hooks are scripts in `.git/hooks/` that run on events:

```bash
.git/hooks/pre-commit       # run before a commit is created (linting, tests)
.git/hooks/commit-msg       # validate commit messages
.git/hooks/pre-push         # run before pushing
.git/hooks/post-merge       # run after a merge
```

Example `pre-commit` to block commits with debug prints:

```bash
#!/usr/bin/env bash
if git diff --cached | grep -nE "console\.log|pdb\.set_trace"; then
  echo "Remove debug statements before committing."
  exit 1
fi
```

Use frameworks like **pre-commit** (Python) or **husky** (Node) to manage hooks across a team.

#### 3.3.8 Branching strategies

| Strategy | Description | Best for |
|----------|-------------|----------|
| **Git Flow** | `main`, `develop`, `feature/*`, `release/*`, `hotfix/*` | Scheduled releases |
| **GitHub Flow** | `main` + short-lived feature branches + PRs | Continuous deployment |
| **GitLab Flow** | GitHub Flow + environment branches | Staged environments |
| **Trunk-Based** | Everyone commits to `main` frequently behind flags | High-velocity CI/CD |

#### 3.3.9 Useful power commands

```bash
git clean -fdn                    # dry-run: show what would be deleted
git clean -fd                     # remove untracked files & dirs
git reflog expire --expire=now --all
git gc --prune=now                # garbage collect / compress
git fsck --full                   # verify integrity
git log --stat                    # files changed per commit
git shortlog -sn                  # commit count by author
git rev-parse HEAD                # current commit hash
git archive -o release.zip HEAD   # export a snapshot
```

---

## 4. Hands-on Tasks

### Task 1: Initialize a repo and make your first commit

```bash
mkdir git-lab && cd git-lab
git init
echo "# Git Lab" > README.md
git add README.md
git commit -m "Initial commit"
git log --oneline
```

**Expected:** One commit listed, e.g., `a1b2c3d Initial commit`.

### Task 2: Stage selectively and inspect diffs

```bash
echo "line 1" > a.txt
echo "line 2" > b.txt
git add a.txt
git status
```

**Expected:** `a.txt` staged (green), `b.txt` untracked (red).

### Task 3: Create and merge a branch

```bash
git switch -c feature
echo "feature work" >> README.md
git commit -am "Add feature note"
git switch main
git merge feature
git log --oneline --graph --all
```

**Expected:** `main` now includes the feature commit.

### Task 4: Create and resolve a merge conflict

```bash
git switch -c branch-a
echo "Version A" > conflict.txt
git add conflict.txt && git commit -m "A version"

git switch main
echo "Version B" > conflict.txt
git add conflict.txt && git commit -m "B version"

git merge branch-a            # conflict!
# edit conflict.txt to resolve, then:
git add conflict.txt
git commit -m "Resolve conflict"
```

**Expected:** Merge completes after manual resolution.

### Task 5: Undo a commit three ways

```bash
echo "oops" >> README.md
git commit -am "Bad commit"

git revert HEAD               # safe, creates inverse commit
# OR
git reset --soft HEAD~1       # keep changes staged
# OR
git reset --hard HEAD~1       # discard entirely
```

**Expected:** Understand the difference between revert and reset.

### Task 6: Stash work in progress

```bash
echo "WIP" >> README.md
git stash
git status                    # clean
git stash pop
git status                    # WIP restored
```

### Task 7: Tag a release

```bash
git tag -a v1.0.0 -m "First release"
git show v1.0.0
git tag
```

### Task 8: Interactive rebase to squash commits

```bash
echo 1 > f.txt; git add f.txt; git commit -m "wip 1"
echo 2 >> f.txt; git commit -am "wip 2"
echo 3 >> f.txt; git commit -am "wip 3"
git rebase -i HEAD~3          # mark wip 2 and wip 3 as 'squash'
git log --oneline
```

**Expected:** Three commits combined into one.

### Task 9: Recover a "lost" commit with reflog

```bash
git commit --allow-empty -m "important"
HASH=$(git rev-parse HEAD)
git reset --hard HEAD~1       # "lose" it
git reflog                    # find it
git cherry-pick "$HASH"       # recover
```

### Task 10: Connect to a remote and push

```bash
# Create an empty repo on GitHub/GitLab first, then:
git remote add origin git@github.com:you/git-lab.git
git push -u origin main
git remote -v
```

**Expected:** Code appears on the remote; upstream tracking is set.

---

## 5. Projects

### Project 1 (Beginner): Personal Dotfiles Repository

**Goal:** Version-control your shell and editor configuration.

Steps:
1. `git init` a `dotfiles` repo.
2. Add `.bashrc`, `.vimrc`, `.gitconfig`, aliases.
3. Write a `README.md` and an `install.sh` that symlinks files into `$HOME`.
4. Add a `.gitignore` for secrets.
5. Push to GitHub; tag `v1.0.0`.

**Skills:** init, add/commit, .gitignore, README, tagging, remotes.

### Project 2 (Intermediate): Collaborative Feature Workflow

**Goal:** Simulate team collaboration using branches and pull requests.

Steps:
1. Fork or create a shared repo.
2. Create a `feature/*` branch from `main`.
3. Make several commits with meaningful messages (Conventional Commits style).
4. Open a Pull/Merge Request; request review.
5. Address review feedback with `git commit --amend` / new commits.
6. Resolve a deliberate merge conflict.
7. Squash-merge and delete the branch.

**Skills:** branching strategy, PRs, conflict resolution, code review, commit hygiene.

### Project 3 (Advanced): Git-Backed CI/CD + GitOps Setup

**Goal:** Use Git as the source of truth driving automation.

Steps:
1. Structure a repo with app code + `.github/workflows/` (or `.gitlab-ci.yml`).
2. Configure a pipeline that runs on push: lint → test → build → deploy.
3. Add a `pre-commit` hook framework enforcing formatting and tests locally.
4. Adopt a branching strategy (GitHub Flow) with protected `main` and required reviews.
5. Use **tags** to trigger production releases (`v*` tags deploy).
6. Implement a GitOps directory of Kubernetes/Terraform manifests; a controller (e.g., Argo CD) or pipeline reconciles the cluster to match Git.
7. Document rollback by reverting a commit/tag.

**Skills:** hooks, CI/CD integration, tags as releases, protected branches, GitOps principles, rollback strategy.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Commit small and often** with focused, atomic changes.
- **Write good commit messages:** imperative mood, short summary (≤50 chars), blank line, then details. Consider **Conventional Commits** (`feat:`, `fix:`, `docs:`, `refactor:`).
- **Branch per feature/bug**; keep branches short-lived.
- **Never commit secrets** — use `.gitignore`, environment variables, and secret managers. If leaked, rotate the secret (history rewrite alone is not enough once pushed).
- **Pull/rebase before pushing** to reduce conflicts.
- **Use `.gitignore` early** to avoid committing build artifacts.
- **Review with `git diff --staged`** before committing.
- **Protect `main`** with required reviews and CI checks.
- **Tag releases** with annotated tags.
- **Use `git revert` for shared history**, `reset` only for local/unpushed work.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| `git push --force` to shared branch | Overwrites others' work | Use `--force-with-lease`; avoid on shared branches |
| Committing secrets | Security breach (lives in history) | Rotate the secret; use BFG/`filter-repo` to scrub; prevent with hooks |
| Giant "misc" commits | Hard to review/revert | Commit atomically |
| Rebasing shared/public branches | Breaks teammates' history | Only rebase local, unpushed commits |
| `git reset --hard` carelessly | Loses uncommitted work | Stash or commit first; reflog may save you |
| Forgetting to pull before work | Painful conflicts | `git pull --rebase` regularly |
| Committing to wrong branch | Messy history | `git stash`, switch, `git stash pop` |
| Tracking generated files | Bloated repo, conflicts | Add to `.gitignore`; `git rm --cached` |

---

## 7. Interview Questions

### Beginner

1. **What is the difference between `git fetch` and `git pull`?**
   *`fetch` downloads remote changes without merging; `pull` = `fetch` + `merge` (or rebase).*

2. **What is the staging area?**
   *An intermediate area (index) where you assemble exactly what will go into the next commit.*

3. **How do you create and switch to a new branch in one command?**
   *`git switch -c branch-name` (or `git checkout -b branch-name`).*

4. **What does `.gitignore` do?**
   *Tells Git which untracked files/patterns to ignore so they aren't tracked.*

5. **How do you see the commit history compactly?**
   *`git log --oneline`.*

### Intermediate

6. **Explain the difference between `merge` and `rebase`.**
   *Merge preserves history and creates a merge commit when branches diverge; rebase rewrites commits onto a new base for a linear history. Rebase should not be used on shared/published commits.*

7. **What's the difference between `git reset` and `git revert`?**
   *`reset` moves the branch pointer (rewrites history, local); `revert` creates a new commit that undoes a previous one (safe for shared history).*

8. **How do you undo the last commit but keep the changes?**
   *`git reset --soft HEAD~1` (keeps staged) or `--mixed` (keeps unstaged).*

9. **What is a fast-forward merge?**
   *When the target branch has no new commits, Git just moves its pointer forward to the source branch's tip — no merge commit created.*

10. **How do you resolve a merge conflict?**
    *Edit files to remove conflict markers and choose the correct content, `git add` them, then `git commit` (or `git merge --continue`).*

### Advanced

11. **What does `git cherry-pick` do and when is it useful?**
    *Applies the changes of a specific commit onto the current branch. Useful for backporting a fix to a release branch without merging everything.*

12. **How would you find which commit introduced a bug?**
    *`git bisect` — binary search marking commits good/bad, optionally automated with `git bisect run <test>`.*

13. **How can you recover a commit after a hard reset?**
    *Use `git reflog` to find the commit's hash, then `git checkout`/`cherry-pick`/`reset` to it.*

14. **Explain Git's object model.**
    *Content-addressable store of blobs (file contents), trees (directories), commits (snapshots with parent links), and tags — all identified by SHA hashes, forming a DAG.*

15. **What is the difference between `--force` and `--force-with-lease`?**
    *`--force` overwrites the remote unconditionally; `--force-with-lease` only overwrites if the remote hasn't changed since your last fetch, preventing accidental loss of others' work.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** Which command stages all changes including deletions?
- A) `git add .`  B) `git add -A`  C) `git stage`  D) `git commit -a`

**Q2.** What does HEAD refer to?
- A) The first commit  B) The remote branch  C) The current commit/branch pointer  D) The staging area

**Q3.** Which command creates an annotated tag?
- A) `git tag v1`  B) `git tag -a v1 -m "msg"`  C) `git tag --light v1`  D) `git branch v1`

**Q4.** Which is safe for rewriting shared history?
- A) `git rebase`  B) `git reset --hard`  C) `git revert`  D) `git commit --amend`

**Q5.** What does `git stash pop` do?
- A) Deletes all stashes  B) Lists stashes  C) Applies and removes the latest stash  D) Creates a new stash

**Q6.** Which command shows who changed each line of a file?
- A) `git log`  B) `git blame`  C) `git diff`  D) `git show`

**Q7.** A fast-forward merge happens when:
- A) Branches diverged  B) The target branch has no new commits  C) There's a conflict  D) You use `--no-ff`

**Q8.** Which records HEAD movements for recovery?
- A) `git log`  B) `git reflog`  C) `git status`  D) `git fsck`

**Q9.** What does `git pull --rebase` do?
- A) fetch + merge  B) fetch + rebase  C) push + rebase  D) reset + pull

**Q10.** Which flag makes a force push safer?
- A) `--force`  B) `--hard`  C) `--force-with-lease`  D) `--soft`

### Short Answer

**S1.** Write the command to create a branch named `bugfix` and switch to it (modern syntax).

**S2.** How do you unstage a file named `app.py` while keeping its changes?

**S3.** What command performs a binary search to find a buggy commit?

**S4.** How do you push a branch and set it to track the remote upstream?

**S5.** Write a command to view the last 3 commits in a one-line graph including all branches.

### Answer Key

**Multiple Choice:** Q1-B, Q2-C, Q3-B, Q4-C, Q5-C, Q6-B, Q7-B, Q8-B, Q9-B, Q10-C

**Short Answer:**
- **S1.** `git switch -c bugfix`
- **S2.** `git restore --staged app.py` (or `git reset HEAD app.py`)
- **S3.** `git bisect`
- **S4.** `git push -u origin <branch>`
- **S5.** `git log --oneline --graph --all -3`

---

## 9. Further Resources

### Official Documentation
- Pro Git book (free) — https://git-scm.com/book/en/v2
- Git reference manual — https://git-scm.com/docs
- GitHub Docs — https://docs.github.com
- GitLab Docs — https://docs.gitlab.com

### Interactive Learning
- Learn Git Branching (visual) — https://learngitbranching.js.org
- Oh My Git! (game) — https://ohmygit.org
- Git Immersion — https://gitimmersion.com
- Atlassian Git Tutorials — https://www.atlassian.com/git/tutorials

### Tools
- `git-extras`, `tig` (text UI), `lazygit`, `gitui`
- GitKraken, Sourcetree, GitHub Desktop (GUIs)
- pre-commit framework — https://pre-commit.com
- BFG Repo-Cleaner / `git filter-repo` — for scrubbing history

### Cheatsheets
- GitHub Git Cheat Sheet — https://education.github.com/git-cheat-sheet-education.pdf
- DevHints Git — https://devhints.io/git

### Reading
- Conventional Commits — https://www.conventionalcommits.org
- A successful Git branching model (Git Flow) — Vincent Driessen
- Trunk-Based Development — https://trunkbaseddevelopment.com

---

*End of Git — Zero to Hero.*
