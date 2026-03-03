---
name: onboarding-a-repository
description: Use when starting work on any repository for the first time — sets up remotes, plans directory, fork AGENTS.md, project memory block, and validation baseline
---

# Onboarding a Repository

## Overview

Run this checklist **once per repo, before any other work begins.** Every step is mandatory. The skill ends with a journal entry and, if the skill had gaps, an in-session update.

**Announce at start:** "I'm using the onboarding-a-repository skill to set up this repository."

---

## Checklist

### Step 1: Verify remote layout

```bash
git remote -v
```

Expected:
```
origin    git@github.com:castrojo/<repo>.git (fetch)
origin    git@github.com:castrojo/<repo>.git (push)
upstream  git@github.com:<org>/<repo>.git (fetch)
upstream  git@github.com:<org>/<repo>.git (push)
```

**If wrong:** Fix it before doing anything else.

```bash
# Rename your fork remote to origin (if named something else)
git remote rename <fork-name> origin

# If origin currently points to upstream, rotate it:
git remote rename origin upstream-old
git remote add origin git@github.com:castrojo/<repo>.git
git remote set-url upstream git@github.com:<org>/<repo>.git
git remote remove upstream-old

# Fetch and set tracking branches
git fetch upstream
git branch --set-upstream-to=upstream/main main
# If lts branch exists:
git branch --set-upstream-to=upstream/lts lts
```

Full reference: `~/.config/opencode/plans/git-workflow.md`

---

### Step 2: Create plans directory

```bash
mkdir -p ~/.config/opencode/plans/<repo-name>/
```

This is where all project plans, architecture notes, and LLM session artifacts live. **Nothing from this directory ever goes inside the repo.**

---

### Step 3: Discover validation commands

Run these to understand the project's build/test tooling:

```bash
# Check for just (most common in this workflow)
just --list 2>/dev/null || echo "no Justfile"

# Check for make
make help 2>/dev/null || make --dry-run 2>/dev/null | head -20 || echo "no Makefile"

# Check package.json scripts
cat package.json 2>/dev/null | grep -A20 '"scripts"' || echo "no package.json"

# Check for cargo, go, pytest, etc.
ls Cargo.toml go.mod pyproject.toml 2>/dev/null
```

Record the validation command (e.g. `just check && just lint`, `make test`, `npm test`). You will need it in Step 5 and Step 8.

---

### Step 4: Write initial project notes

Create `~/.config/opencode/plans/<repo-name>/project-notes.md` with:

```markdown
# <Repo Name> — Project Notes

## Quick Reference

- **Upstream:** git@github.com:<org>/<repo>.git
- **Fork:** git@github.com:castrojo/<repo>.git
- **Target branch:** main (or lts, stable — identify from upstream)
- **Validation:** <exact command discovered in Step 3>
- **Build tool:** just / make / npm / cargo / ...

## What This Project Is

<1-2 sentences describing the project>

## Key Directories

- `<dir>/` — <purpose>
```

Update this file over time as you learn the project.

---

### Step 5: Check for AGENTS.md on your fork

```bash
# Does castrojo/<repo> already have an AGENTS.md?
git log --oneline --all -- AGENTS.md | head -5
ls AGENTS.md 2>/dev/null && echo "exists" || echo "missing"
```

- If it exists on your fork's branch: review it, update `project` memory block (Step 7), done.
- If it's missing: create one using the **Fork AGENTS.md Pattern** below.

---

### Step 6: Create fork AGENTS.md (if missing)

Create `AGENTS.md` in the repo root using the pattern documented below.

**Commit it to your fork's `main` branch immediately:**

```bash
git add AGENTS.md
git commit -m "chore: add AGENTS.md for AI-assisted workflow

Assisted-by: <Model> via OpenCode"
git push origin main
```

**Hard rules — enforced unconditionally:**
- NEVER include AGENTS.md in any PR to the upstream repo
- NEVER send AGENTS.md to any remote other than `origin` (your `castrojo/<repo>` fork)
- If upstream already has an AGENTS.md, treat it as reference only — your fork's file is authoritative for your workflow

