# Branch Strategy for Solo Vibe Coders

A simple branching system that keeps your production app safe while you experiment freely.

---

## The Four Branch Types

```
main            Your live app. Vercel deploys this. Always works.
dev             Your daily work. Commit here all day.
experiment/*    Wild ideas. Keep or trash, main stays safe.
hotfix/*        Emergency fixes when production breaks.
```

## Visual Flow: Daily Work

```
main:  ────●────────────────────────●────────────── (always deployable)
            \                      / merge
dev:         ●────●────●────●────●
              c1   c2   c3   c4  (your daily commits)
```

**How it works:**
1. You work on `dev` all day, committing often.
2. When you are ready to deploy, merge `dev` into `main`.
3. Vercel sees the push to `main` and deploys automatically.

```bash
# Morning: start working
git checkout dev
git pull

# During the day: commit often
git add .
git commit -m "feat(dashboard): add chart component"
git push

# Ready to deploy
git checkout main
git pull
git merge dev
git push
# Vercel deploys automatically

# Back to work
git checkout dev
```

## Visual Flow: Experiments

```
main:         ────●──────────────●──────────────── 
                   \            / merge (it worked!)
experiment/ai:      ●────●────●
                     c1   c2   c3

main:         ────●──────────────────────────────── 
                   \            
experiment/3d:      ●────●────X  (trashed, main untouched)
                     c1   c2
```

**Experiments that work get merged. Experiments that fail get deleted. Main never breaks.**

```bash
# Start an experiment
git checkout main
git checkout -b experiment/ai-chat-widget

# Build the experiment, commit as you go
git add .
git commit -m "feat(chat): add basic chat UI"
git push -u origin experiment/ai-chat-widget

# OPTION A: It worked! Merge it.
git checkout main
git merge experiment/ai-chat-widget
git push
git branch -d experiment/ai-chat-widget
git push origin --delete experiment/ai-chat-widget

# OPTION B: It failed. Delete it.
git checkout main
git branch -D experiment/ai-chat-widget
git push origin --delete experiment/ai-chat-widget
```

## Visual Flow: Hotfix

```
main:    ────●──────●────●─────────── 
              \    / merge hotfix
hotfix/x:      ●──
               fix

dev:     ────●──────────●──────────── 
                       / merge main (get the fix)
```

**When production breaks, fix it directly from main so the fix deploys immediately.**

```bash
# Production is broken!
git checkout main
git pull
git checkout -b hotfix/login-crash

# Fix the bug
git add .
git commit -m "fix(auth): handle null session in middleware"

# Merge fix into main and deploy
git checkout main
git merge hotfix/login-crash
git push
# Vercel deploys the fix

# Clean up
git branch -d hotfix/login-crash

# Also bring the fix into dev
git checkout dev
git merge main
git push
```

## Visual Flow: Multiple Experiments at Once

```
main:              ────●──────────────────●────────●──────
                        \                / merge   |
experiment/dark:         ●────●────●────●          |
                                                   |
                        \                         / merge
experiment/pricing:      ●────●────●────●────●───●
```

You can run multiple experiments at the same time. Each one branches from `main` and lives in isolation. Merge them one at a time when they are ready.

```bash
# Start experiment A
git checkout main
git checkout -b experiment/dark-mode

# Start experiment B (from main, not from experiment A)
git checkout main
git checkout -b experiment/new-pricing

# Work on whichever one you want
git checkout experiment/dark-mode
# make changes, commit, push

git checkout experiment/new-pricing
# make changes, commit, push
```

## Branch Naming Rules

| Type | Pattern | Example |
|------|---------|---------|
| Production | `main` | `main` |
| Daily work | `dev` | `dev` |
| Experiment | `experiment/short-name` | `experiment/ai-chat` |
| Hotfix | `hotfix/short-name` | `hotfix/login-crash` |

Rules:
- All lowercase
- Use hyphens, not spaces or underscores
- Keep names short but descriptive
- No special characters

## When to Use Each Branch

| Situation | Branch to use |
|-----------|--------------|
| Building a new feature | `dev` |
| Trying a wild idea that might not work | `experiment/idea-name` |
| Production is broken, users are affected | `hotfix/bug-name` |
| Deploying to production | merge into `main` |
| Starting a totally new direction for the app | `experiment/v2-rewrite` |
| Quick CSS or copy change | `dev` (commit and merge to main) |

## Keeping Branches in Sync

Your `dev` branch can fall behind `main` if you merge hotfixes or experiments directly into main. Periodically sync:

```bash
git checkout dev
git merge main
git push
```

Do this:
- After every hotfix merge to main
- After merging an experiment to main
- At the start of each work session

## Cleaning Up Old Branches

```bash
# See all branches
git branch

# Delete a local branch you are done with
git branch -d experiment/old-idea

# Delete a remote branch
git push origin --delete experiment/old-idea

# Delete all local branches that are already merged
git branch --merged main | grep -v "main\|dev" | xargs git branch -d
```

## Full Lifecycle Example

```
Week 1:
  main:  ──●───────────────────────────●── (deploy v1)
             \                        /
  dev:        ●──●──●──●──●──●──●──●

Week 2:
  main:  ──●────────●──────────────────●── (deploy v2)
                     |                /
  hotfix/bug:  ──────●  (emergency)  /
                                    /
  dev:        ●──●──●──●──●──●──●

Week 3:
  main:  ──●────────────────●──────────●── (deploy v3)
             \             / (merge)   |
  experiment: ●──●──●──●──            |
                                      /
  dev:        ●──●──●──●──●──●──●──●
```

This is the entire system. Four branch types, simple rules, production stays safe.
