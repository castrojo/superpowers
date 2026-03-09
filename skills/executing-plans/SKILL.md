---
name: executing-plans
description: Use when you have a written implementation plan to execute in a separate session with review checkpoints
---

# Executing Plans

## Overview

Load plan, review critically, execute tasks in batches, report for review between batches.

**Core principle:** Batch execution with checkpoints for architect review.

**Announce at start:** "I'm using the executing-plans skill to implement this plan."

## The Process

### Step 1: Load and Review Plan
1. Read plan file
2. Run `plan-self-review` as a subagent: scores plan 100pt, produces deficiency checklist, edits plan **inline** to resolve all issues. Do not skip — unreviewed plans accumulate hidden gaps.
3. Run `architecture-review` as a subagent: checks structural issues. Resolve all critical/high items inline before execution begins.
4. Review critically - identify any remaining questions or concerns
5. Present your review summary to the user (open questions, risks, decisions needed)
6. **MANDATORY STOP:** Ask explicitly: "Ready to proceed with the first batch, or do you want to adjust the plan first?"
7. **Do NOT create TodoWrite. Do NOT touch any file. Wait for explicit go-ahead.**

### Step 2: Execute Batch
**Default: First 3 tasks**

For each task:
1. Mark as in_progress
2. Follow each step exactly (plan has bite-sized steps)
3. Run verifications as specified
4. Mark as completed

### Step 3: Report
When batch complete:
- Show what was implemented
- Show verification output
- Say: "Ready for feedback."

### Step 4: Continue
Based on feedback:
- Apply changes if needed
- Execute next batch
- Repeat until complete

### Step 5: Complete Development

After all tasks complete and verified:
- Announce: "I'm using the finishing-a-development-branch skill to complete this work."
- **REQUIRED SUB-SKILL:** Use superpowers:finishing-a-development-branch
- Follow that skill to verify tests, present options, execute choice

## When to Stop and Ask for Help

**STOP executing immediately when:**
- Hit a blocker mid-batch (missing dependency, test fails, instruction unclear)
- Plan has critical gaps preventing starting
- You don't understand an instruction
- Verification fails repeatedly

**Ask for clarification rather than guessing.**

## When to Revisit Earlier Steps

**Return to Review (Step 1) when:**
- Partner updates the plan based on your feedback
- Fundamental approach needs rethinking

**Don't force through blockers** - stop and ask.

## Remember
- Review plan critically first
- **ALWAYS stop and confirm before executing — even if the plan looks perfect**
- Follow plan steps exactly
- Don't skip verifications
- Reference skills when plan says to
- Between batches: just report and wait
- Stop when blocked, don't guess
- Never start implementation on main/master branch without explicit user consent

## Red Flags — You Are About to Skip the Confirmation

| Thought | Reality |
|---|---|
| "The plan looks good, no concerns" | Confirmation is not about concerns. It is unconditional. |
| "User said 'continue' or 'proceed'" | Means resume the workflow. Step 1 still requires explicit go-ahead. |
| "I'll just start the first task" | No. Stop. Ask first. Always. |
| "The user is clearly ready" | Let the user say that. Don't infer it. |

## Integration

**Required workflow skills:**
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
- **superpowers:writing-plans** - Creates the plan this skill executes
- **superpowers:finishing-a-development-branch** - Complete development after all tasks
