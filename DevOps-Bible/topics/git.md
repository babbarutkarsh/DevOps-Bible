---
title: Git
nav_order: 51
description: "Git — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Git — DevOps Interview Preparation

## Table of Contents
- [Git Fundamentals](#git-fundamentals)
- [Branching Strategies](#branching-strategies)
- [Essential Commands](#essential-commands)
- [Merge vs Rebase](#merge-vs-rebase)
- [Git Internals](#git-internals)
- [Advanced Git](#advanced-git)
- [Scenario-Based Questions](#scenario-based-questions)

---

## Git Fundamentals

### Q: Explain Git's three areas.

```
Working Directory        Staging Area (Index)        Local Repository
(your files)             (git add)                   (git commit)
     │                        │                           │
     │── git add ───────────►│                           │
     │                        │── git commit ────────────►│
     │                        │                           │── git push ──► Remote
     │◄── git checkout ──────│                           │
     │                        │◄── git reset HEAD ───────│
     │◄────────────────── git reset --hard ──────────────│
```

### Q: What is a Git commit?

A commit is a **snapshot** of the entire project, not a diff. It contains:
- Tree object (directory structure pointing to blob objects)
- Parent commit hash(es)
- Author, committer, timestamp
- Commit message
- SHA-1 hash (unique identifier)

---

## Branching Strategies

### Q: Compare branching strategies.

**Trunk-Based Development (recommended for FAANG-level CI/CD):**
```
main ─────●────●────●────●────●────●──────
           \  /      \  /
            ●         ●
     (short-lived feature branches, <1 day)
```
- Engineers merge to `main` multiple times per day
- Feature flags for incomplete features
- Best for: Continuous deployment, high-velocity teams

**GitHub Flow:**
```
main ─────●──────●──────●──────●────────
           \    / \    /
            ●──●   ●──●
          (feature) (feature)
```
- Branch from main, PR back to main
- Simple, works for most teams
- Best for: Web apps with continuous delivery

**GitFlow:**
```
main    ────●────────────────●────────────
             \              /
develop ──●───●──●──●──●──●──●──●────────
            \  /    \     /
             ●       ●──●
          (feature) (release branch)
```
- `main` (production), `develop` (integration), feature branches, release branches, hotfix branches
- More complex, slower release cadence
- Best for: Products with versioned releases (mobile apps, libraries)

### Q: What are feature flags and why use them with trunk-based?

Feature flags allow merging incomplete code to main safely:
```python
if feature_flags.is_enabled("new-checkout-flow", user):
    return new_checkout()
else:
    return old_checkout()
```

**Benefits:** Decouple deployment from release, A/B testing, kill switch for broken features, gradual rollout.

**Tools:** LaunchDarkly, Unleash, Flagsmith, AWS AppConfig

---

## Essential Commands

### Q: Daily workflow commands.

```bash
# Setup
git clone <url>
git config --global user.name "Name"
git config --global user.email "email@example.com"

# Branch operations
git branch                          # List branches
git branch feature/login            # Create branch
git checkout -b feature/login       # Create and switch
git switch -c feature/login         # Modern alternative
git branch -d feature/login         # Delete (safe)
git branch -D feature/login         # Force delete

# Stage and commit
git add .                           # Stage all
git add -p                          # Interactive staging (review hunks)
git commit -m "feat: add login"
git commit --amend                  # Edit last commit

# Sync
git fetch origin                    # Download without merging
git pull origin main                # Fetch + merge
git pull --rebase origin main       # Fetch + rebase (cleaner history)
git push origin feature/login
git push -u origin feature/login    # Set upstream tracking

# Stash
git stash                           # Save work in progress
git stash list
git stash pop                       # Restore and remove
git stash apply                     # Restore, keep in stash

# History
git log --oneline --graph -20
git log --author="alice" --since="1 week ago"
git show <commit>
git diff                            # Working dir vs staging
git diff --staged                   # Staging vs last commit
git diff main..feature              # Between branches

# Undo
git restore <file>                  # Discard working changes
git restore --staged <file>         # Unstage
git revert <commit>                 # Create opposite commit (safe for shared history)
git reset --soft HEAD~1             # Undo commit, keep staged
git reset --mixed HEAD~1            # Undo commit, keep in working dir
git reset --hard HEAD~1             # Undo everything (DANGEROUS)
```

### Q: git reset --soft vs --mixed vs --hard?

| Mode | HEAD | Staging | Working Dir | Use Case |
|------|------|---------|-------------|----------|
| `--soft` | Moves | Unchanged | Unchanged | Redo commit message, squash |
| `--mixed` (default) | Moves | Reset | Unchanged | Unstage changes |
| `--hard` | Moves | Reset | Reset | Discard everything (dangerous) |

---

## Merge vs Rebase

### Q: When to merge vs rebase?

```
Merge (git merge):                    Rebase (git rebase):
    ●───●───●───● (main)                 ●───●───●───● (main)
     \       \ /                                       \
      ●───●───● (feature)                              ●'──●'──●' (feature)
              ↑ merge commit                           (replayed on top)
```

| Aspect | Merge | Rebase |
|--------|-------|--------|
| History | Preserves all history (merge commits) | Linear, clean history |
| Safety | Safe for shared branches | **Never rebase shared/public branches** |
| Conflicts | Resolve once | Resolve per commit |
| Use when | Merging to main (PR merge) | Updating feature branch from main |
| Command | `git merge main` | `git rebase main` |

**Golden rule:** Rebase local/private branches. Merge into shared branches.

### Q: What is interactive rebase?

```bash
git rebase -i HEAD~5     # Rewrite last 5 commits

# Editor opens:
pick a1b2c3d feat: add user model
pick b2c3d4e fix: typo in user model
pick c3d4e5f feat: add user API
pick d4e5f6g fix: API validation
pick e5f6g7h feat: add user tests

# Change to:
pick a1b2c3d feat: add user model
squash b2c3d4e fix: typo in user model     # Combine with previous
pick c3d4e5f feat: add user API
fixup d4e5f6g fix: API validation          # Combine, discard message
pick e5f6g7h feat: add user tests

# Commands: pick, reword, edit, squash, fixup, drop
```

---

## Git Internals

### Q: What are Git objects?

```
.git/objects/
├── Blob    — File content (SHA-1 of content)
├── Tree    — Directory listing (pointers to blobs and other trees)
├── Commit  — Snapshot (pointer to tree + parent + metadata)
└── Tag     — Named pointer to a commit

Commit → Tree → Blobs
  │        └──→ Trees → Blobs
  └──→ Parent Commit
```

### Q: What is HEAD, refs, and detached HEAD?

```
HEAD → .git/HEAD → "ref: refs/heads/main"
                              │
                              ▼
                    refs/heads/main → abc123 (commit hash)
```

- **HEAD** — Pointer to current branch tip
- **Detached HEAD** — HEAD points directly to a commit (not a branch). Any new commits are lost unless you create a branch.
- **refs** — Human-readable names for commits (`refs/heads/main`, `refs/tags/v1.0`)

---

## Advanced Git

### Q: Git hooks for DevOps.

```bash
# .git/hooks/ (local) or use tools like Husky, pre-commit

# pre-commit — Run before commit is created
# lint, format, secret scanning
#!/bin/sh
npm run lint
detect-secrets scan

# commit-msg — Validate commit message format
#!/bin/sh
if ! grep -qE "^(feat|fix|docs|style|refactor|test|chore):" "$1"; then
    echo "Commit message must follow Conventional Commits format"
    exit 1
fi

# pre-push — Run before push
# tests, build verification
#!/bin/sh
npm test
```

### Q: Git bisect for debugging.

```bash
git bisect start
git bisect bad                  # Current commit is broken
git bisect good v1.0.0          # Known good commit

# Git checks out middle commit
# You test it, then:
git bisect good                 # If this commit works
git bisect bad                  # If this commit is broken

# Git narrows down to the commit that introduced the bug
# Binary search: O(log n) commits to check

git bisect reset                # Exit bisect mode
```

### Q: Cherry-pick and its use cases.

```bash
git cherry-pick <commit-hash>   # Apply specific commit to current branch

# Use cases:
# - Hotfix: Apply a fix from develop to release branch
# - Backporting: Apply feature to older version
# - Selective merging: Pick specific commits without full merge

git cherry-pick --no-commit <hash>  # Stage changes without committing
git cherry-pick <hash1> <hash2>     # Multiple commits
```

---

## Scenario-Based Questions

### Q: You accidentally committed a secret (API key). How to remove it from history?

```bash
# 1. IMMEDIATELY rotate the secret (it's already exposed)

# 2. Remove from history
# Option A: BFG Repo Cleaner (faster, simpler)
bfg --replace-text passwords.txt repo.git

# Option B: git filter-branch (slower)
git filter-branch --force --tree-filter \
  "sed -i 's/API_KEY_VALUE/REMOVED/g' config.py" HEAD

# Option C: git filter-repo (recommended replacement for filter-branch)
git filter-repo --invert-paths --path secrets.yaml

# 3. Force push
git push --force --all

# 4. Inform team to re-clone

# 5. Prevention:
# - Use .gitignore for secret files
# - pre-commit hooks with detect-secrets/gitleaks
# - Use environment variables or secret managers
```

### Q: Two developers made conflicting changes. How to resolve?

```bash
git merge feature-branch
# CONFLICT in src/app.js

# 1. Open file, find conflict markers:
<<<<<<< HEAD
  const timeout = 5000;
=======
  const timeout = 3000;
>>>>>>> feature-branch

# 2. Resolve by editing (choose one, combine, or rewrite)
  const timeout = 5000;  // Decided to keep main's value

# 3. Stage and commit
git add src/app.js
git commit -m "merge: resolve timeout conflict, keep 5000ms"
```

### Q: You need to undo the last 3 commits on a shared branch.

```bash
# NEVER use git reset on shared branches
# Use git revert instead (creates opposite commits)

git revert --no-commit HEAD~3..HEAD   # Revert last 3 commits
git commit -m "revert: undo last 3 commits due to regression"
git push origin main
```

---

## Key Resources

- **Pro Git (book, free)** — https://git-scm.com/book
- **Oh Shit, Git!?!** — https://ohshitgit.com
- **Learn Git Branching** — https://learngitbranching.js.org (interactive)
- **Conventional Commits** — https://www.conventionalcommits.org
- **Atlassian Git Tutorials** — https://www.atlassian.com/git/tutorials
