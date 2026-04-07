# Intent-to-Command Mapping

Complete reference of vibe coder phrases mapped to exact git commands. Over 50 scenarios organized by category.

---

## Saving Work

| # | What you say | What to run |
|---|-------------|-------------|
| 1 | "Save my work" | `git add . && git commit -m "feat(scope): description" && git push` |
| 2 | "Save just this one file" | `git add src/app/page.tsx && git commit -m "feat(home): update hero section" && git push` |
| 3 | "Save but don't upload yet" | `git add . && git commit -m "feat(scope): description"` |
| 4 | "Upload my saves to GitHub" | `git push` |
| 5 | "Save everything including new files" | `git add . && git commit -m "feat(scope): add new files" && git push` |
| 6 | "I want to save my progress but I'm not done" | `git add . && git commit -m "chore(scope): work in progress on feature"` |
| 7 | "Put my changes aside for a minute" | `git stash` (get them back with `git stash pop`) |
| 8 | "Get back the changes I put aside" | `git stash pop` |
| 9 | "See what I put aside" | `git stash list` |
| 10 | "Save multiple sets of changes separately" | `git stash push -m "pricing page changes"` then work, then `git stash push -m "auth changes"` |

## Undoing and Going Back

| # | What you say | What to run |
|---|-------------|-------------|
| 11 | "Undo everything I just did" | `git checkout -- .` (erases unsaved changes, cannot undo) |
| 12 | "Undo changes to one file" | `git checkout -- src/app/page.tsx` |
| 13 | "Go back to my last save" | `git reset --hard HEAD` (erases everything since last commit) |
| 14 | "Undo my last commit but keep the files" | `git reset --soft HEAD~1` |
| 15 | "Undo my last 3 commits but keep the files" | `git reset --soft HEAD~3` |
| 16 | "Completely erase my last commit" | `git reset --hard HEAD~1` |
| 17 | "Undo a commit I already pushed" | `git revert HEAD && git push` |
| 18 | "Undo a specific old commit" | `git revert abc1234 && git push` |
| 19 | "Go back to how things were yesterday" | `git log --oneline --before="yesterday"` then `git checkout abc1234 -- .` |
| 20 | "Go back to a specific save point" | `git log --oneline -20` find the hash, then `git checkout abc1234 -- .` |
| 21 | "I accidentally deleted a file" | `git checkout HEAD -- path/to/file.tsx` |
| 22 | "Restore a file from 5 commits ago" | `git checkout HEAD~5 -- path/to/file.tsx` |
| 23 | "Get back something I deleted days ago" | `git reflog` find the hash, `git checkout abc1234 -- path/to/file.tsx` |

## Branching and Experiments

| # | What you say | What to run |
|---|-------------|-------------|
| 24 | "Let me try something experimental" | `git checkout -b experiment/feature-name` |
| 25 | "That experiment worked, keep it" | `git checkout main && git merge experiment/feature-name && git push` |
| 26 | "That experiment failed, trash it" | `git checkout main && git branch -D experiment/feature-name` |
| 27 | "I want two versions of my app" | Create two branches: `git checkout -b version-a` and `git checkout -b version-b` |
| 28 | "Switch to my other version" | `git checkout branch-name` |
| 29 | "What branches do I have?" | `git branch` |
| 30 | "What branch am I on?" | `git branch --show-current` |
| 31 | "Create a branch for this bug fix" | `git checkout -b hotfix/describe-the-bug` |
| 32 | "Combine my experiment into the main app" | `git checkout main && git merge experiment/feature-name && git push` |
| 33 | "Start fresh from main" | `git checkout main && git pull && git checkout -b new-feature` |
| 34 | "Delete a branch I don't need" | `git branch -d branch-name` (safe) or `git branch -D branch-name` (force) |
| 35 | "Rename my branch" | `git branch -m old-name new-name` |
| 36 | "Copy someone else's branch" | `git fetch origin && git checkout their-branch-name` |

## Viewing History and Changes

