# The Day I Accidentally Committed with Wrong Git Credentials (And How to Fix It)

## The Problem: When Two Git Accounts Collide

Picture this: You have two GitHub accounts – maybe one personal and one for your projects. You've been working on a personal repository, but your global Git config was set to your other account. Now you've made several commits and pushed them to GitHub, only to realize all your commits show the wrong author information.

Sound familiar? This exact scenario happened recently, and here's the complete story of how to fix it safely.

## Understanding the Challenge

Before diving into the solution, let's understand what we're dealing with:

**Git History**: Every Git commit contains immutable author information (name and email). Once committed, this data becomes part of the repository's permanent history.

**Remote vs Local**: When commits are already pushed to GitHub/GitLab, simply changing your local Git config won't fix the historical commits.

**History Rewriting**: To fix past commits, we need to rewrite Git history, which creates new commit objects with corrected information.

## The Solution: git-filter-repo

Git-filter-repo is the modern, recommended tool for rewriting repository history, replacing the older and much slower git-filter-branch. It's significantly faster (often 10x or more) and includes built-in safety measures to prevent accidental data loss.

### Why git-filter-repo over git-filter-branch?

Git-filter-branch is deprecated due to performance and safety issues, while git-filter-repo is now officially recommended by the Git project. Git-filter-repo includes safety checks like requiring fresh clones and preventing data corruption.

## Step-by-Step Fix Guide

### Prerequisites
- ⚠️ **No collaborators have pulled the affected commits**
- ⚠️ **Create a backup**: `git clone your-repo your-repo-backup`
- ✅ **You have push access to the repository**
- ✅ **All commits are already pushed** (this guide is for rewriting remote history)

### Step 1: Install git-filter-repo

Git-filter-repo requires Python 3.6 or above:

```bash
pip install git-filter-repo
```

Verify installation:
```bash
git filter-repo --version
```

### Step 2: Create the mailmap file

Create a `mailmap.txt` file following the Git mailmap format:

```bash
# mailmap.txt format: Correct Name <correct.email> Wrong Name <wrong.email>
Correct Name <correct.email@example.com> <wrong.email@example.com>
Correct Name <correct.email@example.com> Wrong Name <wrong.email@example.com>
```

**Replace with your actual details:**
- `Correct Name <correct.email@example.com>` → Your intended author info
- `<wrong.email@example.com>` and `Wrong Name` → The incorrect credentials that were used

### Step 3: Run git-filter-repo

```bash
git filter-repo --mailmap mailmap.txt
```

**What happens here:**
- Git-filter-repo processes history using `git fast-export | filter | git fast-import`
- It removes the remote origin as a safety feature to prevent accidental pushes
- All commits with matching author info get rewritten with correct credentials

### Step 4: Re-add remote origin

Git-filter-repo removes remotes as a safety measure, so we need to add it back:

```bash
git remote add origin <your-repo-url>
```

### Step 5: Update remote tracking and force push

```bash
# Fetch latest remote state
git fetch origin

# Set up tracking (if needed)
git branch --set-upstream-to=origin/main main

# Force push the corrected history
git push --force-with-lease origin main
```

**If `--force-with-lease` fails with "stale info":**
Force-with-lease is safer than regular force as it checks that the remote hasn't changed since your last fetch. If it fails:

```bash
git push --force origin main
```

### Step 6: Verify and clean up

```bash
# Verify the changes worked
git log --pretty=format:"%h %an <%ae> %s"

# Remove the mailmap file
rm mailmap.txt
```

## Troubleshooting Common Issues

### Issue 1: "Origin does not appear to be a git repository"

**Problem**: Git-filter-repo removes the remote origin automatically as a safety feature.

**Solution**: Re-add your remote origin (Step 4 above).

### Issue 2: "Stale info" error with --force-with-lease

**Problem**: Your local repository doesn't have up-to-date remote tracking information.

**Solutions**:
```bash
# Option 1: Update tracking first (safer)
git fetch origin
git push --force-with-lease origin main

# Option 2: Use regular force (if no collaborators)
git push --force origin main
```

### Issue 3: Divergent branches after filter-repo

**Problem**: Your local branch shows as diverged from remote with different commit counts.

**Critical**: **DO NOT** run `git pull` as this will merge the old (wrong) commits back into your corrected history.

**Solution**: Reset local to match the corrected remote:
```bash
# Option 1: Hard reset (discards local changes)
git fetch origin
git reset --hard origin/main

# Option 2: Preserve workspace changes
git stash
git fetch origin
git reset --hard origin/main
git stash pop
```

## Understanding Old Commit Behavior

After successfully rewriting history, you might notice something strange: **old commit hashes are still accessible** via direct URLs, showing a message like "This commit does not belong to any branch on this repository."

**This is completely normal and expected:**

✅ **Why it happens**: Git creates new commit objects with new hashes when rewriting history. Old commit objects remain in the hosting platform's object store temporarily for recovery purposes.

✅ **What it means**: Old commits are "orphaned" - no longer part of any branch's history but temporarily accessible for ~90 days.

✅ **No action needed**: These will be automatically garbage collected. Your repository history is correctly rewritten.

✅ **Verification**: Check that `git log` shows corrected author information and new commit hashes.

## Prevention: Avoiding Future Mix-ups

Set up conditional Git configs to automatically use the right credentials based on directory:

```bash
# In ~/.gitconfig
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work
[includeIf "gitdir:~/personal/"]
    path = ~/.gitconfig-personal
```

Create separate config files:
```bash
# ~/.gitconfig-work
[user]
    name = Your Work Name
    email = work.email@company.com

# ~/.gitconfig-personal  
[user]
    name = Your Personal Name
    email = personal.email@example.com
```

## Important Safety Notes

⚠️ **History Rewriting Risks**: 
- Rewriting history is irreversible and can cause issues if collaborators have already pulled the commits
- Forces re-cloning for anyone who has pulled the affected commits
- Removes commit signatures as they depend on commit hashes

⚠️ **When NOT to use this approach**:
- If teammates have already pulled the affected commits
- On shared/production repositories without team coordination
- If you're unsure about the implications

✅ **Safe scenarios**:
- Personal repositories
- Recent commits that haven't been pulled by others
- Team repositories with full coordination

## Key Takeaways

1. **git-filter-repo is the modern standard** for rewriting Git history safely and efficiently
2. **Force-with-lease is safer than regular force** for pushing rewritten history
3. **Old commits remaining accessible is normal** after history rewriting
4. **Prevention through conditional configs** saves future headaches
5. **Always backup before rewriting history** - this operation is irreversible

By following this guide, you can safely correct author information in your Git repositories while understanding exactly what's happening at each step. The key is taking the right precautions and understanding the implications of rewriting history.

*Remember: When in doubt, consult with your team before rewriting shared repository history!*

## Learn more:

- https://www.git-tower.com/learn/git/faq/git-filter-repo
- https://selfformat.com/blog/2025/02/26/how-to-change-git-history-with-git-filter-repo/
- https://www.redhat.com/en/blog/clean-git-repository
- https://www.datacamp.com/courses-all?number=1&content_type=course&technology_array=Git
