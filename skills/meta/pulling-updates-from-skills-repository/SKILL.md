---
name: Pulling Updates from Skills Repository
description: Sync local skills repository with upstream changes from obra/superpowers-skills
when_to_use: when session start indicates new upstream skills available, or when manually updating to latest versions
version: 1.1.0
---

# Updating Skills from Upstream

## Overview

Pull and merge upstream changes from obra/superpowers-skills into your local skills repository while preserving your personal modifications.

**Announce at start:** "I'm using the Updating Skills skill to sync with upstream."

## Prerequisites

Your skills repo must have an `upstream` remote pointing to obra/superpowers-skills. The plugin sets this up automatically.

## The Process

### Step 1: Check Current Status

Run:
```bash
cd ~/.config/superpowers/skills
git status
```

**If working directory is dirty:** Proceed to Step 2 (stash changes)
**If clean:** Skip to Step 3

### Step 2: Stash Uncommitted Changes (if needed)

Run:
```bash
git stash push -m "Temporary stash before upstream update"
```

Record: Whether changes were stashed (you'll need to unstash later)

### Step 3: Fetch Upstream Changes

Run:
```bash
git fetch upstream
```

Expected: Fetches latest commits from obra/superpowers-skills

### Step 4: Check What's New

Run:
```bash
git log HEAD..upstream/main --oneline
```

Show user: List of new commits being pulled

### Step 5: Merge Upstream Changes

Run:
```bash
git merge upstream/main
```

**If merge succeeds cleanly:** Proceed to Step 6
**If conflicts occur:** Proceed to conflict resolution

### Step 6: Handle Merge Conflicts (if any)

If conflicts:
1. Run `git status` to see conflicted files
2. For each conflict, explain to user what changed in both versions
3. Ask user which version to keep or how to merge
4. Edit files to resolve
5. Run `git add <resolved-file>` for each
6. Run `git commit` to complete merge

### Step 7: Unstash Changes (if stashed in Step 2)

If you stashed changes:
```bash
git stash pop
```

**If conflicts with unstashed changes:** Help user resolve them

### Step 8: Verify Everything Works

Run:
```bash
${SUPERPOWERS_SKILLS_ROOT}/skills/using-skills/find-skills
```

Expected: Skills list displays correctly

### Step 9: Announce Completion

Tell user:
- How many new commits were merged
- Whether any conflicts were resolved
- Whether their stashed changes were restored
- That skills are now up to date

## Common Issues

**"Already up to date"**: Your local repo is current, no action needed

**Detached HEAD**: You're not on a branch. Ask user if they want to create a branch or check out main.

**No upstream remote**: Run `git remote add upstream https://github.com/obra/superpowers-skills.git`

## Remember

- Always stash uncommitted work before merging
- Explain conflicts clearly to user
- Test that skills work after update
- User's local commits/branches are preserved
