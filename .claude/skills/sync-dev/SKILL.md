---
name: sync-dev
description: |
  Use when: User wants to sync the dev branch updates from upstream fork parent,
  update dev branch from remote, or merge dev changes into current branch.
  Triggers: "update dev", "sync dev", "merge dev", "pull dev", 
            "dev branch update", "get dev changes", "sync with dev",
            "sync fork", "update fork", "sync upstream".
  Capability: Sync fork's dev branch from upstream and merge into current branch safely.
---

# Sync Dev Branch Skill

This skill automates the safe workflow of syncing the `dev` branch from the upstream parent repository (fork sync) and merging those changes into the current feature branch.

## Prerequisites

- `gh` CLI must be installed and authenticated
- Current repository must be a fork (check with `gh repo view --json parent`)

## Workflow

When the user asks to sync dev or update dev branch:

### Option A: Using gh repo sync (Recommended for Forks)

If the repository is a fork of an upstream repo, use `gh repo sync`:

```bash
# Sync dev branch from upstream parent
gh repo sync --branch dev

# Verify sync succeeded
git log --oneline dev -5
```

**What it does:**
- Fetches latest `dev` from the upstream parent repository
- Fast-forwards the local fork's `dev` branch to match upstream
- Works on both local and remote (GitHub) fork

### Option B: Using git pull (If not a fork or gh unavailable)

```bash
# Save current branch
git branch --show-current

# Update dev
git checkout dev
git pull origin dev

# Return to original branch and merge
git checkout <original-branch>
git merge dev --no-edit
```

## Complete Sync Workflow

```bash
# Step 1: Sync dev from upstream (fork sync)
gh repo sync --branch dev

# Step 2: Return to feature branch
git checkout <current-branch>

# Step 3: Merge dev changes
git merge dev --no-edit

# Step 4: Verify
git log --oneline --graph --all -10

# Step 5: Push if needed (optional)
git push origin <current-branch>
```

## Safety Rules

- **Never push dev branch directly** — dev is the integration branch
- **Always return to original branch** after updating dev
- **Report conflicts clearly** if merge fails
- **Use --no-edit** for non-interactive merges
- **Verify fork relationship** before using `gh repo sync`

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| `gh repo sync` fails | Not a fork or no upstream | Use `git pull origin dev` instead |
| Merge conflicts | Divergent changes | Resolve manually, then `git merge --continue` |
| `dev` branch doesn't exist | Branch not created | `git checkout -b dev origin/dev` |
| Permission denied | gh not authenticated | Run `gh auth login` |

## Example Commands

```bash
# Quick sync (one-liner)
gh repo sync --branch dev && git checkout tzj/harness && git merge dev --no-edit

# Full workflow with verification
gh repo sync --branch dev
git checkout tzj/harness
git merge dev --no-edit
git log --oneline --graph --all -10
```

## Success Criteria

- dev branch matches upstream/dev HEAD
- Current branch contains all dev commits
- No uncommitted changes left
- Working tree is clean