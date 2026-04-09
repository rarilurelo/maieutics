# Implementation Worker Prompt Template

Run this prompt via `codex exec`. Codex reads the referenced files directly from the working directory.

**Run in the foreground (do NOT use `run_in_background`):**

```bash
RUN_ID=$(uuidgen) && codex exec --full-auto --reasoning-effort high -o /tmp/maieutics-impl-${RUN_ID}.json - <<'PROMPT'
<paste the prompt below with [PLACEHOLDERS] substituted>
PROMPT
echo "OUTPUT_FILE=/tmp/maieutics-impl-${RUN_ID}.json"
```

Parse the JSON from the `-o` output file (`/tmp/maieutics-impl-${RUN_ID}.json`), not from stdout. Generate a new `RUN_ID` for each invocation. **Never use glob patterns to find the output file.**

````text
You are implementing Task [N]: [task name].

## Task Description
[FULL TEXT of task from the plan - paste it here; do not make the worker read the file]

## Authoritative Context
- Raw user input / inquiry record: [PASTE relevant excerpt and/or path]
- Approved design synthesis: [PASTE relevant excerpt and/or path]
- Approved execution plan: [PASTE relevant excerpt and/or path]
- Carried findings from earlier tasks: [PASTE bullet list]

## Context
[Scene-setting: where this fits, dependencies, architectural context]

## Source-of-Truth Rules
- The inquiry record and raw user answers are authoritative.
- If the task text appears to conflict with the inquiry record or approved design synthesis, output a "blocked" status with the conflict details instead of guessing.
- Do not widen scope.
- Do not invent extra features.

## Side-Effect Boundary Rules
You may edit files, run local tests, and create local git commits. You MUST NOT install dependencies, run remote git operations (push, fetch, pull), or work outside the feature worktree.

## Your Job
1. Implement exactly what the task specifies
2. Write tests (following TDD if the task says to)
3. Verify the implementation works
4. Commit your work
5. Self-review
6. Output the result

If the task cannot proceed (missing dependency output, conflicting requirements, ambiguous spec that cannot be resolved from authoritative context), do not guess. Output `status: "blocked"` with the reason. You need not clean up — the controller handles full rollback.

Work from: [directory]

## Before Reporting Back: Self-Review
Review your work with fresh eyes.

Ask yourself:

**Completeness**
- Did I fully implement everything in the task?
- Did I miss any requirements?
- Did I ignore anything from the inquiry record or design context?

**Quality**
- Is this my best work?
- Are names clear and accurate?
- Is the code clean and maintainable?

**Discipline**
- Did I avoid overbuilding (YAGNI)?
- Did I only build what was requested?
- Did I follow existing patterns in the codebase?

**Testing**
- Do tests verify behavior?
- Did I follow TDD if required?
- Are tests comprehensive enough for this task?

If you find issues during self-review, fix them before reporting.

## Output
Return JSON inside one fenced code block and nothing else.

```json
{
  "status": "completed | blocked",
  "blocker_reason": "null or string",
  "summary": "What was implemented",
  "files_changed": ["path/to/file"],
  "tests_run": "test command and results summary",
  "commit_hash": "abc1234 or null if blocked",
  "self_review_findings": ["finding1"],
  "project_discoveries": ["durable facts for later tasks"]
}
```
````
