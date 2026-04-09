# Git Commands for Freshers — The Complete Cheatsheet

**The 20 commands that do 95% of the work. Learn these, and you'll survive any team.**

If you're a fresher joining your first dev team, nobody will sit with you and explain git. They'll assume you know it. And then, on day two, your tech lead will say something like:

> "Push your changes to a feature branch, rebase onto main, and open a PR. Don't force push."

If that sentence made you panic, this doc is for you.

Git has hundreds of commands. Most of them you'll never touch. This is the **exact set of commands I've used in 10 years of shipping code** — organized by the order you'll learn them in a real job.

Read it once. Bookmark it. Come back when you're stuck.

---

## Mental Model First — What is Git, Really?

Before the commands, understand this:

Git tracks your code in **4 places**:

1. **Working Directory** — the files on your laptop right now
2. **Staging Area** (Index) — files you've marked as "ready to commit"
3. **Local Repository** — commits saved on your machine
4. **Remote Repository** — commits saved on GitHub/GitLab/Bitbucket

Every git command moves code between these 4 places. That's it.

<p align="center">
  <img src="images/git-commands-for-freshers/workflow-animated.svg" alt="Git's 4 buckets — animated" width="100%"/>
</p>

> **In short:** `add` moves files to staging, `commit` moves staging to local repo, `push` moves local to remote, `pull` brings remote back to local.

Once you see git as "moving files between 4 buckets", nothing confuses you.

---

## Setup — Run These Once, Forget Forever

Before you commit anything, tell git who you are. Without this, your commits show up as "unknown".

```bash
# Your identity — shows up on every commit you make
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Set VS Code as your default editor (optional but saves pain)
git config --global core.editor "code --wait"

# Set default branch name to 'main' for new repos
git config --global init.defaultBranch main
```

**Why `--global`?** It applies to every repo on your machine. Without it, you'd set this per project.

---

## The 5 Commands You'll Use Every Single Day

These 5 commands are 80% of git usage. Master these first.

### 1. `git clone` — Copy a repo to your machine

```bash
git clone https://github.com/company/project.git
cd project
```

This downloads the entire repo — all branches, all history, all files. You'll run this exactly once per project.

**Common mistake:** running `git clone` inside another git repo. Always cd out first.

### 2. `git status` — What changed?

```bash
git status
```

Your most-used command. Shows:
- Which files you've modified
- Which files are staged (ready to commit)
- Which branch you're on

**Run this before every commit.** Every single time. No exceptions.

### 3. `git add` — Stage your changes

```bash
# Stage one specific file
git add src/login.js

# Stage everything in current directory
git add .

# Stage only parts of a file (interactive)
git add -p
```

Staging = telling git "include these in the next commit".

**Pro tip:** prefer `git add <file>` over `git add .`. The dot can accidentally include secrets, debug files, or node_modules.

### 4. `git commit` — Save a snapshot

```bash
# Commit with a message
git commit -m "fix login redirect bug"

# Commit everything tracked (skip staging)
git commit -am "fix login redirect bug"
```

A commit is a permanent snapshot. Your code at this exact moment is saved forever in git history.

**Commit message style:** short, imperative mood. "fix bug" not "fixed a bug" or "fixing bugs".

### 5. `git push` — Send commits to remote

```bash
# Push to the current branch on origin
git push

# First push of a new branch
git push -u origin feature/login-fix
```

Push uploads your local commits to GitHub. Until you push, your work only exists on your laptop.

**The `-u` flag** sets up "tracking" — after that first push, you can just run `git push` without arguments.

---

## The Next 5 — When You Work With a Team

Once you're collaborating, these become critical.

### 6. `git pull` — Get the latest from remote

```bash
git pull
```

Pull = fetch + merge. Downloads new commits from GitHub and merges them into your current branch.

**Golden rule:** always `git pull` before starting new work. Otherwise you'll commit on top of stale code and hit merge conflicts.

### 7. `git branch` — See and manage branches

```bash
# List all local branches
git branch

# Create a new branch (doesn't switch to it)
git branch feature/dark-mode

# Delete a branch (only if merged)
git branch -d feature/old-work

# Force delete (dangerous — unmerged changes lost)
git branch -D feature/abandoned
```

A branch is just a movable pointer to a commit. Branches are cheap — create one for every feature, every bugfix, every experiment.

<p align="center">
  <img src="images/git-commands-for-freshers/branch-merge-animated.svg" alt="Branching and merging — animated" width="100%"/>
</p>

### 8. `git checkout` — Switch branches

```bash
# Switch to an existing branch
git checkout main

# Create AND switch in one command
git checkout -b feature/search-filter
```

**Note:** in newer git (2.23+), `git switch` replaces checkout for branches, and `git restore` replaces it for files. Both still work.

```bash
# Modern equivalent
git switch main
git switch -c feature/search-filter
```

### 9. `git merge` — Combine branches

```bash
# While on main, merge feature branch into main
git checkout main
git pull
git merge feature/dark-mode
```

