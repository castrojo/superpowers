---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and you need to decide how to integrate the work - guides completion of development work by presenting structured options for merge, PR, or cleanup
---

# Finishing a Development Branch

## Overview

Guide completion of development work by presenting clear options and handling chosen workflow.

**Core principle:** Verify tests → Present options → Execute choice → Clean up.

**Announce at start:** "I'm using the finishing-a-development-branch skill to complete this work."

## The Process

### Step 1: Verify Tests

**Before presenting options, verify tests pass:**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**If tests fail:**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

Stop. Don't proceed to Step 2.

**If tests pass:** Continue to Step 1.5.

### Step 1.5: Check for Epic Reference

Check if this work relates to an epic:

```bash
# Check commits for epic references
EPIC_NUM=$(git log --oneline -20 | grep -oP '(?:epic|Epic)[- ]#?\K\d+' | head -1)

# Check for plan file with epic reference
if [ -z "$EPIC_NUM" ]; then
  PLAN_FILE=$(git log --oneline -20 | grep -oP 'docs/plans/[^\s]+\.md' | head -1)
  if [ -n "$PLAN_FILE" ] && [ -f "$PLAN_FILE" ]; then
    EPIC_NUM=$(grep -oP 'Epic Issue.*#\K\d+' "$PLAN_FILE" | head -1)
  fi
fi

# Verify it's an epic
if [ -n "$EPIC_NUM" ]; then
  if gh issue view "$EPIC_NUM" --json labels --jq '.labels[].name' | grep -q "epic"; then
    echo "✅ Found epic: #$EPIC_NUM"
  else
    EPIC_NUM=""
  fi
fi
```

Store `$EPIC_NUM` for later steps.

### Step 2: Determine Base Branch

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

Or ask: "This branch split from main - is that correct?"

### Step 2.5: Detect Production Branch Promotion Context

If the base branch is a separate long-lived production branch (e.g. `lts`, `stable`, `prod`) that
receives content from `main` via promotion — not a feature branch receiving a feature — stop here.
**Do not proceed with a merge-based PR.**

**Detect with:**
```bash
# Large diverged history = promotion context
git log --oneline origin/<base-branch>...origin/main | wc -l
# If output >> 1, you're in a promotion context

# No-op check — if files already match, nothing to do
git diff origin/<base-branch> origin/main -- <files>
# Empty output = already in sync, no PR needed
```

**Signs you're in this context:**
- Repo has `main` + a separate `lts`/`stable`/`prod` branch
- Large diverged history between the two branches
- Intent is "sync these files to production", not "merge a feature"

**The rule:** Never use `git merge origin/main` on a branch off `<prod>`. A merge drags the full
diverged history into `<prod>`, polluting it with every intermediate commit that was never
cleanly promoted.

**Correct pattern — single clean commit:**
```bash
# 1. Branch off production tip — NOT main
git checkout -b fix/sync-<description> origin/<prod-branch>

# 2. Copy files directly from main — no merge
git checkout origin/main -- <file1> <file2> ...

# 3. Verify: only one new commit above prod tip
git log --oneline origin/<prod-branch>..HEAD   # must be exactly 1

# 4. Verify: files match main exactly
git diff origin/main -- <files>                # must be empty

# 5. Commit and push
git commit -m "fix: sync <description> from main"
git push origin fix/sync-<description>
gh pr create --base <prod-branch> --head fix/sync-<description>
```

**Then proceed to Step 3 with `<prod-branch>` as base.**

### Step 3: Present Options

Present exactly these 4 options:

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Don't add explanation** - keep options concise.

### Step 4: Execute Choice

#### Option 1: Merge Locally

```bash
# Switch to base branch
git checkout <base-branch>

# Pull latest
git pull

# Merge feature branch
git merge <feature-branch>

# Verify tests on merged result
<test command>

# If tests pass
git branch -d <feature-branch>
```

**If epic detected in Step 1.5:**

Use `epic-journey-update` skill to document implementation journey.

Provide:
- Epic number: `$EPIC_NUM`
- Branch: `<feature-branch>`
- Base: `<base-branch>`

Then: Cleanup worktree (Step 5)

#### Option 2: Push and Create PR

**MANDATORY: Follow the Upstream PR Protocol below. Do not run `gh pr create` directly.**

Push the working branch to origin first:

```bash
git push -u origin <feature-branch>
```

Then execute the full Upstream PR Protocol (see below).

**Note:** Epic journey updated when PR merges (use epic-journey-update skill at that time).

Then: Cleanup worktree (Step 5)

---

## Upstream PR Protocol

**This protocol applies to every PR sent to an upstream repo. No exceptions, including
when the user explicitly asks you to skip steps.**

### Step A: Detect upstream

```bash
git remote -v
# If an `upstream` remote exists, this repo has an upstream. Full protocol required.
```

### Step B: Squash to one clean commit

