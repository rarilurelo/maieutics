---
name: writing-plans
description: "Use when you have an approved design or requirements for a multi-step task, before touching code. This version adds multi-perspective design/plan review and auto-fix loops before implementation."
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for the codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

This version assumes you already have:

- an approved design doc
- a discovery log containing the user's raw input, generated questions, and answers

Before handing the plan off to implementation, you MUST run a **multi-perspective design/plan review** via `codex exec`. Codex only reviews. **You** fix the design and/or plan yourself. Repeat until no **Critical** or **Important** issues remain, or until Codex says the only safe next step is to ask the human grouped questions.

Announce at start: **"I'm using the writing-plans skill to create the implementation plan."**

## Required Inputs

You MUST read and use all of these:

- `docs/plans/YYYY-MM-DD-<topic>-discovery.md`
- `docs/plans/YYYY-MM-DD-<topic>-design.md`
- relevant repo/project context
- `.maieutics/multi-perspective.json` if present, otherwise `../brainstorming/references/multi-perspective.default.json`

The discovery log is authoritative. If the design conflicts with it, update the design. Do NOT plan against stale assumptions.

## Output Files

**Plan:**
- Save to `docs/plans/YYYY-MM-DD-<topic>-plan.md`

**Review log:**
- Save to `docs/plans/YYYY-MM-DD-<topic>-design-plan-review.md`
- Start from [references/design-plan-review-log-template.md](references/design-plan-review-log-template.md)

## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Load authoritative inputs** — discovery log, design doc, perspective config, repo context
2. **Write the first draft of the plan** — exact files, commands, tests, commits
3. **Save the plan draft** — use the required filename
4. **Run design/plan review via codex exec** — use [design-plan-reviewer-prompt.md](design-plan-reviewer-prompt.md), passing file paths to the discovery log, design doc, plan doc, and perspective config
5. **Fix issues yourself** — update design and/or plan when the reviewer returns Critical or Important issues that can be resolved from existing context
6. **Ask grouped user questions when truly needed** — if the reviewer says the issue cannot be resolved without the human, ask the user all pending blocker questions in one message, append answers to the discovery log, then update design and/or plan
7. **Repeat the review loop** — continue until no Critical or Important issues remain
8. **Commit design, plan, and review log** — they are part of the durable workflow state
9. **Offer execution choice** — subagent-driven in this session or executing-plans in a separate session

## Bite-Sized Task Granularity

Each step is one action (2-5 minutes):

- "Write the failing test" — step
- "Run it to make sure it fails" — step
- "Implement the minimal code to make the test pass" — step
- "Run the tests and make sure they pass" — step
- "Commit" — step

## Plan Document Header

Every plan MUST start with this header:

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use executing-plans to implement this plan task-by-task.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

**Authoritative Inputs:**
- Discovery Log: `docs/plans/YYYY-MM-DD-<topic>-discovery.md`
- Approved Design: `docs/plans/YYYY-MM-DD-<topic>-design.md`

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

## Multi-Perspective Design/Plan Review

Use `codex exec` so the main Claude Code planning session stays focused. Codex reads the files directly — pass file paths, not file contents.

### How to Run the Review

1. Ensure the discovery log, design doc, and plan draft are all saved to disk with the latest changes
2. Run `codex exec` with the prompt from [design-plan-reviewer-prompt.md](design-plan-reviewer-prompt.md), substituting:
   - `[DISCOVERY_LOG_PATH]` → actual path (e.g. `docs/plans/2026-03-10-slack-bot-discovery.md`)
   - `[DESIGN_DOC_PATH]` → actual path (e.g. `docs/plans/2026-03-10-slack-bot-design.md`)
   - `[PLAN_DOC_PATH]` → actual path (e.g. `docs/plans/2026-03-10-slack-bot-plan.md`)
   - `[PERSPECTIVE_CONFIG_PATH]` → `.maieutics/multi-perspective.json` or the bundled default
   - `Project Context Summary` → a short summary of the repo
3. Parse the returned JSON and act on the result

#### Exact Command

```bash
codex exec --full-auto -s read-only -o /tmp/maieutics-review.json - <<'PROMPT'
<substituted prompt content from design-plan-reviewer-prompt.md>
PROMPT
```

- `--full-auto` — non-interactive execution with sandboxed auto-approval
- `-s read-only` — reviewer only reads files, never writes
- `-o /tmp/maieutics-review.json` — saves the last message for reliable parsing
- `-` — read prompt from stdin (use heredoc)

Parse the output from the `-o` file, not from stdout (stdout contains progress logs).

### Review Loop Limit

The review loop runs **at most 3 rounds**. If Critical or Important issues remain after 3 rounds, stop self-fixing and escalate to the user:

- Present the unresolved issues clearly
- Ask the user for direction on each remaining issue
- Append answers to the discovery log
- Apply the user's decisions, then run one final review

This prevents infinite fix loops where each fix introduces new issues.

### Review Outcomes

**If status is `approved`:**
- Continue to execution handoff

**If status is `needs-fix`:**
- Fix the design and/or plan yourself
- Append the review round to the review log
- Re-run the external review (respecting the 3-round limit)

**If status is `needs-user-input`:**
- Ask the user the grouped blocker questions
- Append answers verbatim to the discovery log
- Update design and/or plan
- Re-run the external review

### Blocking Rule

You MUST NOT hand the plan off for implementation while any **Critical** or **Important** review issue remains unresolved.

**Minor** issues may be recorded and optionally cleaned up, but they do not block execution.

## Remember

- Exact file paths always
- Complete code in the plan (not "add validation")
- Exact commands with expected output
- Discovery log is authoritative
- Codex only reviews; it does not fix
- DRY, YAGNI, TDD, frequent commits

## Execution Handoff

After saving the plan and passing review, offer execution choice:

**"Plan complete and saved to `docs/plans/<topic>-plan.md`. The design/plan review loop is clean. Two execution options:**

1. **Subagent-Driven (this session)** — fresh implementation worker per task, standard per-task review, then multi-perspective final implementation review
2. **Parallel Session (separate)** — open a new session with `executing-plans`

**Which approach?"**

If **Subagent-Driven** chosen:
- **REQUIRED SUB-SKILL:** Use `subagent-driven-development`
- Stay in this session

If **Parallel Session** chosen:
- Guide them to open a new session in the worktree
- **REQUIRED SUB-SKILL:** New session uses `executing-plans`
