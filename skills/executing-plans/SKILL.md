---
name: executing-plans
description: Use when you have a written implementation plan to execute in a separate session with review checkpoints
---

# Executing Plans

## Overview

Thin routing skill. Loads plan from DB, presents a light review, gates on explicit user confirmation, then hands off to the loop system. Does NOT execute tasks directly.

**Core principle:** The loop system (loop-start → loop-task × N → loop-gate → loop-end) is the execution engine. This skill is only the on-ramp.

**Announce at start:** "I'm using the executing-plans skill to load the plan and route to the loop."

---

## Step 1: Load plan from DB

Plans live in the workflow-state DB — not in files. Never use Read or Bash to load a plan.

```
workflow-state_get_plan_tasks(repo: "<REPO>", plan_id: "<plan_id>")
```

If `plan_id` is unknown, ask the user: "What is the plan_id? (format: YYYY-MM-DD-feature-name)"

Display the task list:
```
Plan: <plan_id>
Tasks: <N> total
  [ ] 1. <description>
  [ ] 2. <description>
  ...
```

---

## Step 2: Light review

`plan-self-review` and `architecture-review` were already run during `writing-plans`. Do NOT re-run them.

Instead, do a quick sanity check — read the task list and flag only:
- Tasks that reference files or APIs that no longer exist
- Tasks whose dependencies are clearly wrong (e.g. task 3 requires output from task 5)
- Any task description so vague it cannot be executed without guessing

If nothing is flagged: say "Plan looks clean." and proceed.
If issues are found: list them and ask the user to clarify before proceeding.

---

## Step 3: MANDATORY STOP

Use the `question` tool — do NOT touch any file until the user explicitly confirms:

```
question: "Plan loaded (<N> tasks). Ready to set up the worktree and start the loop?"
options:
  - "Yes — set up worktree and start loop (Recommended)"
  - "No — I want to review or adjust the plan first"
  - "No — start loop without a new worktree (already on the right branch)"
```

**Do NOT create any TodoWrite. Do NOT touch any file. Wait for explicit go-ahead.**

If the user wants to adjust the plan: help them update the DB tasks via `workflow-state_update_task_status` or by re-importing a revised task list via `workflow-state_import_plan`, then return to Step 3.

---

## Step 4: Set up isolated workspace (if worktree option chosen)

**REQUIRED SUB-SKILL:** Use `superpowers:using-git-worktrees`

Follow that skill to create an isolated branch and verify the clean baseline.

If the user chose "start loop without a new worktree": skip this step.

---

## Step 5: Route to loop-start

**Invoke `loop-start` now.** This is not optional and not deferred — execute it in this same response.

The loop system handles all execution from here:
- `loop-start` → initializes DB state, confirms run count with user, sets goal
- `loop-task` × N → each run dispatches a subagent; devaipod is the execution environment for build/validation tasks
- `loop-gate` → after all runs in a phase: processes [GAP] findings, gates phase transition
- `loop-end` → after all phases: backport review, integrity checklist, state reset

Do not return to this skill after handing off to loop-start. The loop skills own execution from here.

---

## When to Stop and Ask

**STOP immediately when:**
- Plan cannot be loaded from DB (plan_id not found)
- Plan has critical structural issues (Step 2 flags blockers)
- User asks to pause mid-loop (handle via loop-task's stop protocol)

**Never:**
- Execute tasks directly in this skill — that is loop-task's job
- Read plan files from disk — plans are in the DB
- Skip the MANDATORY STOP in Step 3
- Start implementation on main without explicit user consent

---

## Integration

**This skill is called from:**
- `writing-plans` (Execution Handoff → Parallel Session option)
- User explicitly opening a new session to execute a saved plan

**This skill calls:**
- `superpowers:using-git-worktrees` — isolated workspace (Step 4)
- `loop-start` — hands off execution to the loop system (Step 5)

**This skill does NOT call:**
- `plan-self-review` — already done during writing-plans
- `architecture-review` — already done during writing-plans
- `finishing-a-development-branch` — called by loop-end if a PR is needed
