# Git Recovery Patterns

Every "I messed up" scenario with the exact fix. Stay calm. Git almost never actually loses data.

---

## Recovery Decision Tree

```
What happened?
|
+-- I changed files but haven't committed
|   +-- I want to undo ALL changes -------> git checkout -- .
|   +-- I want to undo ONE file ----------> git checkout -- path/to/file
|   +-- I want to save changes for later -> git stash
|
+-- I committed but haven't pushed
|   +-- Undo commit, keep changes --------> git reset --soft HEAD~1
|   +-- Undo commit, erase changes -------> git reset --hard HEAD~1
|   +-- Fix the commit message -----------> git commit --amend -m "new msg"
|
+-- I committed AND pushed
|   +-- Undo safely (adds new commit) ----> git revert HEAD && git push
|   +-- Undo multiple commits ------------> git revert HEAD~3..HEAD && git push
|
+-- I deleted something
|   +-- File was in last commit ----------> git checkout HEAD -- path/to/file
|   +-- File was in an older commit ------> git reflog, find hash, checkout
|   +-- Entire branch deleted ------------> git reflog, find hash, git checkout -b name hash
|
+-- Something else
    +-- Committed to wrong branch --------> See "Wrong Branch" below
    +-- Committed secrets ----------------> See "Secrets Exposed" below
    +-- Merge conflicts ------------------> See "Merge Conflicts" below
    +-- Repo is totally broken -----------> See "Nuclear Option" below
```

---

## Scenario 1: Undo Uncommitted Changes

**You changed files but have not committed yet.**

### Undo all changes (go back to last commit)

```bash
git checkout -- .
```

Warning: this erases all your changes permanently. There is no undo.

### Undo changes to a single file

```bash
git checkout -- src/app/page.tsx
```

### Save changes for later without committing

```bash
git stash
```

Get them back:
```bash
git stash pop
```

### See what you would be erasing

```bash
git diff
```

---

## Scenario 2: Undo a Commit (Not Pushed Yet)

**You committed but have not pushed to GitHub.**

### Undo commit, keep your file changes

```bash
git reset --soft HEAD~1
```

Your files stay changed, just the commit is removed. You can edit and re-commit.

### Undo commit and erase all changes

```bash
git reset --hard HEAD~1
```

Warning: this erases the commit AND your changes. Cannot undo.

### Undo the last 3 commits, keep file changes

```bash
git reset --soft HEAD~3
```

### Fix a typo in your commit message

```bash
git commit --amend -m "fix(auth): correct session validation logic"
```

Only do this if you have NOT pushed yet. If you already pushed, just make a new commit.

---

## Scenario 3: Undo a Pushed Commit

**You committed and pushed, and now the code is on GitHub.**

### Undo the last commit safely

```bash
git revert HEAD
git push
```

This creates a NEW commit that reverses the previous one. Safe for shared repos.

### Undo a specific old commit

```bash
git log --oneline -20
# find the commit hash
git revert abc1234
git push
```

### Undo multiple pushed commits

```bash
# Undo the last 3 commits
git revert HEAD~3..HEAD --no-commit
git commit -m "revert: undo last 3 commits due to regression"
git push
```

---

## Scenario 4: Deleted Files Recovery

**You deleted a file and need it back.**

### File was committed and you just deleted it from your filesystem

```bash
git checkout HEAD -- src/components/Header.tsx
```

### File existed in an older commit

```bash
# Find when the file last existed
git log --all --oneline -- src/components/Header.tsx
# Check out from that commit
git checkout abc1234 -- src/components/Header.tsx
```

### You deleted a file and committed the deletion

```bash
# Find the commit before the deletion
git log --diff-filter=D --oneline -- src/components/Header.tsx
# The output shows the commit that deleted it. Use the commit BEFORE it:
git checkout abc1234~1 -- src/components/Header.tsx
```

### You deleted an entire branch

```bash
# Branches are just pointers. The commits still exist.
git reflog
# Find the last commit of that branch
git checkout -b recovered-branch abc1234
```

---

## Scenario 5: Committed Secrets / API Keys

**You accidentally committed `.env.local`, API keys, passwords, or tokens.**

This is an emergency. Keys exposed in git history can be found by bots within minutes.

### Step 1: Rotate the secrets NOW

Before doing anything with git, go to every service dashboard and generate new keys:
- Supabase: Settings > API > Regenerate keys
- Stripe: Developers > API keys > Roll keys
- Any other service: find the key rotation option

Update your `.env.local` with the new keys.

### Step 2: Remove secrets from git history

```bash
# Option A: BFG Repo Cleaner (recommended, fast)
# Install: brew install bfg (macOS) or download from https://rsa.pub/bfg
bfg --delete-files .env.local
bfg --delete-files .env
bfg --replace-text passwords.txt  # file containing strings to redact

git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force
```

```bash
# Option B: git filter-repo (if BFG not available)
pip install git-filter-repo
git filter-repo --path .env.local --invert-paths
git push --force
```

### Step 3: Prevent it from happening again

Add to `.gitignore`:
```gitignore
.env
.env.local
.env.production
.env.development.local
.env.test.local
.env.production.local
.env*.local
```

Check if any env files are tracked:
```bash
git ls-files | grep -i env
```

If any show up, untrack them:
```bash
git rm --cached .env.local
git commit -m "chore: remove tracked env file"
git push
```

### Step 4: Check for exposure

- GitHub: check if the repo is public. If yes, assume keys were stolen.
- Check https://github.com/settings/security for any GitHub security alerts.

---

## Scenario 6: Committed to the Wrong Branch