```bash
# 1. Identify the target upstream branch (usually lts or main)
TARGET=lts  # or main — check with user if unclear

# 2. Create a clean branch off upstream tip
git checkout -b clean/<branch-name> upstream/$TARGET

# 3. Squash all working branch commits into one
git merge --squash <working-branch>

# 4. Commit with a single clean conventional commit message
#    - No WIP markers, no LLM noise, no "Assisted-by" co-author artifacts in the subject
#    - Format: type(scope): description
#    - Assisted-by footer is still required
git commit -m "type(scope): description

Assisted-by: [Model] via [Tool]"

# 5. Verify exactly one commit above upstream tip
git log --oneline upstream/$TARGET..HEAD
# Must show exactly 1 line. If more, something went wrong — stop and fix.

# 6. Push clean branch to origin (your fork), never to upstream
git push origin clean/<branch-name>
```

### Step C: Double confirmation (both required, no exceptions)

**Q1** — use the `question` tool:
> "You are about to open a PR to `<upstream-org>/<repo>`. This will be visible to upstream
> maintainers. Are you sure?"
> Options: `Yes, proceed` / `No, abort`

If Q1 is not `Yes, proceed`: **STOP. Do not continue.**

**Q2** — use the `question` tool (only after Q1 confirmed):
> "FINAL CONFIRMATION: The browser will open with a pre-filled PR form targeting
> `<upstream-org>/<repo>`. You must manually click 'Create Pull Request'. Proceed?"
> Options: `Open browser now` / `Abort`

If Q2 is not `Open browser now`: **STOP. Do not continue.**

### Step D: Open browser — user submits

```bash
gh pr create --web --repo <upstream-org>/<repo> \
  --base $TARGET \
  --head castrojo/<repo>:clean/<branch-name> \
  --title "type(scope): description"
```

The browser opens with the form pre-filled. **You do nothing further. The user clicks
"Create Pull Request".**

### Step E: Clean up the squash branch

After the user confirms they have submitted:

```bash
git branch -d clean/<branch-name>
git push origin --delete clean/<branch-name>
```

The working branch (`<feature-branch>`) is kept until the upstream PR merges.

---

#### Option 3: Keep As-Is

Report: "Keeping branch <name>. Worktree preserved at <path>."

**Don't cleanup worktree.**

#### Option 4: Discard

**Confirm first:**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

Wait for exact confirmation.

If confirmed:
```bash
git checkout <base-branch>
git branch -D <feature-branch>
```

Then: Cleanup worktree (Step 5)

### Step 5: Cleanup Worktree

**For Options 1, 2, 4:**

Check if in worktree:
```bash
git worktree list | grep $(git branch --show-current)
```

If yes:
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree.

## Quick Reference

| Option | Merge | Push | Keep Worktree | Cleanup Branch |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | ✓ | - | - | ✓ |
| 2. Create PR | - | ✓ (origin only) | ✓ | - (keep until PR merges) |
| 3. Keep as-is | - | - | ✓ | - |
| 4. Discard | - | - | - | ✓ (force) |

**Option 2 always requires:** squash → double confirm → `--web` → user submits manually.

## Common Mistakes

**Skipping test verification**
- **Problem:** Merge broken code, create failing PR
- **Fix:** Always verify tests before offering options

**Open-ended questions**
- **Problem:** "What should I do next?" → ambiguous
- **Fix:** Present exactly 4 structured options

**Automatic worktree cleanup**
- **Problem:** Remove worktree when might need it (Option 2, 3)
- **Fix:** Only cleanup for Options 1 and 4

**No confirmation for discard**
- **Problem:** Accidentally delete work
- **Fix:** Require typed "discard" confirmation

**Merging main into a branch off a production branch**
- **Problem:** Drags full diverged history into production, polluting it with every intermediate commit
- **Fix:** Never merge. Use `git checkout origin/main -- <files>` to copy files directly as a single clean commit.

## Red Flags

**Never:**
- Proceed with failing tests
- Merge without verifying tests on result
- Delete work without confirmation
- Force-push without explicit request
- Merge `main` into a branch based off `lts`/`stable`/`prod` → use `git checkout origin/main -- <files>` instead
- Run `gh pr create` without completing both confirmation dialogs
- Run `gh pr create` to upstream without `--web`
- Push to the `upstream` remote
- Send multi-commit history upstream — squash is mandatory
- Auto-submit a PR — user always clicks "Create Pull Request" in the browser

**Always:**
- Verify tests before offering options
- Present exactly 4 options
- Get typed confirmation for Option 4
- Clean up worktree for Options 1 & 4 only
- Load this skill before any PR work (mandated by `~/.config/opencode/AGENTS.md`)

## Integration

**Called by:**
- **subagent-driven-development** (Step 7) - After all tasks complete
- **executing-plans** (Step 5) - After all batches complete

**Pairs with:**
- **using-git-worktrees** - Cleans up worktree created by that skill
