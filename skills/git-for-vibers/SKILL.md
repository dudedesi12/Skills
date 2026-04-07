---
name: git-for-vibers
description: "Use this skill whenever the user mentions git, GitHub, commits, branches, push, pull, merge, revert, 'save my code', 'go back', 'undo', 'I messed up', version control, PR, pull request, 'what changed', deploy from branch, or any code versioning concern. Also trigger when user is about to do something risky ('let me try something') to suggest branching first. Even if they don't say 'git' — any intent around saving, versioning, or recovering code triggers this skill."
---

# Git for Vibers

Git saves snapshots of your code so you can go back in time, try experiments safely, and never lose work. You do not need to understand git internals. Just tell me what you want in plain English.

## The Only Mental Model You Need

Think of git like a video game save system:
- **Commit** = save your game
- **Branch** = start an alternate timeline
- **Push** = upload your save to the cloud
- **Pull** = download saves from the cloud
- **Merge** = combine two timelines into one
- **Revert** = load an old save

## Intent-to-Command Quick Reference

### "Save my work"

```bash
git add .
git commit -m "feat(auth): add login page with email validation"
git push
```

Always write a real commit message. Never use "update" or "fix" or "stuff". See the commit message section below.

### "I messed up, go back"

**Undo changes I have NOT saved (committed) yet:**
```bash
git stash
```
This puts your changes in a drawer. Get them back with `git stash pop`.

**Throw away ALL unsaved changes (cannot undo this):**
```bash
git checkout -- .
```

**Undo the LAST commit but keep the files changed:**
```bash
git reset --soft HEAD~1
```

**Undo the last commit completely, erase everything:**
```bash
git reset --hard HEAD~1
```

**Undo a commit that is already pushed (safe for shared repos):**
```bash
git revert HEAD
git push
```

### "Let me try something experimental"

Always branch before experimenting. Never experiment on main.

```bash
git checkout -b experiment/new-pricing-page
```

Work freely. Commit as much as you want. Nothing touches main.

### "That experiment worked, keep it"

```bash
git checkout main
git merge experiment/new-pricing-page
git push
git branch -d experiment/new-pricing-page
```

### "That experiment failed, trash it"

```bash
git checkout main
git branch -D experiment/new-pricing-page
```

### "What changed?"

**See which files changed:**
```bash
git diff --stat
```

**See the last 10 saves:**
```bash
git log --oneline -10
```

**See what changed in a specific file:**
```bash
git log --oneline -10 -- src/app/page.tsx
```

### "Show me yesterday's version"

```bash
git log --oneline --after="yesterday" --before="today"
```

**Restore a specific file from a past commit:**
```bash
git log --oneline -20 -- path/to/file.tsx
# find the commit hash (the short code on the left)
git checkout abc1234 -- path/to/file.tsx
```

### "I deleted something important"

Git remembers everything, even things you think you deleted.

```bash
git reflog
# find the commit hash where the file existed
git checkout abc1234 -- path/to/deleted-file.tsx
```

### "Someone else's code broke mine"

Use bisect to find which commit broke things:

```bash
git bisect start
git bisect bad                  # current version is broken
git bisect good abc1234         # this old commit worked fine
# git checks out a middle commit — test it, then:
git bisect good                 # if this commit works
git bisect bad                  # if this commit is broken
# repeat until git tells you the exact breaking commit
git bisect reset                # done, go back to normal
```

## Commit Messages That Actually Help

Every commit message follows this format:

```
type(scope): short description of what you did
```

**Types:**
| Type | When to use | Example |
|------|-------------|---------|
| feat | New feature or page | `feat(dashboard): add revenue chart` |
| fix | Bug fix | `fix(auth): prevent double login redirect` |
| chore | Dependencies, config, cleanup | `chore(deps): upgrade next to 15.1` |
| docs | Documentation | `docs(readme): add deploy instructions` |
| style | UI/CSS changes only | `style(landing): increase hero font size` |
| refactor | Code restructure, no behavior change | `refactor(api): extract validation helpers` |

**Scope** is the area of the app you changed: auth, dashboard, api, landing, db, etc.

**How to pick the right type:**
1. Did you add something new users can see or use? `feat`
2. Did you fix something that was broken? `fix`
3. Did you just move code around or rename things? `refactor`
4. Did you only change how something looks? `style`
5. Did you update packages or config files? `chore`
6. Did you update comments or documentation? `docs`

**Auto-generating commit messages from changes:**

Look at what files changed and write the message based on that:

```bash
git diff --stat
```

- Changed files in `app/auth/` -> scope is `auth`
- Added a new page -> type is `feat`
- Fixed a broken button -> type is `fix`
- Only changed `.css` or Tailwind classes -> type is `style`

**Rejected commit messages** (never accept these):
- "update"
- "fix"
- "changes"
- "stuff"
- "wip"
- "asdf"
- "test"
- Any single word without a type and scope

## Branch Strategy for Solo Vibers

You only need four kinds of branches:

```
main              Always works. This is what Vercel deploys.
  |
  +-- dev         Your daily work branch.
  |
  +-- experiment/ Wild ideas. Delete if they fail.
  |     experiment/ai-chat
  |     experiment/dark-mode
  |
  +-- hotfix/     Urgent fixes to production.
        hotfix/login-crash
```