**You made commits on `main` when you meant to be on `dev`.**

### Move last commit to the correct branch

```bash
# Note the commit hash
git log --oneline -1
# abc1234 feat(dashboard): add chart

# Undo the commit on main (keep the changes)
git reset --soft HEAD~1

# Stash the changes
git stash

# Switch to the correct branch
git checkout dev

# Apply the changes
git stash pop
git add .
git commit -m "feat(dashboard): add chart"
git push
```

### Move last 3 commits to a new branch

```bash
# Create a new branch at current position (keeps the commits)
git branch new-feature

# Reset main back 3 commits
git reset --hard HEAD~3

# Switch to new branch — your commits are there
git checkout new-feature
```

---

## Scenario 7: Merge Conflicts

**Git says "CONFLICT" and refuses to merge.**

### Understanding what happened

Two branches changed the same line in the same file. Git does not know which version to keep. It marks the conflict in the file like this:

```
function getUser() {
<<<<<<< HEAD
  return db.query("SELECT * FROM users WHERE active = true");
=======
  return db.query("SELECT * FROM users WHERE verified = true");
>>>>>>> feature-branch
}
```

- Between `<<<<<<< HEAD` and `=======` is YOUR version (the branch you are on).
- Between `=======` and `>>>>>>> feature-branch` is THEIR version (the branch you are merging).

### How to fix

1. Open the file with the conflict.
2. Decide which version to keep (or combine both).
3. Delete the `<<<<<<<`, `=======`, and `>>>>>>>` marker lines.
4. Save the file.

The fixed version might look like:
```
function getUser() {
  return db.query("SELECT * FROM users WHERE active = true AND verified = true");
}
```

5. Stage and commit:
```bash
git add .
git commit -m "fix: resolve merge conflicts in user query"
```

### If there are too many conflicts and you want to abort

```bash
git merge --abort
```

This cancels the merge entirely and goes back to the state before you started.

### Preventing merge conflicts

- Merge `main` into your branch frequently: `git merge main`
- Keep commits small and focused on one change
- Avoid two people editing the same file at the same time

---

## Scenario 8: Finding What Broke Your App (git bisect)

**Your app was working last week, now it is broken, and you don't know which commit caused it.**

```bash
# Start bisect
git bisect start

# Tell git the current state is broken
git bisect bad

# Tell git a known good commit (check git log to find one)
git log --oneline -30
git bisect good abc1234

# Git checks out a commit in the middle. Test your app.
npm run build  # or whatever test you use

# If this commit works:
git bisect good

# If this commit is broken:
git bisect bad

# Repeat until git says:
# "abc1234 is the first bad commit"

# Done. Reset to normal:
git bisect reset
```

Now you know exactly which commit broke things. Look at it:
```bash
git show abc1234
```

---

## Scenario 9: Detached HEAD State

**Git says "You are in 'detached HEAD' state" and you are confused.**

This happens when you check out a specific commit instead of a branch. You are looking at old code but not on any branch.

### Get back to safety

```bash
git checkout main
```

### If you made changes in detached HEAD and want to keep them

```bash
git checkout -b save-my-changes
git add .
git commit -m "feat: save changes from detached HEAD"
```

---

## Scenario 10: Nuclear Option (Total Repo Reset)

**Nothing works. Conflicts everywhere. Git commands giving weird errors. You just want to start over.**

### Step 1: Backup your current code

```bash
cp -r my-project my-project-backup
```

### Step 2: Re-clone from GitHub

```bash
cd ..
rm -rf my-project
git clone https://github.com/yourname/my-project.git
cd my-project
npm install
```

### Step 3: Recover any uncommitted work

If you had uncommitted changes you want to keep, copy them from `my-project-backup`:

```bash
# Compare the backup with the fresh clone to find your changes
diff -rq my-project-backup/src my-project/src
```

Copy over the files you need manually.

### Step 4: If GitHub repo itself is broken

If the remote repo is also messed up and you have a working local copy:

```bash
# Create a new repo
gh repo create my-project-v2 --public

# Copy your code (without .git folder)
cp -r my-project-backup my-project-v2
cd my-project-v2
rm -rf .git

# Fresh git init
git init
git add .
git commit -m "chore: fresh start from working backup"
git remote add origin https://github.com/yourname/my-project-v2.git
git push -u origin main
```

---

## Scenario 11: Git Says "Permission Denied" or Auth Errors

```bash
# Check your current auth
gh auth status

# Re-authenticate
gh auth login

# If using SSH and it broke
ssh -T git@github.com
# If that fails, switch to HTTPS:
git remote set-url origin https://github.com/yourname/my-project.git
```

---

## Scenario 12: Repository is Too Large / Slow

**Git operations take forever because the repo has large files.**

```bash
# See what is taking up space
git rev-list --objects --all | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | sort -rnk3 | head -20

# Remove large files from history
bfg --strip-blobs-bigger-than 10M
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force
```

Prevent it in the future by adding to `.gitignore`:
```gitignore
*.mp4
*.zip
*.tar.gz
*.dmg
*.iso
public/videos/
```

---

## Golden Rules for Avoiding Recovery Situations

1. **Commit often.** Small commits are easy to undo. Big commits are not.
2. **Push daily.** If your laptop dies, GitHub has your code.
3. **Branch before experimenting.** Never experiment on `main`.
4. **Never commit `.env` files.** Check `.gitignore` before your first commit.
5. **Read the error message.** Git usually tells you exactly what to do.
6. **When in doubt, `git stash`.** It saves your work without committing.
7. **Before any dangerous command, check `git status`.** Know what you are about to affect.
