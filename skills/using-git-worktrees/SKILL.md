---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - creates isolated git worktrees with smart directory selection and safety verification
---

# Using Git Worktrees

## Overview

Git worktrees create isolated workspaces sharing the same repository, allowing work on multiple branches simultaneously without switching.

**Core principle:** Call `worktree_create` — the plugin handles the rest. A new terminal opens with OpenCode already running in the worktree.

**Announce at start:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## How It Works

The `opencode-worktree` plugin is globally installed. It provides two tools:

| Tool | Purpose |
|------|---------|
| `worktree_create(branch, baseBranch?)` | Creates worktree, syncs files, runs hooks, forks session, spawns terminal with OpenCode |
| `worktree_delete(reason)` | Commits snapshot, removes worktree, cleans up session |

Worktrees are stored at `~/.local/share/opencode/worktree/<project-id>/<branch>/` — outside the repository, so no `.gitignore` concerns.

## Creation Steps

### 1. Call `worktree_create`

```
worktree_create(branch: "feature/my-feature", baseBranch?: "main")
```

The plugin will:
1. Create the git worktree at `~/.local/share/opencode/worktree/<project-id>/<branch>`
2. Copy files and symlink dirs per `.opencode/worktree.jsonc` (auto-created on first use)
3. Run `postCreate` hooks (e.g. `pnpm install`) if configured
4. Fork the current session with context (plan, delegations)
5. Spawn a new terminal window/tmux window with OpenCode running

### 2. Offer to Open VS Code

Use the `question` tool to ask the user via the TUI selector:

- Question: "Open VS Code in the worktree so you can follow along?"
- Header: "Open VS Code?"
- Options:
  - `Yes` — "Open VS Code in the worktree now"
  - `No` — "Skip, I'll open it myself if needed"
- `custom: false` (selector only, no free-text)

If the user selects **Yes**, run:
```bash
code <worktreePath>
```

### 2b. Offer to Open Browser (web projects only)

If the project has a dev server (detected by `package.json` with `dev`/`start` script, `Makefile` with a `serve` target, etc.), use the `question` tool to ask:

- Question: "Open a browser with the live site running in hot reload?"
- Header: "Open browser?"
- Options:
  - `Yes, start dev server + open browser` — "Start the dev server and open the site in the browser"
  - `No` — "Skip"
- `custom: false`

If the user selects **Yes**:
1. Start the dev server in the background (e.g. `npm run dev &`, `pnpm dev &`) inside the worktree
2. Wait a moment for it to bind to a port, then open the URL in the default browser:
   ```bash
   xdg-open http://localhost:<port>   # Linux
   open http://localhost:<port>        # macOS
   ```
   Detect the port from the dev server output or fall back to common defaults (3000, 5173, 4321, 8080).

### 3. Run Project Setup (in new terminal)

After the terminal spawns, auto-detect and run setup if not already handled by hooks:

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

### 4. Verify Clean Baseline

Run tests to confirm the worktree starts clean:

```bash
# Use project-appropriate command
npm test / cargo test / pytest / go test ./... / just check
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Report ready.

### 5. Report Location

```
Worktree ready at ~/.local/share/opencode/worktree/<project-id>/<branch>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Cleanup

When work is complete, call `worktree_delete` — do NOT use `git worktree remove` directly:

```
worktree_delete(reason: "Feature complete")
```

The plugin will commit all uncommitted changes with a snapshot message, remove the worktree, and clean up the session.

## Quick Reference

| Situation | Action |
|-----------|--------|
| Starting a feature / branch work | `worktree_create(branch)` |
| Creating from specific base | `worktree_create(branch, baseBranch)` |
| Work complete | `worktree_delete(reason)` |
| Tests fail at baseline | Report failures + ask before proceeding |
| Project needs `.env` sync | Configure `.opencode/worktree.jsonc` |
| Project needs `node_modules` symlink | Configure `.opencode/worktree.jsonc` |

## Project Config (`.opencode/worktree.jsonc`)

Auto-created on first `worktree_create`. Common patterns:

```jsonc
{
  "sync": {
    "copyFiles": [".env", ".env.local"],   // copied per worktree
    "symlinkDirs": ["node_modules"]         // symlinked to save space
  },
  "hooks": {
    "postCreate": ["pnpm install"],
    "preDelete": ["docker compose down"]
  }
}
```

## Red Flags

**Never:**
- Call `git worktree add` directly — use `worktree_create`
- Call `git worktree remove` directly — use `worktree_delete`
- Skip baseline test verification
- Proceed with failing tests without asking

**Always:**
- Use `worktree_create` so the terminal spawns automatically
- Use `worktree_delete` so the snapshot commit and cleanup happen correctly
- Auto-detect and run project setup if not covered by hooks
- Verify clean test baseline

## Integration

**Called by:**
- **brainstorming** (Phase 4) - REQUIRED when design is approved and implementation follows
- **subagent-driven-development** - REQUIRED before executing any tasks
- **executing-plans** - REQUIRED before executing any tasks
- Any skill needing isolated workspace

**Pairs with:**
- **finishing-a-development-branch** - REQUIRED for cleanup after work complete
