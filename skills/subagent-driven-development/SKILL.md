---
name: subagent-driven-development
description: "Use when executing implementation plans with independent tasks in the current session. This version adds authoritative discovery context and a final multi-perspective implementation review loop."
---

# Subagent-Driven Development

Execute the plan by dispatching a **fresh implementation worker per task**, with standard per-task review after each task:

1. spec compliance review
2. code quality review

After all planned tasks are complete, run a **final multi-perspective implementation review** via `codex exec`. If Codex finds Critical or Important issues, implementation continues until those issues are fixed. If Codex says the remaining blockers require the human, ask grouped user questions, update the authoritative artifacts, and continue.

## Core Principle

Fresh worker per task + standard per-task reviews + final multi-perspective implementation review = higher quality, less context pollution, better alignment with what the human actually asked for.

## Authoritative Inputs

Before executing any task, identify and keep track of:

- `docs/plans/YYYY-MM-DD-<topic>-discovery.md`
- `docs/plans/YYYY-MM-DD-<topic>-design.md`
- `docs/plans/YYYY-MM-DD-<topic>-plan.md`
- Perspective config: `.maieutics/multi-perspective.json` if present, otherwise `../brainstorming/references/multi-perspective.default.json`

The discovery log and user answers are authoritative. If the plan later proves inconsistent with them, update the plan and implement against the corrected version.

## When to Use

Use this when:

- you have an implementation plan
- tasks are mostly independent
- you want to stay in the current session
- you want automatic review loops between tasks and before finishing the branch

If tasks are tightly coupled or you want human checkpoints between batches, use `executing-plans` instead.

## Process

### Initial Setup

1. Read the plan file once
2. Extract all tasks with full text and context
3. Load the discovery log and approved design
4. Create TodoWrite entries for all tasks
5. Start an `Accumulated Discoveries` block for facts learned during implementation that later tasks should inherit

### Per Task Loop

For each task:

1. Dispatch a fresh implementation worker using [implementer-prompt.md](implementer-prompt.md)
2. Give the worker:
   - the full task text
   - relevant discovery log excerpts
   - relevant design context
   - accumulated discoveries from earlier tasks
   - repo context needed for the task
3. If the worker asks questions, answer them before allowing work to continue
4. Worker implements, tests, commits, self-reviews
5. Dispatch spec compliance review using [spec-reviewer-prompt.md](spec-reviewer-prompt.md)
6. If spec review fails, send the issues back to the implementation worker and repeat until spec compliant
7. Dispatch code quality review using [code-quality-reviewer-prompt.md](code-quality-reviewer-prompt.md)
8. If code quality review finds blocking issues, send them back to the implementation worker and repeat until clean
9. Record any durable discoveries from the worker's report and carry them into later tasks
10. Mark the task complete in TodoWrite

### Final Multi-Perspective Implementation Review

After all plan tasks are complete:

1. Save or update `docs/plans/YYYY-MM-DD-<topic>-implementation-review.md` using [references/implementation-review-log-template.md](references/implementation-review-log-template.md)
2. Run `codex exec` with the prompt from [implementation-reviewer-prompt.md](implementation-reviewer-prompt.md), substituting:
   - `[DISCOVERY_LOG_PATH]` → actual path to the discovery log
   - `[DESIGN_DOC_PATH]` → actual path to the design doc
   - `[PLAN_DOC_PATH]` → actual path to the plan doc
   - `[PERSPECTIVE_CONFIG_PATH]` → `.maieutics/multi-perspective.json` or the bundled default
   - `Accumulated Discoveries` → bullet list of durable discoveries from task workers
   - `Current Implementation State` → branch context, diff summary, test results
3. Parse the returned JSON and act on the result

### Final Review Outcomes

**If status is `approved`:**
- Continue to `finishing-a-development-branch`

**If status is `needs-fix`:**
- Convert the issues into follow-up implementation work
- If the findings reveal a design/plan mismatch, update the design and/or plan first
- Dispatch the implementation worker to fix the issues
- Re-run the final multi-perspective implementation review

**If status is `needs-user-input`:**
- Ask the user all blocker questions in one grouped message
- Append the answers to the discovery log
- Update design and/or plan if needed
- Implement the resulting changes
- Re-run the final multi-perspective implementation review

## Blocking Rule

You MUST NOT finish the branch while any **Critical** or **Important** issue from the final multi-perspective implementation review remains unresolved.

**Minor** issues may be recorded, but they do not block completion.

## Prompt Templates

- [implementer-prompt.md](implementer-prompt.md) — dispatch implementation worker
- [spec-reviewer-prompt.md](spec-reviewer-prompt.md) — dispatch spec compliance reviewer
- [code-quality-reviewer-prompt.md](code-quality-reviewer-prompt.md) — dispatch code quality reviewer
- [implementation-reviewer-prompt.md](implementation-reviewer-prompt.md) — dispatch final multi-perspective implementation reviewer

## Advantages

### vs. Manual Execution

- Fresh context per task
- Better TDD discipline
- Durable discoveries can be passed forward
- Clarifying questions happen before or during work, not after
- Final implementation gets checked against discovery, design, and plan — not just task text

### vs. Executing Plans

- Same session, faster iteration
- Automatic review checkpoints
- Final whole-feature review before branch completion

## Red Flags

Never do these:

- Start implementation without a plan
- Ignore the discovery log or treat it as optional
- Skip spec review or code quality review for a task
- Skip the final multi-perspective implementation review
- Let Codex edit code directly
- Proceed with unresolved Critical or Important review issues
- Make workers read the plan file themselves when you can provide the exact task text
- Ignore worker questions or reviewer escalation questions
- Finish the branch before the final review loop is clean
