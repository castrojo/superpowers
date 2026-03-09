---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** This should be run in a dedicated worktree (created by brainstorming skill).

**Save plans to:** `~/.config/opencode/plans/<repo-name>/YYYY-MM-DD-<feature-name>.md`

**NEVER save plans inside the git repo** (no `docs/`, no `.planning/`, no `plans/` in repo root). Plans are personal workflow artifacts — they do not belong in version control.

## Bite-Sized Task Granularity

**Each step is one action (2-5 minutes):**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

**Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

**Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## Remember
- Exact file paths always
- Complete code in plan (not "add validation")
- Exact commands with expected output
- Reference relevant skills with @ syntax
- DRY, YAGNI, TDD, frequent commits

## Plan Review (Mandatory Before Execution)

After saving the plan file, **run both review skills before offering execution options.**
Do NOT skip this. Do NOT proceed to the execution handoff if either review finds blockers.

### Step 1: Document completeness — `plan-self-review`

**REQUIRED SUB-SKILL:** Use `plan-self-review`

Score the plan (100pt: clarity 25, comprehensiveness 25, feasibility 25, consistency 25).
Produce a prioritized deficiency checklist. Edit the plan to resolve all deficiencies.
Confirm no logical contradictions or missing elements (scope, risks, dependencies).

Re-save the plan file after any edits.

### Step 2: Architecture review — `architecture-review`

**REQUIRED SUB-SKILL:** Use `architecture-review`

Apply to the proposed code structure in the plan:
- Map directory/module boundaries being added or changed
- Check for circular dependencies, god modules, leaky abstractions
- Classify any issues: critical (blocks dev), high (maintenance burden), medium/low (friction)
- Document findings inline in the plan under a `## Known Issues / Required Fixes` section

Any critical or high severity issue MUST be resolved in the plan before proceeding.
Medium/low issues MUST be documented inline with a resolution note.

### Step 3: Annotate the plan

Add a header block after the plan's `---` separator documenting review results:

```markdown
## KNOWN ISSUES / REQUIRED FIXES

> Identified during pre-execution correctness review on YYYY-MM-DD.
> Each issue is annotated inline in the relevant task.
> Resolve each one as you reach that task — do not skip them.

### CRITICAL — will cause compile failure or silent runtime bug
...

### MODERATE — wrong props / wrong data shape / code won't work as described
...

### MINOR — cosmetic / easy to fix in passing
...

### INFORMATIONAL — important context for the executor, no code changes required
...
```

If no issues are found in a severity tier, omit that tier.

---

## Execution Handoff

After both reviews pass and the plan is updated:

### Step 1: Import plan to workflow-state DB and delete the markdown file (MANDATORY)

Plans are tracked in the workflow-state postgres DB — NOT in `opencode-config`. Any session
can query the DB directly via MCP. Committed markdown files are redundant state that will drift.

**Import each numbered task:**

```
workflow-state_import_plan(
  repo: "<repo>",
  plan_id: "<YYYY-MM-DD-feature>",
  tasks: "[{\"task_num\": 1, \"description\": \"...\"}, ...]"
)
```

**Then delete the markdown file:**

```bash
rm ~/.config/opencode/plans/<repo-name>/<plan-file>.md
```

Do NOT commit the markdown file to opencode-config. If there are uncommitted plan files in
`~/.config/opencode/plans/`, delete them after importing — never commit them.

### Step 2: Offer execution choice

**"Plan imported to DB (plan_id: `<plan-id>`). Two execution options:**

**1. Subagent-Driven (this session)** — invoke `loop-start` now, then `loop-task` per task with a fresh subagent each run; devaipod is the execution environment for build/validation; `loop-gate` after all runs; `loop-end` to close.

**2. Parallel Session (separate)** — open a new session with `executing-plans`; it loads the plan from DB, gates on confirmation, then routes to `loop-start → loop-task → loop-gate → loop-end`.

Both options run through the loop system. The difference is only session context.

**Which approach?"**

**If Subagent-Driven chosen:**
- Invoke `loop-start` immediately in this session
- **REQUIRED SUB-SKILL:** Use `loop-start`, then `loop-task` × N, then `loop-gate`, then `loop-end`

**If Parallel Session chosen:**
- Guide them to open new session in worktree
- **REQUIRED SUB-SKILL:** New session uses `superpowers:executing-plans`