| # | What you say | What to run |
|---|-------------|-------------|
| 37 | "What changed?" | `git diff --stat` |
| 38 | "What did I change in this file?" | `git diff src/app/page.tsx` |
| 39 | "Show me my recent saves" | `git log --oneline -10` |
| 40 | "What did I do today?" | `git log --oneline --after="midnight"` |
| 41 | "What did I do this week?" | `git log --oneline --after="1 week ago"` |
| 42 | "Who changed this file?" | `git log --oneline -- path/to/file.tsx` |
| 43 | "Show me the exact changes in a commit" | `git show abc1234` |
| 44 | "What's different between my branch and main?" | `git diff main...HEAD --stat` |
| 45 | "How many lines did I change?" | `git diff --shortstat` |
| 46 | "Show me yesterday's version of a file" | `git log --oneline --before="yesterday" -- file.tsx` then `git show abc1234:file.tsx` |

## Collaborating and Syncing

| # | What you say | What to run |
|---|-------------|-------------|
| 47 | "Get the latest code" | `git pull` |
| 48 | "Download changes without applying them" | `git fetch` |
| 49 | "Someone updated main, I need those changes" | `git checkout main && git pull && git checkout my-branch && git merge main` |
| 50 | "Send my branch to GitHub" | `git push -u origin branch-name` |
| 51 | "Create a pull request" | `gh pr create --title "type(scope): description" --body "summary"` |
| 52 | "See open pull requests" | `gh pr list` |
| 53 | "Check out someone's pull request" | `gh pr checkout 42` |
| 54 | "Merge a pull request" | `gh pr merge --squash --delete-branch` |
| 55 | "See the status of my PR checks" | `gh pr checks` |

## Fixing Mistakes

| # | What you say | What to run |
|---|-------------|-------------|
| 56 | "I committed secrets / API keys" | Rotate keys first, then `bfg --delete-files .env.local && git push --force` |
| 57 | "My commit message has a typo" | `git commit --amend -m "correct message"` (only if not pushed yet) |
| 58 | "I committed to the wrong branch" | `git reset --soft HEAD~1 && git stash && git checkout right-branch && git stash pop && git add . && git commit -m "message"` |
| 59 | "I need to split my commit into two" | `git reset --soft HEAD~1` then commit files in groups |
| 60 | "Something broke and I don't know what" | `git bisect start && git bisect bad && git bisect good abc1234` then test each |
| 61 | "My repo is totally messed up" | `cp -r project project-backup && cd .. && rm -rf project && git clone url` |
| 62 | "I have merge conflicts" | Edit the files, remove `<<<<<<<`/`=======`/`>>>>>>>` markers, then `git add . && git commit -m "fix: resolve merge conflicts"` |
| 63 | "I pushed something broken to main" | `git revert HEAD && git push` |
| 64 | "I want to undo multiple pushed commits" | `git revert HEAD~3..HEAD && git push` |

## Setup and Configuration

| # | What you say | What to run |
|---|-------------|-------------|
| 65 | "Start version control on my project" | `git init && git add . && git commit -m "chore: initial commit"` |
| 66 | "Connect my project to GitHub" | `gh repo create project-name --public --source=. --push` |
| 67 | "Clone a project from GitHub" | `git clone https://github.com/user/repo.git` |
| 68 | "Set up my git identity" | `git config --global user.name "Name" && git config --global user.email "email@example.com"` |
| 69 | "Make git ignore certain files" | Create `.gitignore` and add patterns like `node_modules/`, `.env.local`, `.next/` |
| 70 | "See my git configuration" | `git config --list` |

## Deployment-Related

| # | What you say | What to run |
|---|-------------|-------------|
| 71 | "Deploy my changes" | `git checkout main && git merge dev && git push` (Vercel auto-deploys main) |
| 72 | "Deploy from a specific branch" | Push the branch, set it as production branch in Vercel dashboard |
| 73 | "Roll back the deploy" | `git revert HEAD && git push` on main |
| 74 | "Preview this branch before deploying" | `git push -u origin branch-name` then create a PR (Vercel makes preview URL) |
| 75 | "Check if main is safe to deploy" | `git checkout main && npm run build && npx tsc --noEmit` |
