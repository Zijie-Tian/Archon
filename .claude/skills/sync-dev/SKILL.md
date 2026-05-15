---
name: sync-dev
description: |
  Use when: User wants to sync the dev branch updates to current branch, 
  update dev branch from remote, or merge dev changes.
  Triggers: "update dev", "sync dev", "merge dev", "pull dev", 
            "dev branch update", "get dev changes", "sync with dev".
  Capability: Fast-forward merge dev branch updates into current branch safely.
---

# Sync Dev Branch Skill

This skill automates the safe workflow of updating the local `dev` branch from remote and merging those changes into the current feature branch.

## Workflow

When the user asks to sync dev or update dev branch:

1. **Check current branch**
   - Run `git branch --show-current` to identify current branch
   - If already on `dev`, just pull and done

2. **Update dev branch**
   - `git checkout dev`
   - `git pull origin dev`
   - Report how many commits were fetched

3. **Return to original branch**
   - `git checkout <original-branch>`

4. **Merge dev into current branch**
   - `git merge dev --no-edit`
   - If fast-forward possible, use it
   - If merge conflicts occur, stop and report them to user

5. **Push if needed**
   - If user asks to push: `git push origin <current-branch>`

## Safety Rules

- **Never push dev branch directly** — dev is the integration branch
- **Always return to original branch** after updating dev
- **Report conflicts clearly** if merge fails
- **Use --no-edit** for non-interactive merges

## Example Commands

```bash
# Full sync workflow
git checkout dev && git pull origin dev && git checkout tzj/harness && git merge dev --no-edit

# Check status after merge
git log --oneline --graph --all -10
```

## Success Criteria

- dev branch is at origin/dev HEAD
- Current branch contains all dev commits
- No uncommitted changes left
- Working tree is clean