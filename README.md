# Part 1: Inside the `.git` Directory (Internal Architecture)

Every Git repository keeps its database and configuration in the hidden `.git/` directory.

### Key Files and Directories

| Path | Purpose | How to Inspect |
| :--- | :--- | :--- |
| `.git/config` | Repository-specific settings, remotes, branches | `cat .git/config` or `git config -l --local` |
| `.git/HEAD` | Points to the currently checked-out branch / commit ref | `cat .git/HEAD` |
| `.git/refs/heads/` | Local branch pointers (each file contains a 40-char SHA) | `ls -la .git/refs/heads/` |
| `.git/refs/remotes/` | Tracking pointers for remote branches (e.g. `origin/main`) | `tree .git/refs/remotes/` |
| `.git/refs/tags/` | Tag pointers | `ls .git/refs/tags/` |
| `.git/index` | Binary staging area (the index / cache of staged changes) | `git ls-files --stage` |
| `.git/objects/` | Content-addressable object store (blobs, trees, commits, tags) | `git cat-file -p <hash>` |
| `.git/hooks/` | Client/server hook scripts triggered on commit, push, etc. | `ls .git/hooks/` |
| `.git/info/exclude` | Local gitignore file not shared with the team | `cat .git/info/exclude` |
| `.git/logs/` | Reflog logs (history of where `HEAD` and branches pointed) | `git reflog` or `tail .git/logs/HEAD` |
| `.git/FETCH_HEAD` | Stores the state of branches fetched from the last `git fetch` | `cat .git/FETCH_HEAD` |
| `.git/ORIG_HEAD` | Backup pointer before dangerous operations (rebase, merge, reset) | `cat .git/ORIG_HEAD` |

### Low-Level ("Plumbing") Inspection Commands

```bash
# Verify the root .git directory
git rev-parse --git-dir

# View contents of any git object (commit, tree, blob)
git cat-file -p <commit-or-blob-hash>

# Check the type of a git object (commit, tree, blob, tag)
git cat-file -t <hash>

# View everything currently in the staging index
git ls-files --stage

# Inspect untracked, modified, and ignored files via ls-files
git ls-files -mo --exclude-standard

# Repository size and object counts
git count-objects -vH

# Integrity check (check for corrupted or dangling objects)
git fsck --full --lost-found
```

---

# Part 2: "Check All The Things" (Diagnostic Superpowers)

These are the most useful commands for seeing the exact state of branches, remotes, uncommitted changes, and commit differences.

### 1. Status & Dirty Working Tree
```bash
# Short status with branch & tracking info (ahead/behind counts)
git status -sb

# Check untracked files ignored by .gitignore
git status --ignored

# Check diff of unstaged changes (working tree vs index)
git diff

# Check diff of staged changes (index vs HEAD)
git diff --staged    # or: git diff --cached

# Check summary of file changes without full diff
git diff --stat
```

### 2. Remotes & Remote Tracking
```bash
# List all configured remotes with URLs
git remote -v

# Detailed inspection of a remote (active branches, stale tracking branches)
git remote show origin
git remote show upstream

# Get specific URLs
git remote get-url origin
git remote get-url upstream

# Test connection to remote
git ls-remote origin
git ls-remote upstream
```

### 3. Branches & Upstream Tracking
```bash
# List local branches with commit SHA, commit subject, and tracking status [ahead/behind]
git branch -vv

# List ALL branches (local + remote)
git branch -a -vv

# See which branches contain a specific commit
git branch -a --contains <commit-hash>

# See which branches are already merged into current branch
git branch --merged

# See which branches are NOT merged into current branch
git branch --no-merged
```

