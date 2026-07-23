---
title: Git
nav_order: 51
description: "Git — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Git — DevOps Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents
- [Git Fundamentals](#git-fundamentals)
- [Branching Strategies](#branching-strategies)
- [Essential Commands](#essential-commands)
- [Merge vs Rebase](#merge-vs-rebase)
- [Git Internals](#git-internals)
- [Tags and Releases](#tags-and-releases)
- [Advanced Git](#advanced-git)
- [Git Reflog & Recovery](#git-reflog--recovery)
- [Submodules vs Subtree](#submodules-vs-subtree)
- [Monorepo Techniques](#monorepo-techniques)
- [Commit Signing](#commit-signing)
- [Large Repos & Git LFS](#large-repos--git-lfs)
- [Scenario-Based Questions](#scenario-based-questions)

---

## Git Fundamentals

### 🔥 Q: Explain Git's three areas.

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

### 🔥 Q: What is a Git commit?

A commit is a **snapshot** of the entire project, not a diff. It contains:
- Tree object (directory structure pointing to blob objects)
- Parent commit hash(es)
- Author, committer, timestamp
- Commit message
- SHA-1 hash (unique identifier)

---

## Branching Strategies

### 🔥 Q: Compare branching strategies.

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

### ⭐ Q: What are feature flags and why use them with trunk-based?

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

### 🔥 Q: Daily workflow commands.

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

### 🔥 Q: git reset --soft vs --mixed vs --hard?

| Mode | HEAD | Staging | Working Dir | Use Case |
|------|------|---------|-------------|----------|
| `--soft` | Moves | Unchanged | Unchanged | Redo commit message, squash |
| `--mixed` (default) | Moves | Reset | Unchanged | Unstage changes |
| `--hard` | Moves | Reset | Reset | Discard everything (dangerous) |

### 🔥 Q: git reset vs git revert vs git restore?

```bash
# git reset — Move branch pointer (rewrites history, use locally only)
git reset --soft HEAD~1        # Undo commit, keep changes staged
git reset --mixed HEAD~1       # Undo commit, unstage changes
git reset --hard HEAD~1        # Undo commit, discard all changes

# git revert — Create opposite commit (safe for shared branches)
git revert <commit>            # Adds new commit that undoes changes
git revert --no-commit HEAD~3..HEAD  # Revert multiple, stage for single commit

# git restore — Discard working or staging area changes (no commit)
git restore <file>             # Discard working directory changes
git restore --staged <file>    # Unstage (keep working changes)
git restore --source=HEAD~2 <file>  # Restore file from older commit
```

| Command | Use When | Shared Branch Safe? | History |
|---------|----------|---------------------|---------|
| `git reset` | Local cleanup | ❌ No | Rewrites |
| `git revert` | Undo on shared branch | ✅ Yes | Adds commit |
| `git restore` | Discard uncommitted work | ✅ Yes | No change |

### 💡 Q: What is git worktree and when to use it?

`git worktree` allows multiple working directories from a single repo — each checked out to a different branch simultaneously.

```bash
# Check out main branch in separate directory
git worktree add ../myproject-main main

# Create new branch in separate worktree
git worktree add -b feature/new-api ../myproject-api

# List worktrees
git worktree list

# Remove worktree
git worktree remove ../myproject-main

# Prune stale worktrees
git worktree prune
```

**Use cases:**
- Review PR while keeping current feature branch untouched
- Run tests on main while developing in feature branch
- Work on hotfix without stashing or committing WIP
- Compare builds across branches side-by-side

---

## Merge vs Rebase

### 🔥 Q: When to merge vs rebase?

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

### ⭐ Q: Fast-forward merge vs no-ff merge vs squash merge?

```
Fast-Forward (default when possible):
    ●───●───●───● (main, feature merged)
    No merge commit

No-FF (--no-ff):
    ●───●───●───●───● (main)
             \     /
              ●───● (feature)
    Preserves branch history

Squash (--squash):
    ●───●───●───● (main)
             \
              ●───●───● (feature)
    All feature commits → single commit on main
```

| Type | Command | History | Use When |
|------|---------|---------|----------|
| Fast-forward | `git merge feature` (if possible) | Linear | Small, clean feature branch |
| No-FF | `git merge --no-ff feature` | Shows branch existed | Want to preserve feature context |
| Squash | `git merge --squash feature` | Collapses to 1 commit | Many WIP commits, want clean history |

**GitHub/GitLab PR merge strategies:**
- **Merge commit** — Preserves all commits + merge commit
- **Squash and merge** — Collapses PR to single commit (popular for trunk-based)
- **Rebase and merge** — Replays commits on main (linear history, no merge commit)

### ⭐ Q: What is interactive rebase?

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

### ⭐ Q: What are Git objects?

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

### ⭐ Q: What is HEAD, refs, and detached HEAD?

```
HEAD → .git/HEAD → "ref: refs/heads/main"
                              │
                              ▼
                    refs/heads/main → abc123 (commit hash)
```

- **HEAD** — Pointer to current branch tip
- **Detached HEAD** — HEAD points directly to a commit (not a branch). Any new commits are lost unless you create a branch.
- **refs** — Human-readable names for commits (`refs/heads/main`, `refs/tags/v1.0`)

### ⭐ Q: How to recover from detached HEAD?

```bash
# You're in detached HEAD state
git status
# HEAD detached at abc123

# SCENARIO 1: You made commits you want to keep
git branch recover-work        # Create branch at current position
git checkout main              # Return to main
git merge recover-work         # Or rebase, cherry-pick, etc.

# SCENARIO 2: You made no commits, just want to return
git checkout main              # Or any branch

# SCENARIO 3: You don't remember which branch you were on
git reflog                     # Find the commit before detached HEAD
git checkout -b recovered-branch <commit>
```

---

## Tags and Releases

### ⭐ Q: Lightweight vs annotated tags?

```bash
# Lightweight tag (just a pointer to a commit)
git tag v1.0.0

# Annotated tag (full Git object with metadata)
git tag -a v1.0.0 -m "Release version 1.0.0"

# View tag details
git show v1.0.0

# List all tags
git tag -l

# Push tags to remote
git push origin v1.0.0         # Push specific tag
git push origin --tags         # Push all tags
git push origin --follow-tags  # Push only annotated tags

# Delete tag
git tag -d v1.0.0              # Delete local
git push origin --delete v1.0.0  # Delete remote
```

| Aspect | Lightweight | Annotated |
|--------|-------------|-----------|
| Metadata | No | Yes (tagger, date, message) |
| Object type | Pointer | Full Git object |
| Signing | ❌ No | ✅ Can be GPG signed |
| Use for | Temporary, private | Releases, public versions |
| Recommended | ❌ No | ✅ Yes for releases |

**Best practice:** Always use annotated tags for releases.

### ⭐ Q: Semantic versioning for Git tags?

```bash
# Format: MAJOR.MINOR.PATCH
git tag -a v1.2.3 -m "Release 1.2.3"

# MAJOR — Breaking changes (v2.0.0)
# MINOR — New features, backward compatible (v1.3.0)
# PATCH — Bug fixes (v1.2.4)

# Pre-release versions
git tag -a v1.3.0-rc.1 -m "Release candidate 1"
git tag -a v1.3.0-alpha.1 -m "Alpha release"
git tag -a v1.3.0-beta.2 -m "Beta 2"
```

---

## Advanced Git

### 🔥 Q: Git hooks for DevOps.

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

### ⭐ Q: Git bisect for debugging.

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

### ⭐ Q: Cherry-pick and its use cases.

```bash
git cherry-pick <commit-hash>   # Apply specific commit to current branch

# Use cases:
# - Hotfix: Apply a fix from develop to release branch
# - Backporting: Apply feature to older version
# - Selective merging: Pick specific commits without full merge

git cherry-pick --no-commit <hash>  # Stage changes without committing
git cherry-pick <hash1> <hash2>     # Multiple commits

# Resolve conflicts during cherry-pick
git cherry-pick <commit>
# CONFLICT — fix conflicts
git add <resolved-files>
git cherry-pick --continue

# Abort cherry-pick
git cherry-pick --abort
```

### 💡 Q: What is git rerere and when is it useful?

**rerere** = "reuse recorded resolution" — Git remembers how you resolved conflicts and auto-applies the same resolution next time.

```bash
# Enable rerere
git config --global rerere.enabled true

# How it works:
1. You resolve a merge conflict in file X
2. Git records the resolution
3. Later, same conflict appears (e.g., rebasing, cherry-picking)
4. Git auto-applies your previous resolution

# View recorded resolutions
git rerere status
git rerere diff

# Forget a resolution
git rerere forget <path>
```

**Use cases:**
- Frequent rebasing of long-lived feature branches
- Repeated merge conflicts across team members
- Complex multi-step rebases

### ⭐ Q: git stash internals and advanced usage.

```bash
# Basic stash
git stash                      # Stash working + staged changes
git stash save "WIP: feature"  # With message

# Stash with untracked files
git stash -u                   # Include untracked
git stash -a                   # Include untracked + ignored

# Partial stash
git stash -p                   # Interactive hunk selection

# Inspect stash
git stash list
git stash show stash@{0}       # Summary
git stash show -p stash@{0}    # Full diff

# Apply and manage
git stash pop                  # Apply + delete
git stash apply stash@{2}      # Apply specific stash, keep in list
git stash drop stash@{1}       # Delete specific stash
git stash clear                # Delete all stashes

# Create branch from stash
git stash branch feature/recovered stash@{0}
```

**Internals:** Stash is a special commit on `refs/stash` with the working directory and index as separate commits.

---

## Git Reflog & Recovery

### 🔥 Q: What is git reflog and how does it save you?

`git reflog` records every HEAD movement — commits, resets, checkouts, rebases. It's your **time machine** for recovering "lost" work.

```bash
git reflog                     # Show HEAD history (last ~90 days)
git reflog show main           # Reflog for specific branch

# Output format:
abc123 HEAD@{0}: commit: Add feature X
def456 HEAD@{1}: rebase: checkout main
789abc HEAD@{2}: commit: WIP changes
...

# Recover lost commit
git reflog                     # Find the commit hash
git checkout -b recovered-branch abc123

# Or cherry-pick specific commit
git cherry-pick abc123

# Or reset branch to old state
git reset --hard HEAD@{3}
```

### 🔥 Q: Scenario: You ran `git reset --hard` and lost commits. Recover them.

```bash
# 1. Find the lost commit in reflog
git reflog
# abc123 HEAD@{1}: commit: Important feature (before reset)
# def456 HEAD@{0}: reset: moving to HEAD~3

# 2. Recover the commit
git checkout -b recovery-branch abc123

# Or cherry-pick onto current branch
git cherry-pick abc123

# Or reset to before the accident
git reset --hard HEAD@{1}
```

### ⭐ Q: Scenario: You rebased and lost commits. Recover them.

```bash
# Find the commit before rebase started
git reflog
# abc123 HEAD@{5}: commit: Feature complete (before rebase)
# ...
# def456 HEAD@{0}: rebase: <current state>

# Recover by resetting to before rebase
git reset --hard abc123

# Or create a backup branch
git branch backup-before-rebase abc123
```

### ⭐ Q: Scenario: Recover a deleted branch.

```bash
# 1. Find the branch tip in reflog
git reflog
# abc123 HEAD@{3}: commit: Last commit on deleted-branch

# 2. Recreate the branch
git branch recovered-branch abc123

# Or if you know the branch name
git reflog show deleted-branch
git branch deleted-branch <commit>
```

---

## Submodules vs Subtree

### 💡 Q: Git submodules vs subtree — when to use each?

**Git Submodules:**
```bash
# Add submodule
git submodule add https://github.com/org/lib.git libs/lib

# Clone repo with submodules
git clone --recurse-submodules <url>

# Update submodules
git submodule update --remote

# Initialize after clone
git submodule init
git submodule update
```

**Git Subtree:**
```bash
# Add subtree
git subtree add --prefix=libs/lib https://github.com/org/lib.git main --squash

# Pull updates
git subtree pull --prefix=libs/lib https://github.com/org/lib.git main --squash

# Push changes back to upstream
git subtree push --prefix=libs/lib https://github.com/org/lib.git feature-branch
```

| Aspect | Submodules | Subtree |
|--------|------------|---------|
| External reference | ✅ Pointer to commit | ❌ Full copy |
| Clone complexity | Requires `--recurse-submodules` | Just `git clone` |
| Learning curve | Steep | Moderate |
| Workflow | Update external dependency | Vendor/merge external code |
| History | Separate | Merged into main repo |
| Use when | Multiple projects share library | Vendoring dependencies, simpler workflow |

**Recommendation:** Subtree for most teams (simpler). Submodules when you need strict separation and versioning.

---

## Monorepo Techniques

### ⭐ Q: Sparse checkout and partial clone for large repos.

**Sparse Checkout** — Check out only specific directories (reduces working directory size):
```bash
# Enable sparse checkout
git sparse-checkout init --cone

# Set directories to check out
git sparse-checkout set apps/api apps/web

# Add more directories
git sparse-checkout add libs/shared

# List current sparse paths
git sparse-checkout list

# Disable (restore full checkout)
git sparse-checkout disable
```

**Partial Clone (Blobless)** — Download only commits and trees, fetch blobs on demand:
```bash
# Clone without file contents (download on checkout)
git clone --filter=blob:none <url>

# Clone with recent history only
git clone --depth=1 <url>              # Shallow clone (last commit)
git clone --depth=10 <url>             # Last 10 commits

# Convert shallow to full
git fetch --unshallow
```

| Technique | Reduces | Use When |
|-----------|---------|----------|
| Sparse checkout | Working directory | Work on specific services in monorepo |
| Partial clone | .git size | Large repo, don't need all files immediately |
| Shallow clone | History | CI builds, don't need history |

**Monorepo tools:** Git sparse-checkout (built-in), `git-filter-repo`, Google's `repo`, Facebook's `sapling` (hg-inspired Git UI).

### 🔥 Q: Monorepo vs polyrepo tradeoffs?

**Monorepo:**
- ✅ Atomic cross-project changes
- ✅ Shared tooling, dependencies
- ✅ Single source of truth
- ❌ Slower Git operations (mitigated by sparse-checkout)
- ❌ CI complexity (build only changed projects)

**Polyrepo:**
- ✅ Independent release cadences
- ✅ Simpler access control
- ✅ Faster per-repo operations
- ❌ Harder to coordinate breaking changes
- ❌ Dependency version hell

**Examples:** Google, Facebook, Microsoft (monorepo). Amazon, Netflix (polyrepo). Many use hybrid.

---

## Commit Signing

### ⭐ Q: Why and how to sign commits?

**Why:** Verify commit authenticity — prove *you* authored the commit, not an impostor using your email.

**GPG Signing:**
```bash
# Generate GPG key
gpg --full-generate-key

# List keys
gpg --list-secret-keys --keyid-format=long

# Configure Git
git config --global user.signingkey <KEY_ID>
git config --global commit.gpgsign true

# Sign commit manually
git commit -S -m "feat: add feature"

# Verify signatures
git log --show-signature
git verify-commit <commit>

# Add GPG key to GitHub/GitLab
gpg --armor --export <KEY_ID>
# Paste into GitHub Settings > SSH and GPG keys
```

**SSH Signing (Git 2.34+, simpler than GPG):**
```bash
# Configure Git to use SSH key
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub

# Sign commits
git config --global commit.gpgsign true

# Add SSH key as signing key on GitHub
# Settings > SSH and GPG keys > New SSH key (Signing Key)
```

**gitsign (Sigstore, keyless signing):**
```bash
# Install gitsign
brew install sigstore/tap/gitsign

# Configure
git config --global gpg.x509.program gitsign
git config --global gpg.format x509
git config --global commit.gpgsign true

# Commits are signed via OIDC (GitHub, Google, Microsoft login)
```

---

## Large Repos & Git LFS

### ⭐ Q: Git LFS for large files.

**Git LFS** (Large File Storage) — Store large files (binaries, videos, datasets) outside Git, replace with pointer.

```bash
# Install
brew install git-lfs
git lfs install

# Track large files
git lfs track "*.psd"
git lfs track "*.mp4"
git lfs track "datasets/**"

# Verify tracked patterns
cat .gitattributes

# Stage and commit
git add .gitattributes
git add large-file.psd
git commit -m "Add design files"

# Push to LFS remote
git push origin main

# Clone with LFS
git clone <url>
git lfs pull                   # Fetch LFS objects

# Migrate existing files to LFS
git lfs migrate import --include="*.zip" --everything
```

**LFS vs Git Submodule vs Subtree:**
| Tool | Purpose |
|------|---------|
| LFS | Large files (binaries, media) |
| Submodule | External Git repos |
| Subtree | Vendored code |

### 💡 Q: Large repo performance tips.

```bash
# 1. Use shallow clone for CI
git clone --depth=1 <url>

# 2. Enable filesystem monitor (faster git status)
git config core.fsmonitor true
git config core.untrackedcache true

# 3. Use commit graph (faster log, blame)
git commit-graph write --reachable

# 4. Garbage collection
git gc --aggressive

# 5. Sparse checkout (monorepo)
git sparse-checkout set apps/api

# 6. Partial clone (blobless)
git clone --filter=blob:none <url>

# 7. Use Git protocol v2 (faster fetch)
git config --global protocol.version 2
```

---

## Scenario-Based Questions

### 🔥 Q: Scenario: Accidentally committed to wrong branch. Move it to correct branch.

```bash
# Currently on main, should have been on feature/login
git log --oneline -1
# abc123 Add login feature (OOPS, this should be on feature/login)

# 1. Create/switch to correct branch (will include the commit)
git branch feature/login       # Create branch at current HEAD
git checkout feature/login

# 2. Reset main to before the commit
git checkout main
git reset --hard HEAD~1        # Remove commit from main

# Or one-liner:
git branch feature/login       # Capture commit
git reset --hard origin/main   # Reset main to remote
git checkout feature/login
```

### ⭐ Q: Scenario: Committed to main instead of creating a feature branch.

```bash
# You're on main with uncommitted work
git checkout -b feature/new-feature  # Creates branch, keeps changes

# Or if already committed:
git branch feature/new-feature       # Create branch at current HEAD
git reset --hard origin/main         # Reset main to remote
git checkout feature/new-feature     # Switch to feature branch
```

### ⭐ Q: Scenario: Split one large commit into multiple smaller commits.

```bash
# 1. Reset to before the large commit (keep changes)
git reset --soft HEAD~1

# 2. Interactively stage and commit parts
git add -p                     # Stage hunks interactively
git commit -m "feat: add user model"

git add src/api/
git commit -m "feat: add user API endpoints"

git add tests/
git commit -m "test: add user tests"

# Or use interactive rebase
git rebase -i HEAD~1
# Change 'pick' to 'edit' for the commit
# When Git stops:
git reset HEAD~1               # Uncommit but keep changes
# Stage and commit in parts
git rebase --continue
```

### ⭐ Q: Scenario: Remove a large file bloating the repo.

```bash
# 1. Find large files
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  awk '/^blob/ {print substr($0,6)}' | sort --numeric-sort --key=2 | tail -10

# 2. Remove using git filter-repo (RECOMMENDED)
pip install git-filter-repo
git filter-repo --path large-file.zip --invert-paths

# 3. Or BFG Repo Cleaner
bfg --delete-files large-file.zip

# 4. Force push
git push origin --force --all

# 5. Notify team to re-clone
```

### ⭐ Q: Scenario: Resolve a gnarly rebase conflict with many commits.

```bash
git rebase main
# CONFLICT in multiple commits

# Option 1: Abort and merge instead
git rebase --abort
git merge main

# Option 2: Use rerere (if enabled)
git config rerere.enabled true
# Resolve conflict once, Git remembers for subsequent commits

# Option 3: Resolve each commit
# Resolve conflict
git add <resolved-files>
git rebase --continue
# Repeat until done

# Option 4: Use theirs/ours strategy for specific files
git checkout --ours <file>     # Keep my version
git checkout --theirs <file>   # Take incoming version
git add <file>
git rebase --continue
```

### 🔥 Q: Scenario: Recover work after "oh no, I lost everything!"

```bash
# 1. Don't panic. Check reflog first
git reflog

# 2. Find your lost work (look for commit messages)
git reflog | grep "feature"
# abc123 HEAD@{10}: commit: Complete feature X

# 3. Recover
git checkout -b recovered abc123

# 4. If reflog is empty (rare), check stash
git stash list

# 5. If no reflog/stash, check .git/objects manually
git fsck --lost-found
# Dangling commits listed
git show <commit-hash>
```

### 🔥 Q: Scenario: Undo last commit but keep changes (not pushed yet).

```bash
# Undo commit, keep staged
git reset --soft HEAD~1

# Undo commit, unstage but keep in working dir
git reset --mixed HEAD~1

# Or amend the commit
git commit --amend -m "Updated message"
```

### 🔥 Q: You accidentally committed a secret (API key). How to remove it from history?

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

### 🔥 Q: Two developers made conflicting changes. How to resolve?

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

### 🔥 Q: Scenario: Undo pushed commits safely on a shared branch.

```bash
# NEVER use git reset on shared branches
# Use git revert instead (creates opposite commits)

git revert --no-commit HEAD~3..HEAD   # Revert last 3 commits
git commit -m "revert: undo last 3 commits due to regression"
git push origin main
```

### 🔥 Q: Scenario: Rebase and push conflict — "non-fast-forward" error.

```bash
git push origin feature
# error: failed to push some refs
# hint: Updates were rejected because the tip of your current branch is behind

# NEVER force push on shared branches (main, develop)
# For personal feature branch:

# Option 1: Pull with rebase
git pull --rebase origin feature

# Option 2: Force push (if you're sure it's your personal branch)
git push --force-with-lease origin feature   # Safer than --force

# Option 3: Create new branch, abandon old
git branch feature-v2
git push origin feature-v2
```

### ⭐ Q: Scenario: Two developers pushed to main simultaneously.

```bash
# Developer A pushed first
# Developer B tries to push:
git push origin main
# error: failed to push

# Developer B pulls changes
git pull --rebase origin main   # Rebase (cleaner) or just 'git pull' (merge)

# Resolve conflicts if any
git add <resolved-files>
git rebase --continue

# Push
git push origin main
```

### 🔥 Q: Scenario: Find which commit introduced a bug.

```bash
# Use git bisect (binary search)
git bisect start
git bisect bad                  # Current commit is broken
git bisect good v1.0.0          # Known good commit

# Git checks out middle commit
# Test it, then:
git bisect good                 # If this commit works
git bisect bad                  # If this commit is broken

# Git narrows down to the commit that introduced the bug
# Binary search: O(log n) commits to check

git bisect reset                # Exit bisect mode

# Automate bisect with a test script
git bisect start HEAD v1.0.0
git bisect run npm test         # Git automatically tests each commit
```

### ⭐ Q: Scenario: PR review requested changes. How to update cleanly?

```bash
# Option 1: Add fixup commits (simple, preserves review comments)
git add <changed-files>
git commit -m "fixup: address review comments"
git push origin feature

# Option 2: Amend last commit (if changes belong to it)
git add <changed-files>
git commit --amend --no-edit
git push --force-with-lease origin feature

# Option 3: Interactive rebase to squash fixups
git rebase -i origin/main
# Mark commits as 'fixup' to squash
git push --force-with-lease origin feature
```

### 💡 Q: Scenario: Work on multiple features simultaneously without context switching.

```bash
# Use git worktree (avoids stashing)
git worktree add ../project-feature-a feature/a
git worktree add ../project-feature-b feature/b

# Work in separate directories
cd ../project-feature-a         # Edit feature A
cd ../project-feature-b         # Edit feature B

# Both use same .git, different working directories
```

---

## Key Resources

- **Pro Git (book, free)** — https://git-scm.com/book
- **Oh Shit, Git!?!** — https://ohshitgit.com
- **Learn Git Branching** — https://learngitbranching.js.org (interactive)
- **Conventional Commits** — https://www.conventionalcommits.org
- **Atlassian Git Tutorials** — https://www.atlassian.com/git/tutorials
- **git-filter-repo** — https://github.com/newren/git-filter-repo (history rewriting)
- **BFG Repo Cleaner** — https://rtyley.github.io/bfg-repo-cleaner
- **Git LFS** — https://git-lfs.github.com
- **Sigstore gitsign** — https://github.com/sigstore/gitsign (keyless commit signing)
- **GitLab Flow** — https://docs.gitlab.com/ee/topics/gitlab_flow.html
- **Trunk-Based Development** — https://trunkbaseddevelopment.com
