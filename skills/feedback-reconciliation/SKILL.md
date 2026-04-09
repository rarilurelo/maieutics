---
name: feedback-reconciliation
description: "Use when external feedback (staging tests, UAT, code review, bug reports) arrives after implementation. Triages each item via lenses classification, promotes accepted changes to the Inquiry Record, and routes to the appropriate pipeline stage."
---

# Reconciling External Feedback

## Overview

Triage external feedback and route it back into the maieutics pipeline. Feedback from staging tests, UAT, code review, or bug reports is **not automatically authoritative**. Each item is classified by layer and disposition via `codex exec`, and only items explicitly accepted by the user are promoted to the Inquiry Record.

This skill does NOT create a new worktree. It operates in the existing feature branch worktree (or prompts the user to re-enter one via `using-git-worktrees`).

**Announce at start:** "I'm using the feedback-reconciliation skill to triage and route external feedback."

## Authority Model

Feedback is input, not authority. The authority order remains:

1. Latest explicit user answers in the inquiry record
2. Raw user input captured in the inquiry record
3. Approved design synthesis
4. Everything else

External feedback enters at level 4. Items promoted to the Inquiry Record after user acceptance move to level 1.

If feedback contradicts an existing authoritative decision, the user MUST explicitly confirm the override before the Inquiry Record is updated.

## Required Inputs

You MUST read and use all of these:

- `docs/plans/YYYY-MM-DD-<topic>-inquiry-record.md` — authoritative inquiry record
- `docs/plans/YYYY-MM-DD-<topic>-design-synthesis.md` — approved design synthesis
- `docs/plans/YYYY-MM-DD-<topic>-execution-plan.md` — approved execution plan
- `.maieutics/lenses.json` if present, otherwise `../inquiry/references/lenses.default.json`
- relevant repo/project context

## Output Files

**Feedback record:**
- Save to `docs/plans/YYYY-MM-DD-<topic>-feedback-record.md`
- Start from [references/feedback-record-template.md](references/feedback-record-template.md)

**Reconciliation record:**
- Save to `docs/plans/YYYY-MM-DD-<topic>-reconciliation-record.md`
- Start from [references/reconciliation-record-template.md](references/reconciliation-record-template.md)

## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Announce skill usage** — "I'm using the feedback-reconciliation skill to triage and route external feedback."
2. **Load authoritative inputs** — inquiry record, design synthesis, execution plan, lenses config, repo context
3. **Collect raw feedback** — capture all feedback verbatim into the feedback record using the template; assign each item a unique ID (F1, F2, ...)
4. **Decompose into atomic items** — break compound feedback into individual items; each gets metadata: source type (staging-test / uat / code-review / bug-report / other), environment (staging / production / local / n/a), evidence, reproducibility (always / intermittent / unknown / not-applicable)
5. **Run feedback reconciliation via codex exec** — use [feedback-reconciler-prompt.md](feedback-reconciler-prompt.md), passing file paths to the inquiry record, design synthesis, execution plan, feedback record, and lenses config
6. **Process classification results** — for each item, note the disposition (accept / reject / defer / needs-user-input) and layer (implementation / plan / design / inquiry)
7. **Present items needing user input** — group by layer, present all `needs-user-input` items in one message, record user answers verbatim in the reconciliation record
8. **Re-run reconciliation if needed** — if user answers change the classification of other items, re-run (max 3 rounds total)
9. **Promote accepted items to Inquiry Record** — append to the Settled Decisions section with a reference to the feedback item ID; mark superseded any decisions being overridden
10. **Update design synthesis if needed** — if any accepted item has layer=design or layer=inquiry, update the relevant sections of the design synthesis
11. **Route to appropriate stage** — based on the highest-affected layer among accepted items (see Routing Logic below)
12. **Commit feedback artifacts** — commit the feedback record and reconciliation record to the feature branch

## Feedback Reconciliation via Codex Exec

Use `codex exec` so the main session stays focused. Codex reads files directly — pass file paths, not file contents.

### How to Run

1. Ensure the feedback record is saved to disk with all decomposed items
2. Run `codex exec` with the prompt from [feedback-reconciler-prompt.md](feedback-reconciler-prompt.md), substituting:
   - `[INQUIRY_RECORD_PATH]` → actual path (e.g. `docs/plans/2026-03-10-slack-bot-inquiry-record.md`)
   - `[DESIGN_SYNTHESIS_PATH]` → actual path
   - `[EXECUTION_PLAN_PATH]` → actual path
   - `[FEEDBACK_RECORD_PATH]` → actual path (e.g. `docs/plans/2026-03-10-slack-bot-feedback-record.md`)
   - `[LENSES_CONFIG_PATH]` → `.maieutics/lenses.json` or the bundled default
   - `Project Context Summary` → a short summary of the repo
3. Parse the returned JSON and act on the result

### Exact Command

