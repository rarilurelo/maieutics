---
name: delegated-execution
description: "Use when executing implementation plans with independent tasks via `codex exec` sessions. This version adds authoritative inquiry context and a final lenses implementation review loop."
---

# Delegated Execution

Execute the plan by dispatching a **fresh `codex exec` session per task**, with standard per-task review after each task:

1. spec compliance review
2. code quality review

This is a one-shot execution model — Codex workers run non-interactively and return structured JSON. If a worker encounters a blocker, it reports the reason via JSON status rather than asking interactive questions; the controller then decides whether to retry, modify the prompt, or escalate to the user.

After all planned tasks are complete, run a **final lenses implementation review** via `codex exec`. If Codex finds Critical or Important issues, implementation continues until those issues are fixed. If Codex says the remaining blockers require the human, ask grouped user questions, record the outcome, confirm any inquiry record changes with the user, and continue.

## Core Principle

Fresh `codex exec` session per task + standard per-task reviews + final lenses implementation review = higher quality, less context pollution, better alignment with what the human actually asked for.

## Authoritative Inputs

Before executing any task, identify and keep track of:

- `docs/plans/YYYY-MM-DD-<topic>-inquiry-record.md`
- `docs/plans/YYYY-MM-DD-<topic>-design-synthesis.md`
- `docs/plans/YYYY-MM-DD-<topic>-execution-plan.md`
- Lenses config: `.maieutics/lenses.json` if present, otherwise `../inquiry/references/lenses.default.json`

The inquiry record, user answers, and confirmed assumptions are authoritative. Working assumptions are unresolved until explicitly handled or confirmed. Invalidated assumptions must not be used. If the plan later proves inconsistent with them, update the plan and implement against the corrected version.

Review findings are **not** authoritative on their own. During review loops, treat "user answer captured" and "inquiry record updated" as separate events. You may record review findings, user answers, and proposed inquiry changes in the implementation review record immediately, but you MUST NOT update the inquiry record until the user explicitly confirms the exact proposed changes.

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
5. Start a `Carried Findings` block for facts learned during implementation that later tasks should inherit

### Per Task Loop

For each task:

1. **Record pre-task HEAD:** run `git rev-parse HEAD` and store the hash for rollback
2. **Run the implementer** via `codex exec` with the prompt from [implementer-prompt.md](implementer-prompt.md), substituting all placeholders. Give the worker:
   - the full task text
   - relevant inquiry record excerpts
   - relevant design context
   - carried findings from earlier tasks
   - repo context needed for the task
3. **Parse JSON output** from the `-o` file
4. **If `status: "blocked"`:** reset to pre-task HEAD (`git reset --hard <hash> && git clean -fd`), analyze `blocker_reason`, optionally ask the user for clarification, then relaunch the implementer with a modified prompt that addresses the blocker
5. **If `status: "completed"`:** proceed to spec review
6. **Run spec compliance review** via `codex exec` with the prompt from [spec-reviewer-prompt.md](spec-reviewer-prompt.md) (read-only)
7. **If spec review `non-compliant`:** reset to pre-task HEAD (`git reset --hard <hash> && git clean -fd`), relaunch the implementer with the spec review issues included as fix instructions in the prompt
8. **Run code quality review** via `codex exec` with the prompt from [code-quality-reviewer-prompt.md](code-quality-reviewer-prompt.md) (read-only)
9. **If code quality review `issues-found` with Critical or Important severity:** reset to pre-task HEAD (`git reset --hard <hash> && git clean -fd`), relaunch the implementer with the code quality issues included as fix instructions in the prompt
10. **Record carried findings** from the implementer's `project_discoveries` and carry them into later tasks
11. **Mark the task complete** in TodoWrite

#### Implementer — Exact Command

**IMPORTANT: Run this command in the foreground (do NOT use `run_in_background`).** Foreground execution keeps `RUN_ID` in scope so you can read the correct output file immediately after completion.

```bash
RUN_ID=$(uuidgen) && codex exec --full-auto -o /tmp/maieutics-impl-${RUN_ID}.json - <<'PROMPT'
<substituted prompt content from implementer-prompt.md>
PROMPT
echo "OUTPUT_FILE=/tmp/maieutics-impl-${RUN_ID}.json"
```