---

### Step 7: Update `project` memory block

Use `memory_set` (scope: project) to write a quick reference for the session:

```
# <Repo Name>

- Validation: <command>
- Build tool: <just/make/npm/...>
- Target branch: <main/lts/...>
- Plans: ~/.config/opencode/plans/<repo-name>/
- Upstream: <org>/<repo>
```

---

### Step 8: Run validation baseline

```bash
<validation command from Step 3>
```

If it passes: note that in the project-notes.md.
If it fails: investigate before starting any work. Do not skip this.

---

### Step 9: Write journal entry

```
journal_write(
  title: "Onboarded <repo-name>",
  body: "Set up remote layout, plans directory, AGENTS.md. Validation: <pass/fail>. Notes: <anything unexpected>.",
  tags: "workflow-learning"
)
```

---

### Step 10: Improve this skill if needed

If any step in this skill was **missing, wrong, or unclear**, invoke `writing-skills` now — before the session ends:

```
skill("writing-skills")
```

Update this skill in-session. Do not defer to a future session.

---

## Fork AGENTS.md Pattern

A fork's project-level AGENTS.md is **context for AI agents working in this specific repo.** It is not a copy of the global rules — those live in `~/.config/opencode/AGENTS.md` and are always in context.

### What to include

| Section | Rule |
|---|---|
| One-paragraph project description | What it is, what it builds, key technology |
| Working effectively / prerequisites | Install commands, environment setup, critical warnings |
| Build commands | Exact commands with timeout notes |
| Validation commands | The `just check && just lint` equivalent |
| Repository structure | Key directories and important files only |
| Common commands reference | Short copy-paste block for daily use |
| Critical project-specific reminders | Anything unique to this codebase the agent must not forget |
| Lazy-load references | If any section would be >100 lines, extract to `~/.config/opencode/plans/<repo>/` and add a one-line reference |

### What to omit (already in global `~/.config/opencode/AGENTS.md`)

| Section | Why to omit |
|---|---|
| Attribution / Assisted-by footer | Global rule, always in context |
| PR submission protocol | Global rule, always in context |
| Conventional commits rule | Global rule, always in context |
| Remote naming convention | Global rule, always in context |
| Branch workflow | Global rule, always in context |
| Banned behaviors list | Global rule, always in context |

### Lazy-load pattern

When a section is large (CI/CD architecture, build failure reference, etc.), do not embed it inline. Instead:

1. Extract to `~/.config/opencode/plans/<repo>/<section-name>.md`
2. Replace the section with one line in AGENTS.md:

```markdown
> CI/CD architecture details: see `~/.config/opencode/plans/<repo-name>/ci-architecture.md`
```

The agent reads it on demand. This keeps AGENTS.md under ~150 lines (target: ~1,000 tokens).

### Canonical example

`bluefin-lts` AGENTS.md at `~/src/bluefin-lts/AGENTS.md` is the reference implementation:
- 124 lines
- Lazy-loads CI architecture, build architecture, development tasks to `~/.config/opencode/plans/bluefin-lts/`
- Contains only project-specific content

### Never-commit rule

```
AGENTS.md committed to: castrojo/<repo> fork, main branch ✅
AGENTS.md in upstream PR:                                   ❌ NEVER
AGENTS.md pushed to upstream remote:                        ❌ NEVER
```

If the upstream repo has its own AGENTS.md, it is treated as reference only — per the authority hierarchy in `~/.config/opencode/AGENTS.md`.

---

## Quick Reference

```bash
# Step 1: Verify remotes
git remote -v

# Step 2: Plans directory
mkdir -p ~/.config/opencode/plans/<repo>/

# Step 3: Discover validation
just --list || make help

# Step 6: Commit fork AGENTS.md
git add AGENTS.md && git commit -m "chore: add AGENTS.md for AI-assisted workflow"
git push origin main   # fork only — NEVER upstream

# Step 8: Validate
just check && just lint   # or project equivalent
```

Full git workflow reference: `~/.config/opencode/plans/git-workflow.md`
