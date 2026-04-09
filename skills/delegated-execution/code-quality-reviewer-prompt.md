# Code Quality Reviewer Prompt Template

Run this prompt via `codex exec`. Codex reads the referenced files directly from the working directory.

**Purpose:** Verify implementation is well-built (clean, tested, maintainable).

**Only dispatch after spec compliance review passes.**

**Run in the foreground (do NOT use `run_in_background`):**

```bash
RUN_ID=$(uuidgen) && codex exec --full-auto -s read-only -o /tmp/maieutics-quality-review-${RUN_ID}.json - <<'PROMPT'
<paste the prompt below with [PLACEHOLDERS] substituted>
PROMPT
echo "OUTPUT_FILE=/tmp/maieutics-quality-review-${RUN_ID}.json"
```

Parse the JSON from the `-o` output file (`/tmp/maieutics-quality-review-${RUN_ID}.json`), not from stdout. Generate a new `RUN_ID` for each invocation. **Never use glob patterns to find the output file.**

````text
You are reviewing whether an implementation is well-built.

## What Was Implemented
[From the implementer's report — what they claim they built]

## Task Requirements
[FULL TEXT of task requirements from the plan]

## CRITICAL: Do Not Trust the Report
Do not trust the implementer's report — read the actual code. The worker may be incomplete, inaccurate, or optimistic.

## What to Check

**Structure**
- Does each file have one clear responsibility with a well-defined interface?
- Are units decomposed so they can be understood and tested independently?
- Is the implementation following the file structure from the plan?
- Did this change create new files that are already large, or significantly grow existing files?
  (Don't flag pre-existing file sizes — focus on what this change contributed.)

**Quality**
- Are names clear and accurate?
- Is the code clean and maintainable?
- Are there unnecessary abstractions or overengineering?
- Does the code follow existing patterns in the codebase?

**Error Handling**
- Are error cases handled appropriately?
- Are there silent failures or swallowed exceptions?

**Testing**
- Do tests verify behavior, not implementation details?
- Are edge cases covered?
- Are tests readable and maintainable?

## Severity Definitions
- Critical: The code has a serious defect, security issue, or architectural problem
- Important: A meaningful quality issue that should be fixed before moving on
- Minor: Good to improve, but does not block progress

Work from: [directory]

## Output
Return JSON inside one fenced code block and nothing else.

```json
{
  "status": "acceptable | issues-found",
  "strengths": ["what was done well"],
  "issues": [
    {
      "severity": "Critical | Important | Minor",
      "description": "specific issue",
      "file_path": "path/to/file",
      "line": 42,
      "fix_suggestion": "what should be changed"
    }
  ]
}
```
````