### Daily workflow

```bash
# Start your day
git checkout dev
git pull

# Work, commit often
git add .
git commit -m "feat(dashboard): add weekly stats view"
git push

# Ready to deploy? Merge dev into main
git checkout main
git pull
git merge dev
git push
# Vercel auto-deploys main

# Back to work
git checkout dev
```

### Starting an experiment

```bash
git checkout main
git checkout -b experiment/ai-chat
# build the feature
git add .
git commit -m "feat(chat): add AI chat widget prototype"
git push -u origin experiment/ai-chat
```

### Hotfix workflow (production is broken)

```bash
git checkout main
git checkout -b hotfix/login-crash
# fix the bug
git add .
git commit -m "fix(auth): handle null user in session check"
git checkout main
git merge hotfix/login-crash
git push
git branch -d hotfix/login-crash
# also merge the fix into dev so you have it
git checkout dev
git merge main
git push
```

## Pull Request Workflow

Even working solo, PRs create a paper trail and let Vercel preview deployments work.

### Creating a PR

```bash
# Push your branch
git push -u origin experiment/ai-chat

# Create PR via GitHub CLI
gh pr create --title "feat(chat): add AI chat widget" --body "## What
- Added real-time chat widget using Supabase realtime
- Styled with Tailwind, responsive on mobile

## Why
Users requested in-app support chat

## Test
- Open /dashboard, chat widget appears bottom-right
- Send message, appears in Supabase table
- Responsive on mobile viewport"
```

### Self-Review Checklist

Before merging any PR, check:

- [ ] `npm run build` passes with no errors
- [ ] No `console.log` left in production code
- [ ] No hardcoded secrets, API keys, or passwords
- [ ] New environment variables added to `.env.example`
- [ ] Page works on mobile (check at 375px width)
- [ ] No TypeScript errors (`npx tsc --noEmit`)
- [ ] Commit messages follow `type(scope): description` format

### Merging a PR

```bash
# Merge via CLI
gh pr merge --squash --delete-branch

# Or merge on GitHub and then locally:
git checkout main
git pull
```

Use `--squash` to combine all commits into one clean commit on main.

## Recovery Patterns

### "I committed secrets / API keys"

This is urgent. Exposed keys can be stolen in seconds.

**Step 1: Rotate the secret immediately.** Go to your provider dashboard (Supabase, Stripe, etc.) and generate a new key. Update your `.env.local`.

**Step 2: Remove from git history:**

```bash
# Install BFG (faster than git filter-branch)
brew install bfg  # or download from https://rsa.pub/bfg

# Remove the file from all history
bfg --delete-files .env.local
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force
```

**Step 3: Add to .gitignore** so it never happens again:

```gitignore
# .gitignore
.env
.env.local
.env.production
.env*.local
```

### "My repo is completely messed up" (nuclear option)

When nothing works and you just want to start fresh:

```bash
# Save your current code somewhere safe first
cp -r my-project my-project-backup

# Re-clone from GitHub
cd ..
rm -rf my-project
git clone https://github.com/yourname/my-project.git
cd my-project
npm install
```

If you had uncommitted work in the backup, copy those files over manually.

### "I committed to the wrong branch"

```bash
# Save the commit hash
git log --oneline -1
# Example output: abc1234 feat(auth): add login page

# Undo the commit on this branch (keep the changes)
git reset --soft HEAD~1

# Switch to the right branch and re-commit
git stash
git checkout correct-branch
git stash pop
git add .
git commit -m "feat(auth): add login page"
```

### "I have merge conflicts"

Merge conflicts happen when two branches changed the same line. Git marks the conflicts like this:

```
<<<<<<< HEAD
your version of the code
=======
the other version of the code
>>>>>>> branch-name
```

**How to fix:** Pick the version you want (or combine both), delete the `<<<<<<<`, `=======`, and `>>>>>>>` markers, then:

```bash
git add .
git commit -m "fix: resolve merge conflicts in auth module"
```

## Essential .gitignore for Next.js + Supabase

```gitignore
# dependencies
node_modules/
.pnp
.pnp.js

# builds
.next/
out/
build/

# env files (NEVER commit these)
.env
.env.local
.env.production
.env.development.local
.env.test.local
.env.production.local
.env*.local

# debug
npm-debug.log*

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Vercel
.vercel

# Supabase local
supabase/.temp/
```

## Dangerous Commands Cheat Sheet

These commands can lose your work. Use with caution:

| Command | What it does | Danger level |
|---------|-------------|--------------|
| `git reset --hard` | Erases all uncommitted changes | HIGH |
| `git push --force` | Overwrites remote history | HIGH |
| `git branch -D` | Deletes branch even if not merged | MEDIUM |
| `git checkout -- .` | Erases all uncommitted file changes | HIGH |
| `git clean -fd` | Deletes all untracked files and dirs | HIGH |

**Rule: Always commit or stash before running any of these.**

## Quick Setup for a New Project

```bash
# Initialize git in your project
git init
git add .
git commit -m "chore: initial project setup with Next.js and Supabase"

# Connect to GitHub
gh repo create my-project --public --source=. --push

# Set up branches
git checkout -b dev
git push -u origin dev
git checkout main
```

Your Vercel project auto-deploys `main`. Preview deployments happen on PRs.