### 4. Comparing Across Branches & Remotes
```bash
# Fetch latest references without merging
git fetch --all --prune

# How many commits are you ahead/behind upstream?
git rev-list --left-right --count HEAD...upstream/main
# Output: <commits_ahead>   <commits_behind>

# See the commits that are in upstream/main but NOT in your local branch
git log HEAD..upstream/main --oneline

# See the commits that are in your local branch but NOT pushed to origin
git log origin/main..HEAD --oneline

# View diff between current branch and upstream/main
git diff HEAD...upstream/main
```

---

# Part 3: Multi-Repo Management Scripts (For `/Users/shiva/all`)

Since you manage multiple microservices in one parent folder, here are battle-tested one-liners:

### A. Quick Dashboard: Branch, Dirty State & Ahead/Behind Status
```bash
for d in */.git; do
  repo=$(dirname "$d")
  [[ "$repo" == *-x ]] && continue
  branch=$(git -C "$repo" rev-parse --abbrev-ref HEAD 2>/dev/null)
  dirty=$(git -C "$repo" status --porcelain 2>/dev/null | wc -l | tr -d ' ')
  
  printf "%-28s | Branch: %-15s | Modified: %2s files\n" "$repo" "$branch" "$dirty"
done
```

### B. Deep Check: Local vs `origin` vs `upstream` Tracking
```bash
for d in */.git; do
  repo=$(dirname "$d")
  [[ "$repo" == *-x ]] && continue
  echo "============================================================"
  echo "📁 $repo"
  echo "------------------------------------------------------------"
  git -C "$repo" branch -vv
  echo ""
done
```

### C. Fetch All Remotes Across All Repos
```bash
for d in */.git; do
  repo=$(dirname "$d")
  [[ "$repo" == *-x ]] && continue
  echo "Fetching $repo..."
  git -C "$repo" fetch --all --prune
done
```

---

# Part 4: Fork & Upstream Workflow Cheatsheet

Since your repositories follow the `origin` (personal fork) and `upstream` (Sunbird-ALL) pattern:

### 1. Initial Remote Setup
```bash
# Add upstream remote if missing
git remote add upstream https://github.com/Sunbird-ALL/<repo-name>.git

# Rename or change URL if needed
git remote set-url origin <new-url>
git remote set-url upstream <new-url>
```

### 2. Syncing Your Fork with Upstream
```bash
# Step 1: Fetch updates from upstream
git fetch upstream

# Step 2: Switch to main (or master)
git switch main

# Step 3: Rebase (clean linear history) OR merge upstream
git merge upstream/main
# OR: git rebase upstream/main

# Step 4: Update your personal fork on GitHub
git push origin main
```

---

# Part 5: Everyday Commands Reference

### Switching & Branching
```bash
# Create and switch to new branch (modern syntax)
git switch -c feature/my-feature

# Switch back to existing branch
git switch main

# Switch to previous branch
git switch -
```

### Commits & Staging
```bash
# Stage changes interactively (chunk by chunk)
git add -p

# Amend previous commit without changing commit message
git commit --amend --no-edit

# Amend previous commit and change message
git commit --amend -m "Updated commit message"
```

### Log & History Visuals
```bash
# The ultimate clean graph log
git log --graph --oneline --decorate --all -n 20

# Search commit history for changes containing a keyword
git log -S "keyword" --source --all

# View log of a single file with diffs
git log -p -n 5 <path/to/file>

# Who changed which line and when
git blame -w -C <path/to/file>
```

### Stash
```bash
# Stash including untracked files
git stash -u -m "WIP before upstream pull"

# View all stashes
git stash list

# View diff inside a stash
git stash show -p stash@{0}

# Pop latest stash and keep index
git stash pop
```

### Undo, Safety Net & Recovery
```bash
# Discard unstaged changes to a file
git restore <file>

# Unstage a file (keep local modifications)
git restore --staged <file>

# Reflog: view ALL past actions (even deleted commits/branches)
git reflog

# Recover to any previous state seen in reflog
git reset --hard HEAD@{1}

# Undo the last commit but keep changes in working directory
git reset --soft HEAD~1
```