Merge takes the changes from one branch and applies them to another. This is how your code gets shipped.

**Merge conflicts** happen when two branches changed the same lines. Git pauses and asks you to pick which version wins.

### 10. `git log` — See commit history

```bash
# Basic log
git log

# Pretty one-line view (much more useful)
git log --oneline

# Graph view showing branches
git log --oneline --graph --all

# Show changes for each commit
git log -p

# Last 5 commits
git log -5
```

**Pro tip:** alias this. Add to `.gitconfig`:
```
[alias]
    lg = log --oneline --graph --all --decorate
```

Now `git lg` shows the entire branch structure.

---

## The Survival Kit — When Things Go Wrong

Every fresher hits these moments. These commands save you.

### 11. `git diff` — See what you changed

```bash
# Unstaged changes (what you've modified but not added)
git diff

# Staged changes (what you've added but not committed)
git diff --staged

# Compare two branches
git diff main feature/login

# Compare two commits
git diff abc123 def456
```

Run this before every commit. Makes sure you're committing what you think you're committing.

### 12. `git stash` — Save work for later

```bash
# Save current changes and clean the working directory
git stash

# Save with a message
git stash push -m "wip on login bug"

# List all stashes
git stash list

# Bring back the most recent stash
git stash pop

# Bring back a specific stash
git stash apply stash@{1}
```

**When to use:** your tech lead says "quick — jump to main and hotfix prod". You have uncommitted work you don't want to lose. Stash it, fix prod, come back, pop the stash.

<p align="center">
  <img src="images/git-commands-for-freshers/stash-animated.svg" alt="Git stash flow — animated" width="100%"/>
</p>

### 13. `git reset` — Undo commits

This is the dangerous one. Three modes:

```bash
# Soft: undo commit, keep changes staged
git reset --soft HEAD~1

# Mixed (default): undo commit, keep changes unstaged
git reset HEAD~1

# Hard: undo commit, DELETE changes forever
git reset --hard HEAD~1
```

**`HEAD~1`** means "one commit back". `HEAD~3` means "three commits back".

**Never `--hard` reset work you haven't committed.** It's gone. No recovery.

### 14. `git revert` — Undo a commit safely

```bash
git revert abc123
```

Unlike reset, revert creates a **new commit** that undoes the changes. History stays intact. **This is the safe way to undo code that's already been pushed.**

Use `revert` on shared branches. Use `reset` only on your local work.

### 15. `git restore` — Throw away local changes

```bash
# Discard unstaged changes to a file
git restore src/login.js

# Discard ALL unstaged changes (careful!)
git restore .

# Unstage a file (keep the changes)
git restore --staged src/login.js
```

This is the modern replacement for `git checkout -- file` and `git reset HEAD file`.

---

## The Pro Commands — Level Up

Once you're comfortable with the basics, these make you look like a senior.

### 16. `git rebase` — Rewrite history

```bash
# Rebase your feature branch onto latest main
git checkout feature/login
git rebase main

# Interactive rebase — squash, reorder, edit commits
git rebase -i HEAD~5
```

Rebase takes your commits and replays them on top of a new base. The result: a **linear history** without merge commits.

**The golden rule of rebase:** never rebase commits that have been pushed to a shared branch. Rebasing rewrites history, which breaks everyone else's copy.

**Interactive rebase (`-i`)** lets you:
- **squash** — combine multiple commits into one
- **reword** — edit commit messages
- **drop** — delete commits
- **reorder** — change commit order

<p align="center">
  <img src="images/git-commands-for-freshers/rebase-vs-merge-animated.svg" alt="Merge vs Rebase — animated" width="100%"/>
</p>

### 17. `git fetch` — Download without merging

```bash
# Get latest from remote, but don't merge
git fetch

# Fetch a specific remote
git fetch origin
```

Fetch is the "safe pull". It downloads new commits from GitHub but doesn't touch your working directory. You can then inspect what changed before merging.

### 18. `git cherry-pick` — Copy a specific commit

```bash
git cherry-pick abc123
```

Takes one commit from any branch and applies it to your current branch. Useful when you need ONE fix from another branch without merging everything.

### 19. `git tag` — Mark a release

```bash
# Create a tag
git tag v1.2.0

# Create an annotated tag (preferred — has message + author)
git tag -a v1.2.0 -m "Release 1.2.0"

# Push tags to remote (not automatic!)
git push --tags
```

Tags mark important commits — usually releases. Unlike branches, tags don't move.

### 20. `git reflog` — Your safety net

```bash
git reflog
```

**This is the command that saves your career.**

Reflog is git's secret log of EVERY change to HEAD — including resets, rebases, and deleted branches. Even if you `--hard` reset and lose work, you can find it in reflog and recover it.

```bash
# Find the lost commit in reflog
git reflog

# Bring it back
git checkout abc123
# or
git reset --hard abc123
```

Reflog entries live for 90 days by default. If you've lost work in git, check reflog first. Most of the time, it's still there.

