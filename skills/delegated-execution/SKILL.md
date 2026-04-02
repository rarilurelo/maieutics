---
name: delegated-execution
description: "Use when executing implementation plans with independent tasks in the current session. This version adds authoritative inquiry context and a final lenses implementation review loop."
---

# Delegated Execution

Execute the plan by dispatching a **fresh implementation worker per task**, with standard per-task review after each task:

1. spec compliance review
2. code quality review

After all planned tasks are complete, run a **final lenses implementation review** via `codex exec`. If Codex finds Critical or Important issues, implementation continues until those issues are fixed. If Codex says the remaining blockers require the human, ask grouped user questions, update the authoritative artifacts, and continue.

## Core Principle

Fresh worker per task + standard per-task reviews + final lenses implementation review = higher quality, less context pollution, better alignment with what the human actually asked for.

## Authoritative Inputs

Before executing any task, identify and keep track of:

- `docs/plans/YYYY-MM-DD-<topic>-inquiry-record.md`
- `docs/plans/YYYY-MM-DD-<topic>-design-synthesis.md`
- `docs/plans/YYYY-MM-DD-<topic>-execution-plan.md`
- Lenses config: `.maieutics/lenses.json` if present, otherwise `../inquiry/references/lenses.default.json`

The inquiry record and user answers are authoritative. If the plan later proves inconsistent with them, update the plan and implement against the corrected version.

## When to Use

Use this when:

- you have an implementation plan
- tasks are mostly independent
- you want to stay in the current session
- you want automatic review loops between tasks and before finishing the branch

If tasks are tightly coupled or you want human checkpoints between batches, use `guided-execution` instead.

## Process

### Initial Setup

1. Read the plan file once
2. Extract all tasks with full text and context
3. Load the inquiry record and approved design synthesis
4. Create TodoWrite entries for all tasks
5. Start an `Carried Findings` block for facts learned during implementation that later tasks should inherit

### Per Task Loop

For each task:

1. Dispatch a fresh implementation worker using [implementer-prompt.md](implementer-prompt.md)
2. Give the worker:
   - the full task text
   - relevant inquiry record excerpts
   - relevant design context
   - carried findings from earlier tasks
   - repo context needed for the task
3. If the worker asks questions, answer them before allowing work to continue
4. Worker implements, tests, commits, self-reviews
5. Dispatch spec compliance review using [spec-reviewer-prompt.md](spec-reviewer-prompt.md)
6. If spec review fails, send the issues back to the implementation worker and repeat until spec compliant
7. Dispatch code quality review using [code-quality-reviewer-prompt.md](code-quality-reviewer-prompt.md)
8. If code quality review finds blocking issues, send them back to the implementation worker and repeat until clean
9. Record any durable discoveries from the worker's report and carry them into later tasks
10. Mark the task complete in TodoWrite

### Final Lenses Implementation Review

After all plan tasks are complete:

1. Save or update `docs/plans/YYYY-MM-DD-<topic>-implementation-review-record.md` using [references/implementation-review-record-template.md](references/implementation-review-record-template.md)
2. Run `codex exec` with the prompt from [implementation-reviewer-prompt.md](implementation-reviewer-prompt.md), substituting:
   - `[INQUIRY_RECORD_PATH]` → actual path to the inquiry record
   - `[DESIGN_SYNTHESIS_PATH]` → actual path to the design synthesis
   - `[EXECUTION_PLAN_PATH]` → actual path to the execution plan
   - `[LENSES_CONFIG_PATH]` → `.maieutics/lenses.json` or the bundled default
   - `Carried Findings` → bullet list of durable discoveries from task workers
   - `Current Implementation State` → branch context, diff summary, test results
3. Parse the returned JSON and act on the result

#### Exact Command

**IMPORTANT: Run this command in the foreground (do NOT use `run_in_background`).** Foreground execution keeps `RUN_ID` in scope so you can read the correct output file immediately after completion.

```bash
RUN_ID=$(uuidgen) && codex exec --full-auto -s read-only -o /tmp/maieutics-impl-review-${RUN_ID}.json - <<'PROMPT'
<substituted prompt content from implementation-reviewer-prompt.md>
PROMPT
echo "OUTPUT_FILE=/tmp/maieutics-impl-review-${RUN_ID}.json"
```

- `RUN_ID=$(uuidgen)` — generates a unique ID per invocation to avoid stale file collisions across sessions
- `--full-auto` — non-interactive execution with sandboxed auto-approval
- `-s read-only` — reviewer only reads files, never writes
- `-o /tmp/maieutics-impl-review-${RUN_ID}.json` — saves the last message to a uniquely-named file for reliable parsing
- `-` — read prompt from stdin (use heredoc)

Parse the output from the `-o` file, not from stdout (stdout contains progress logs). Generate a new `RUN_ID` for each invocation (including retries and subsequent review rounds).

**NEVER use glob patterns (e.g. `ls /tmp/maieutics-impl-review-*.json`) to locate the output file.** Always use the exact `${RUN_ID}` path printed at the end of the command.

### Review Loop Limit

The final review loop runs **at most 3 rounds**. If Critical or Important issues remain after 3 rounds, stop self-fixing and escalate to the user:

- Present the unresolved issues clearly
- Ask the user for direction on each remaining issue
- Append answers to the inquiry record
- Apply the user's decisions, then run one final review

This prevents infinite fix loops where each fix introduces new issues.

### Final Review Outcomes

**If status is `approved`:**
- Continue to `closing-the-branch`

**If status is `needs-fix`:**
- Convert the issues into follow-up implementation work
- If the findings reveal a design/plan mismatch, update the design and/or plan first
- Dispatch the implementation worker to fix the issues
- Re-run the final lenses implementation review (respecting the 3-round limit)

**If status is `needs-user-input`:**
- Ask the user all blocker questions in one grouped message
- Append the answers to the inquiry record
- Update design and/or plan if needed
- Implement the resulting changes
- Re-run the final lenses implementation review

## Blocking Rule

You MUST NOT finish the branch while any **Critical** or **Important** issue from the final lenses implementation review remains unresolved.

**Minor** issues may be recorded, but they do not block completion.

## Prompt Templates

- [implementer-prompt.md](implementer-prompt.md) — dispatch implementation worker
- [spec-reviewer-prompt.md](spec-reviewer-prompt.md) — dispatch spec compliance reviewer
- [code-quality-reviewer-prompt.md](code-quality-reviewer-prompt.md) — dispatch code quality reviewer
- [implementation-reviewer-prompt.md](implementation-reviewer-prompt.md) — dispatch final lenses implementation reviewer

## Advantages

### vs. Manual Execution

- Fresh context per task
- Better TDD discipline
- Durable carried findings can be passed forward
- Clarifying questions happen before or during work, not after
- Final implementation gets checked against inquiry, design, and plan — not just task text

### vs. Guided Execution

- Same session, faster iteration
- Automatic review checkpoints
- Final whole-feature review before branch completion

## Red Flags

Never do these:

- Start implementation without a plan
- Ignore the inquiry record or treat it as optional
- Skip spec review or code quality review for a task
- Skip the final lenses implementation review
- Let Codex edit code directly
- Proceed with unresolved Critical or Important review issues
- Make workers read the plan file themselves when you can provide the exact task text
- Ignore worker questions or reviewer escalation questions
- Finish the branch before the final review loop is clean
