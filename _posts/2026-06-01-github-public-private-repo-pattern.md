---
layout: post
title: "Keeping Your Code Public and Your Data Private with Two GitHub Repos"
date: 2026-06-01
categories: [git, workflow]
tags: [git, github, privacy, workflow, shell]
description: "A practical Git pattern for developers who build personal tools and want to share the code without exposing personal data."
---

# Keeping Your Code Public and Your Data Private with Two GitHub Repos

*A practical Git pattern for developers who build personal tools and want to share the code without exposing personal data.*

---

## The Problem

You built something — a habit tracker, a study planner, a budget tool, a journal app. The code is worth sharing. It shows real skills: a backend API, a frontend framework, maybe Docker or CI/CD. You want employers and collaborators to see it.

But the app also generates personal data. Daily logs. Notes. Goals. Mood entries. That data is yours — it's not part of the portfolio.

GitHub doesn't solve this for you. You can't make one branch private while keeping others public. Privacy is set at the repository level. So you need a different approach.

---

## The Solution — Two Repositories

```
myapp/              ← PUBLIC repo
├── src/            ← application source code
├── server.js       ← backend API
├── Dockerfile      ← deployment config
└── .gitignore      ← excludes data files ← this is the key

myapp-data/         ← PRIVATE repo
├── userdata.json   ← your personal data files
└── settings.json
```

The public repo contains everything someone needs to understand, run, and evaluate your project. The private repo contains only the files the app reads and writes at runtime — your actual data.

---

## Why This Works

Your app writes data to local files on your machine. Those files are listed in `.gitignore` in the public repo, so Git never sees them. A separate local folder is connected to the private repo. When you want to back up or sync your data, you copy the files there and push — to the private repo only.

The two repos never interact. The public repo has no knowledge of the private one.

---

## The Core Git Concept: .gitignore

This is the foundation of the whole pattern. `.gitignore` tells Git which files to never track, no matter what.

In your public repo's `.gitignore`:
```
data/userdata.json
data/settings.json
```

Once a file is in `.gitignore`, it cannot be accidentally committed — even if you run `git add .`. Git simply ignores it.

**Important:** if you ever accidentally committed a data file before adding it to `.gitignore`, you need one extra step to un-track it:

```bash
git rm --cached data/userdata.json
```

This removes the file from Git tracking without deleting it from your disk. Then add it to `.gitignore` and commit that change.

---

## Setting Up the Two-Repo Pattern

### Step 1 — Create the private repo

On GitHub: **+** → **New repository** → name it → check **Private** → **Create repository**.

### Step 2 — Create a local sync folder

Pick a permanent location (not `/tmp/` — that gets wiped on reboot):

```bash
mkdir -p ~/myapp-data
cd ~/myapp-data
git init
git remote add origin https://github.com/yourusername/myapp-data.git
```

Copy your data files in for the first time:
```bash
cp ~/myapp/data/userdata.json ~/myapp-data/
git add .
git commit -m "initial data"
git branch -M main
git push origin main
```

### Step 3 — Protect your public repo

In your public repo's `.gitignore`:
```
data/userdata.json
data/settings.json
```

Commit and push:
```bash
git add .gitignore
git commit -m "exclude data files from public repo"
git push origin main
```

### Step 4 — Automate with shell aliases

Add to `~/.bashrc` or `~/.zshrc`:

```bash
# Start your app — pulls latest code and restores your data
alias app-start='cd ~/myapp && git pull origin main && cp ~/myapp-data/userdata.json data/ 2>/dev/null && npm start'

# Save your data — pushes only to the private repo
alias app-save='cp ~/myapp/data/userdata.json ~/myapp-data/ && cd ~/myapp-data && git add . && git commit -m "data $(date +%Y-%m-%d)" && git push origin main'
```

Activate:
```bash
source ~/.bashrc
```

From now on: `app-start` to begin, `app-save` to back up. Two words.

---

## Syncing Between Two Machines

This pattern also solves the multi-machine problem. If you work on both a laptop and a cloud environment, the private repo acts as the sync point.

Before switching machines:
```bash
app-save
```

On the other machine:
```bash
app-start
```

Your data follows you. No manual file transfers.

---

## What Belongs Where

| Content | Public repo | Private repo |
|---------|-------------|--------------|
| Source code | ✓ | |
| Dockerfile / docker-compose | ✓ | |
| README and documentation | ✓ | |
| Runtime user data | | ✓ |
| Personal notes or logs | | ✓ |
| API keys or credentials | neither — use environment variables | neither |

---

## Key Git Commands You Should Understand

**`git remote add`**
Connects a local folder to a remote GitHub repo. You can have multiple remotes in different local folders. Each folder points to a different repo.

```bash
git remote add origin https://github.com/yourusername/repo.git
git remote -v   # verify both fetch and push URLs
```

**`git rm --cached`**
Removes a file from Git tracking without touching the file on disk. Use when you committed something you should not have.

```bash
git rm --cached path/to/file
```

**`git branch -M main`**
Renames the current branch to `main`. Needed when initializing a new repo locally before the first push, since Git may create a default branch name that does not match GitHub's expected `main`.

**`git cherry-pick`**
Copies one specific commit from anywhere in your Git history into your current branch. Useful when you made a change on the wrong branch and need just that one commit — not a full merge.

```bash
git log --oneline          # find the commit hash you want
git cherry-pick abc1234    # apply it to current branch
```

---

## Common Mistakes

### Mistake 1 — Using /tmp/ as your sync folder

`/tmp/` is cleared on reboot. Any git repo you initialize there disappears. Always use a permanent home directory location like `~/myapp-data/`.

### Mistake 2 — Pushing data to the public repo by accident

Check before you push:
```bash
git status        # see exactly what files are staged
git diff --cached # see the actual changes
```

If you already pushed sensitive data:
```bash
git rm --cached data/userdata.json
echo "data/userdata.json" >> .gitignore
git add .gitignore
git commit -m "remove data file from tracking"
git push origin main
```

Note: if the data was public even briefly, consider it exposed. Rotate any credentials. If it was truly sensitive, GitHub support can assist with history removal.

### Mistake 3 — Being on the wrong branch

Always check:
```bash
git branch
```

The branch with `*` is your current one. Committing to the wrong branch sends your changes somewhere unexpected.

### Mistake 4 — Confusing the two repos

When you have two repos, it is easy to run a push in the wrong folder. A clear naming convention helps — keep the public repo folder named after the project and the private one with a `-data` suffix.

---

## The Security Checklist

Before making any repo public, verify:

- [ ] No API keys or tokens in any file (use environment variables instead)
- [ ] No passwords or connection strings in source code
- [ ] Data files listed in `.gitignore`
- [ ] `git log` shows no commits containing personal data
- [ ] `git status` is clean before pushing

Run this to search your entire git history for a specific string:
```bash
git log -S "search_term" --all
```

If you find sensitive data in your history, removing it requires rewriting history — which is more involved. Prevention is much easier than cleanup.

---

## Summary

The pattern is simple:

1. One public repo for code — employers see it, the community can use it
2. One private repo for data — only you have access
3. `.gitignore` in the public repo prevents data files from ever crossing over
4. A local sync folder connected to the private repo handles backup and multi-machine sync
5. Two shell aliases make the daily workflow a single word

The separation is clean, the setup takes about fifteen minutes, and once it is running you never think about it again.

---

*This pattern applies to any personal tool that generates user data: habit trackers, study planners, journal apps, budget tools, health monitors. The principle is the same regardless of the tech stack.*
