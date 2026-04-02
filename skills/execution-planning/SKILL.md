---
name: execution-planning
description: "Use when you have an approved design or requirements for a multi-step task, before touching code. This version adds lenses design/plan review and auto-fix loops before implementation."
---

# Execution Planning

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for the codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

This version assumes you already have:

- an approved design synthesis
- an inquiry record containing the user's raw input, generated questions, and answers

Before handing the plan off to implementation, you MUST run a **lenses design/plan review** via `codex exec`. Codex only reviews. **You** fix the design and/or plan yourself. Repeat until no **Critical** or **Important** issues remain, or until Codex says the only safe next step is to ask the human grouped questions.

Announce at start: **"I'm using the execution-planning skill to create the implementation plan."**

## Required Inputs

You MUST read and use all of these:

- `docs/plans/YYYY-MM-DD-<topic>-inquiry-record.md`
- `docs/plans/YYYY-MM-DD-<topic>-design-synthesis.md`
- relevant repo/project context
- `.maieutics/lenses.json` if present, otherwise `../inquiry/references/lenses.default.json`

The inquiry record is authoritative. If the design conflicts with it, update the design. Do NOT plan against stale assumptions.

## Output Files

**Plan:**
- Save to `docs/plans/YYYY-MM-DD-<topic>-execution-plan.md`

**Review record:**
- Save to `docs/plans/YYYY-MM-DD-<topic>-plan-review-record.md`
- Start from [references/plan-review-record-template.md](references/plan-review-record-template.md)

## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Load authoritative inputs** — inquiry record, design synthesis, lenses config, repo context
2. **Write the first draft of the plan** — exact files, commands, tests, commits
3. **Save the plan draft** — use the required filename
4. **Run design/plan review via codex exec** — use [design-plan-reviewer-prompt.md](design-plan-reviewer-prompt.md), passing file paths to the inquiry record, design synthesis, execution plan, and lenses config
5. **Fix issues yourself** — update design and/or plan when the reviewer returns Critical or Important issues that can be resolved from existing context
6. **Ask grouped user questions when truly needed** — if the reviewer says the issue cannot be resolved without the human, ask the user all pending blocker questions in one message, append answers to the inquiry record, then update design and/or plan
7. **Repeat the review loop** — continue until no Critical or Important issues remain
8. **Commit design, plan, and review record** — they are part of the durable workflow state
9. **Offer execution choice** — delegated-execution in this session or guided-execution in a separate session

## Bite-Sized Task Granularity

Each step is one action (2-5 minutes):

- "Write the failing test" — step
- "Run it to make sure it fails" — step
- "Implement the minimal code to make the test pass" — step
- "Run the tests and make sure they pass" — step
- "Commit" — step

## Execution Plan Header

Every plan MUST start with this header:

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use guided-execution to implement this plan task-by-task.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

**Authoritative Inputs:**
- Inquiry Record: `docs/plans/YYYY-MM-DD-<topic>-inquiry-record.md`
- Approved Design Synthesis: `docs/plans/YYYY-MM-DD-<topic>-design-synthesis.md`

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

## Lenses Design/Plan Review

Use `codex exec` so the main Claude Code planning session stays focused. Codex reads the files directly — pass file paths, not file contents.

### How to Run the Review

1. Ensure the inquiry record, design synthesis, and plan draft are all saved to disk with the latest changes
2. Run `codex exec` with the prompt from [design-plan-reviewer-prompt.md](design-plan-reviewer-prompt.md), substituting:
   - `[INQUIRY_RECORD_PATH]` → actual path (e.g. `docs/plans/2026-03-10-slack-bot-inquiry-record.md`)
   - `[DESIGN_SYNTHESIS_PATH]` → actual path (e.g. `docs/plans/2026-03-10-slack-bot-design-synthesis.md`)
   - `[EXECUTION_PLAN_PATH]` → actual path (e.g. `docs/plans/2026-03-10-slack-bot-execution-plan.md`)
   - `[LENSES_CONFIG_PATH]` → `.maieutics/lenses.json` or the bundled default
   - `Project Context Summary` → a short summary of the repo
3. Parse the returned JSON and act on the result

#### Exact Command

**IMPORTANT: Run this command in the foreground (do NOT use `run_in_background`).** Foreground execution keeps `RUN_ID` in scope so you can read the correct output file immediately after completion.

```bash
RUN_ID=$(uuidgen) && codex exec --full-auto -s read-only -o /tmp/maieutics-review-${RUN_ID}.json - <<'PROMPT'
<substituted prompt content from design-plan-reviewer-prompt.md>
PROMPT
echo "OUTPUT_FILE=/tmp/maieutics-review-${RUN_ID}.json"
```

- `RUN_ID=$(uuidgen)` — generates a unique ID per invocation to avoid stale file collisions across sessions
- `--full-auto` — non-interactive execution with sandboxed auto-approval
- `-s read-only` — reviewer only reads files, never writes
- `-o /tmp/maieutics-review-${RUN_ID}.json` — saves the last message to a uniquely-named file for reliable parsing
- `-` — read prompt from stdin (use heredoc)

Parse the output from the `-o` file, not from stdout (stdout contains progress logs). Generate a new `RUN_ID` for each invocation (including retries and subsequent review rounds).

**NEVER use glob patterns (e.g. `ls /tmp/maieutics-review-*.json`) to locate the output file.** Always use the exact `${RUN_ID}` path printed at the end of the command.

### Review Loop Limit

The review loop runs **at most 3 rounds**. If Critical or Important issues remain after 3 rounds, stop self-fixing and escalate to the user:

- Present the unresolved issues clearly
- Ask the user for direction on each remaining issue
- Append answers to the inquiry record
- Apply the user's decisions, then run one final review

This prevents infinite fix loops where each fix introduces new issues.

### Review Outcomes

**If status is `approved`:**
- Continue to execution handoff

**If status is `needs-fix`:**
- Fix the design and/or plan yourself
- Append the review round to the review record
- Re-run the external review (respecting the 3-round limit)

**If status is `needs-user-input`:**
- Ask the user the grouped blocker questions
- Append answers verbatim to the inquiry record
- Update design and/or plan
- Re-run the external review

### Blocking Rule

You MUST NOT hand the plan off for implementation while any **Critical** or **Important** review issue remains unresolved.

**Minor** issues may be recorded and optionally cleaned up, but they do not block execution.

## Remember

- Exact file paths always
- Complete code in the plan (not "add validation")
- Exact commands with expected output
- Inquiry record is authoritative
- Codex only reviews; it does not fix
- DRY, YAGNI, TDD, frequent commits

## Execution Handoff

After saving the plan and passing review, offer execution choice:

**"Plan complete and saved to `docs/plans/<topic>-execution-plan.md`. The design/plan review loop is clean. Two execution options:**

1. **Delegated Execution (this session)** — fresh implementation worker per task, standard per-task review, then a final lenses implementation review
2. **Parallel Session (separate)** — open a new session with `guided-execution`

**Which approach?"**

If **Delegated Execution** chosen:
- **REQUIRED SUB-SKILL:** Use `delegated-execution`
- Stay in this session

If **Parallel Session** chosen:
- Guide them to open a new session in the worktree
- **REQUIRED SUB-SKILL:** New session uses `guided-execution`