<p align="center">
  <img src="images/git-commands-for-freshers/reflog-safety-animated.svg" alt="Git reflog safety net — animated" width="100%"/>
</p>

---

## The Team Workflow — Putting It All Together

This is the workflow you'll follow 99% of the time at your first job:

![Team workflow](images/git-commands-for-freshers/team-workflow.png)

> **Tip:** the animated workflow diagram above shows the same flow as a live process — each command moves code one step closer to production.

```bash
# 1. Start from latest main
git checkout main
git pull

# 2. Create a feature branch
git checkout -b feature/user-profile

# 3. Make your changes, then check what changed
git status
git diff

# 4. Stage and commit
git add src/profile.js src/profile.css
git commit -m "add user profile page"

# 5. Push to remote (first time: use -u)
git push -u origin feature/user-profile

# 6. Open a Pull Request on GitHub

# 7. If main moved forward while your PR was open:
git fetch
git rebase origin/main
git push --force-with-lease  # safer than --force

# 8. After PR is merged, clean up
git checkout main
git pull
git branch -d feature/user-profile
```

That's it. That's the workflow. Memorize this and you're 90% of the way there.

---

## Common Mistakes Freshers Make

### Mistake 1: Committing directly to main
**Never.** Always work on a feature branch. Main is sacred.

### Mistake 2: `git push --force` on a shared branch
Force push rewrites history. If someone else has pulled that branch, their local copy becomes invalid. Use `--force-with-lease` instead — it fails safely if someone else has pushed.

### Mistake 3: Committing secrets
API keys, passwords, `.env` files — once committed, they live in history forever. Use `.gitignore` from day one. Add it to your repo BEFORE your first commit.

### Mistake 4: Huge commits
A commit should be one logical change. "fix 5 bugs and add a feature" is wrong. Five commits is right. Small commits are easier to review, easier to revert, easier to understand.

### Mistake 5: Vague commit messages
`"fix"`, `"update"`, `"wip"` — useless. Six months later, nobody knows what you did.

Good messages:
- `fix null pointer in login redirect when cookie expired`
- `add pagination to search results API`
- `refactor auth middleware to use async/await`

---

## The .gitignore File — Non-Negotiable

Before your first commit, create a `.gitignore` at the repo root. Common entries:

```
# Dependencies
node_modules/
vendor/

# Environment
.env
.env.local
*.key

# IDE
.vscode/
.idea/
*.swp

# Build output
dist/
build/
*.log

# OS
.DS_Store
Thumbs.db
```

GitHub maintains a list of templates at [github.com/github/gitignore](https://github.com/github/gitignore). Start from one of those.

---

## Quick Reference — The Cheatsheet

| Command | What It Does |
|---|---|
| `git clone <url>` | Copy a repo to your machine |
| `git status` | What's changed? |
| `git add <file>` | Stage a file for commit |
| `git commit -m "msg"` | Save a snapshot |
| `git push` | Upload commits to remote |
| `git pull` | Download + merge from remote |
| `git branch` | List branches |
| `git checkout -b <name>` | Create + switch to new branch |
| `git merge <branch>` | Combine a branch into current |
| `git log --oneline --graph` | See commit history |
| `git diff` | See unstaged changes |
| `git stash` | Save work temporarily |
| `git reset --soft HEAD~1` | Undo last commit, keep changes |
| `git revert <sha>` | Safely undo a pushed commit |
| `git restore <file>` | Discard local changes |
| `git rebase main` | Replay commits on top of main |
| `git fetch` | Download without merging |
| `git cherry-pick <sha>` | Copy one commit to current branch |
| `git tag v1.0.0` | Mark a release |
| `git reflog` | Recover lost commits |

---

## What to Learn Next

Once you're comfortable with these 20 commands:

1. **Understand the git object model** — blobs, trees, commits, refs. Once you see that git is a content-addressed filesystem, everything clicks.
2. **Learn interactive rebase deeply** — `rebase -i` is the most powerful git command. Learn squash, fixup, edit, reword.
3. **Set up a good `.gitconfig`** — aliases, colors, diff tools. Your own customized setup makes git 2x faster.
4. **Practice merge conflict resolution** — create intentional conflicts in a test repo and practice resolving them. It's a skill that only comes with reps.

---

## One Final Rule

**When in doubt, `git status`.**

Git gives you hints. Status will usually tell you exactly what to do next. Before you run any scary command, check status. Before you commit, check status. Before you push, check status.

That one habit will save you from 80% of the mistakes freshers make.

---

## References

- [Pro Git Book (free)](https://git-scm.com/book/en/v2) — the official, comprehensive git guide
- [Git Documentation](https://git-scm.com/docs)
- [GitHub Flow](https://docs.github.com/en/get-started/quickstart/github-flow)
- [Oh Shit, Git!?!](https://ohshitgit.com/) — quick recipes for when you mess up
- [Learn Git Branching](https://learngitbranching.js.org/) — interactive visual git tutorial