**IMPORTANT: Run this command in the foreground (do NOT use `run_in_background`).** Foreground execution keeps `RUN_ID` in scope so you can read the correct output file immediately after completion.

```bash
RUN_ID=$(uuidgen) && codex exec --full-auto -s read-only -o /tmp/maieutics-feedback-reconciliation-${RUN_ID}.json - <<'PROMPT'
<substituted prompt content from feedback-reconciler-prompt.md>
PROMPT
echo "OUTPUT_FILE=/tmp/maieutics-feedback-reconciliation-${RUN_ID}.json"
```

- `RUN_ID=$(uuidgen)` — generates a unique ID per invocation to avoid stale file collisions across sessions
- `--full-auto` — non-interactive execution with sandboxed auto-approval
- `-s read-only` — reconciler only reads files, never writes
- `-o /tmp/maieutics-feedback-reconciliation-${RUN_ID}.json` — saves the last message to a uniquely-named file for reliable parsing
- `-` — read prompt from stdin (use heredoc)

Parse the output from the `-o` file, not from stdout (stdout contains progress logs). Generate a new `RUN_ID` for each invocation (including retries and subsequent rounds).

**NEVER use glob patterns (e.g. `ls /tmp/maieutics-feedback-reconciliation-*.json`) to locate the output file.** Always use the exact `${RUN_ID}` path printed at the end of the command.

### Reconciliation Loop Limit

The reconciliation loop runs **at most 3 rounds**. If `needs-user-input` items remain after 3 rounds, stop and present unresolved items directly to the user for a final decision.

### Reconciliation Outcomes

**If status is `all-classified`:**
- All items have a definitive disposition
- Proceed to promotion and routing

**If status is `needs-user-input`:**
- Ask the user grouped questions (one message per layer group)
- Append answers to the reconciliation record
- Re-run reconciliation with updated context

## Routing Logic

After promotion, route based on the **highest-affected layer** among all accepted items.

Layer priority (highest to lowest):

1. **inquiry** — value, scope, or success criteria changed
2. **design** — architecture, approach, or component boundary changed
3. **plan** — task structure, sequencing, or file-level change
4. **implementation** — code-level fix only

| Highest Layer | Route To | What Happens |
|---|---|---|
| inquiry | `inquiry` | Change inquiry record status back to `questioning`, append accepted items to Open Questions, invoke inquiry to reopen inquiry |
| design | update design synthesis → `execution-planning` | Update affected design synthesis sections, then invoke execution-planning to create a delta plan |
| plan | `execution-planning` | Invoke execution-planning to create a delta plan covering only the changed portions |
| implementation | `execution-planning` | Even pure implementation fixes go through execution-planning for a delta plan (all implementation work has a plan) |

### Delta Plan Semantics

When routing to `execution-planning`, pass a clear instruction: "Create a delta plan for accepted feedback items F1, F3, F5 (reference: reconciliation record at `docs/plans/...`). Do not re-plan existing completed work."

The delta plan:
- References the original plan but does not duplicate it
- Contains only the new/changed tasks
- Gets its own review loop via the normal execution-planning flow
- After review, hands off to `delegated-execution` or `guided-execution` as usual

### Reopening Inquiry

When routing to `inquiry`:
- Change the inquiry record status to `questioning`
- Append accepted inquiry-level items to the Open Questions section
- Inquiry will see the existing questions/answers and generate focused inquiry rounds on the changed scope

### Rejected and Deferred Items

- **Rejected** items are recorded in the reconciliation record with reason. Not promoted, do not affect routing.
- **Deferred** items are recorded with a note. Optionally appended to the inquiry record's Open Questions section as "deferred from reconciliation round [date]" so they are not lost.

## Red Flags

**Never:**
- Treat raw feedback as authoritative without user confirmation
- Skip the codex classification step
- Route to inquiry for implementation-only changes
- Promote rejected or deferred items to the Inquiry Record
- Create a new worktree — work in the existing one
- Override an existing authoritative decision without explicit user approval

**Always:**
- Record raw feedback verbatim before decomposing
- Use codex exec for classification (not self-classification)
- Ask the user about ambiguous or conflicting items
- Commit feedback artifacts before routing to the next stage

## Integration

**Called after:**
- `closing-the-branch` — when feedback arrives post-completion
- Directly by the user — when external feedback needs triage at any point

**Routes to:**
- `inquiry` — inquiry-level changes
- `execution-planning` — design/plan/implementation-level changes

**Pairs with:**
- `using-git-worktrees` — expects an existing worktree; may need to re-enter one

## Key Principles

- **Classification happens via codex exec** — main session stays clean
- **Codex only classifies** — no design, no planning, no implementation
- **Feedback is not authority** — only promoted items become authoritative
- **User decides** — ambiguous items always go to the user
- **Smallest possible loop** — route to the lowest stage that can address the feedback
- **Delta, not redo** — create delta plans, not full re-plans