- `RUN_ID=$(uuidgen)` — generates a unique ID per invocation to avoid stale file collisions across sessions
- `--full-auto` — non-interactive execution with sandboxed auto-approval
- `-o /tmp/maieutics-impl-${RUN_ID}.json` — saves the last message to a uniquely-named file for reliable parsing
- `-` — read prompt from stdin (use heredoc)

Parse the output from the `-o` file, not from stdout (stdout contains progress logs). Generate a new `RUN_ID` for each invocation (including retries and fix-loop relaunches).

**NEVER use glob patterns (e.g. `ls /tmp/maieutics-impl-*.json`) to locate the output file.** Always use the exact `${RUN_ID}` path printed at the end of the command.

#### Spec Reviewer — Exact Command

**IMPORTANT: Run this command in the foreground (do NOT use `run_in_background`).** Foreground execution keeps `RUN_ID` in scope so you can read the correct output file immediately after completion.

```bash
RUN_ID=$(uuidgen) && codex exec --full-auto -s read-only -o /tmp/maieutics-spec-review-${RUN_ID}.json - <<'PROMPT'
<substituted prompt content from spec-reviewer-prompt.md>
PROMPT
echo "OUTPUT_FILE=/tmp/maieutics-spec-review-${RUN_ID}.json"
```

- `RUN_ID=$(uuidgen)` — generates a unique ID per invocation to avoid stale file collisions across sessions
- `--full-auto` — non-interactive execution with sandboxed auto-approval
- `-s read-only` — reviewer only reads files, never writes
- `-o /tmp/maieutics-spec-review-${RUN_ID}.json` — saves the last message to a uniquely-named file for reliable parsing
- `-` — read prompt from stdin (use heredoc)

Parse the output from the `-o` file, not from stdout (stdout contains progress logs). Generate a new `RUN_ID` for each invocation (including retries and subsequent review rounds).

**NEVER use glob patterns (e.g. `ls /tmp/maieutics-spec-review-*.json`) to locate the output file.** Always use the exact `${RUN_ID}` path printed at the end of the command.

#### Code Quality Reviewer — Exact Command

**IMPORTANT: Run this command in the foreground (do NOT use `run_in_background`).** Foreground execution keeps `RUN_ID` in scope so you can read the correct output file immediately after completion.

```bash
RUN_ID=$(uuidgen) && codex exec --full-auto -s read-only -o /tmp/maieutics-quality-review-${RUN_ID}.json - <<'PROMPT'
<substituted prompt content from code-quality-reviewer-prompt.md>
PROMPT
echo "OUTPUT_FILE=/tmp/maieutics-quality-review-${RUN_ID}.json"
```

- `RUN_ID=$(uuidgen)` — generates a unique ID per invocation to avoid stale file collisions across sessions
- `--full-auto` — non-interactive execution with sandboxed auto-approval
- `-s read-only` — reviewer only reads files, never writes
- `-o /tmp/maieutics-quality-review-${RUN_ID}.json` — saves the last message to a uniquely-named file for reliable parsing
- `-` — read prompt from stdin (use heredoc)

Parse the output from the `-o` file, not from stdout (stdout contains progress logs). Generate a new `RUN_ID` for each invocation (including retries and subsequent review rounds).

**NEVER use glob patterns (e.g. `ls /tmp/maieutics-quality-review-*.json`) to locate the output file.** Always use the exact `${RUN_ID}` path printed at the end of the command.

### Per-Task Review Loop Limit

Per-task implementer/spec/code-quality fix loops are capped at **3 rounds per task**. A "round" is one implementer launch followed by its spec review and code quality review.

After 3 rounds with unresolved Critical or Important issues: stop and escalate to the user.

- Present the unresolved issues clearly
- Ask the user for direction on each remaining issue
- Record the answers and continue with a final implementer launch incorporating the user's decisions

This mirrors the existing final lenses review loop limit and prevents infinite fix loops where each fix introduces new issues.

### Final Lenses Implementation Review

After all plan tasks are complete:

1. Save or update `docs/plans/YYYY-MM-DD-<topic>-implementation-review-record.md` using [references/implementation-review-record-template.md](references/implementation-review-record-template.md)
2. Run `codex exec` with the prompt from [implementation-reviewer-prompt.md](implementation-reviewer-prompt.md), substituting:
   - `[INQUIRY_RECORD_PATH]` -> actual path to the inquiry record
   - `[DESIGN_SYNTHESIS_PATH]` -> actual path to the design synthesis
   - `[EXECUTION_PLAN_PATH]` -> actual path to the execution plan
   - `[LENSES_CONFIG_PATH]` -> `.maieutics/lenses.json` or the bundled default
   - `Carried Findings` -> bullet list of durable discoveries from task workers
   - `Current Implementation State` -> branch context, diff summary, test results, used only as an entry point for direct repo inspection
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

### Inquiry Record Confirmation Gate

Whenever a review round suggests changing the inquiry record:

1. Append the review findings, blocker questions, and any user answers to the implementation review record first
2. Draft a batch of `inquiry_record_update_candidates` containing:
   - target section
   - change type
   - exact text to add or replace
   - related review issue IDs
   - rationale
3. Present the batch to the user with this exact confirmation lead-in:
   - `以下を inquiry record に反映してよいですか`
4. Apply the inquiry record update only if the user explicitly approves the batch
5. If the user supplies revised wording, replace the draft candidates with the user's revised wording and re-present the full batch for confirmation
6. If the user declines or leaves the confirmation unresolved, leave the inquiry record unchanged and stop the review loop in a pending-confirmation state

Do not treat answering a blocker question as implicit permission to update the inquiry record.

### Review Loop Limit

The final review loop runs **at most 3 rounds**. If Critical or Important issues remain after 3 rounds, stop self-fixing and escalate to the user:

- Present the unresolved issues clearly
- Ask the user for direction on each remaining issue
- Record the answers in the implementation review record
- Draft inquiry record update candidates and ask for explicit confirmation before any inquiry record edit
- Only after the inquiry record update batch is explicitly confirmed, apply the user's decisions and run one final review
- If confirmation remains unresolved or declined, stop in a pending-confirmation state without re-running review

This prevents infinite fix loops where each fix introduces new issues.

### Final Review Outcomes

**If status is `approved`:**
- Continue to `closing-the-branch`

**If status is `needs-fix`:**
- Convert the issues into follow-up implementation work
- Record any pending inquiry record changes in the implementation review record only
- If the findings reveal a design/plan mismatch, update the design and/or plan first
- Dispatch the implementation worker to fix the issues
- Re-run the final lenses implementation review (respecting the 3-round limit)

**If status is `needs-user-input`:**
- Ask the user all blocker questions in one grouped message
- Append the answers verbatim to the implementation review record
- Draft inquiry record update candidates
- Ask `以下を inquiry record に反映してよいですか`
- Only if the user explicitly confirms, update the inquiry record and its assumption state
- If the user declines or revises the text, keep the inquiry record unchanged until the batch is explicitly confirmed
- Only after confirmation, update design and/or plan if needed
- Only after confirmation, implement the resulting changes
- Only after confirmation, re-run the final lenses implementation review

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

- Fresh context per task — each `codex exec` session starts clean, avoiding context pollution
- Independent review — spec and code quality reviewers run in isolated read-only sessions
- Durable carried findings passed forward across tasks
- Blockers reported via JSON status, not mid-stream questions
- Final implementation gets checked against inquiry, design, and plan — not just task text

### vs. Guided Execution

- Same session, faster iteration — controller orchestrates `codex exec` workers without leaving the session
- Automatic review checkpoints after every task
- Pre-task HEAD recording enables clean rollback and retry on any failure
- Final whole-feature review before branch completion

## Red Flags

Never do these:

- Start implementation without a plan
- Ignore the inquiry record or treat it as optional
- Skip spec review or code quality review for a task
- Skip the final lenses implementation review
- Let Codex run outside the feature worktree
- Let Codex install dependencies or push to remote
- Proceed with unresolved Critical or Important review issues
- Make workers read the plan file themselves when you can provide the exact task text
- Ignore worker blocker status or reviewer escalation questions
- Finish the branch before the final review loop is clean
- Accept review findings that are not backed by inspected repo evidence
- Update the inquiry record during review without explicit user confirmation
