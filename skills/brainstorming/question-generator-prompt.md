# Multi-Perspective Question Generator Prompt Template

Run this prompt via `codex exec`. Codex reads the referenced files directly from the working directory.

**Run in the foreground (do NOT use `run_in_background`):**

```bash
RUN_ID=$(uuidgen) && codex exec --full-auto -s read-only -o /tmp/maieutics-questions-${RUN_ID}.json - <<'PROMPT'
<paste the prompt below with [PLACEHOLDERS] substituted>
PROMPT
echo "OUTPUT_FILE=/tmp/maieutics-questions-${RUN_ID}.json"
```

Parse the JSON from the `-o` output file (`/tmp/maieutics-questions-${RUN_ID}.json`), not from stdout. Generate a new `RUN_ID` for each invocation. **Never use glob patterns to find the output file.**

Your job is **question generation only**. You do not design the solution. You do not write a plan. You do not implement anything.

````text
You are a multi-perspective question generator.

## Stage
brainstorming

## Authoritative Input Files
Read these files before proceeding:
- Discovery Log (includes raw user input): `[DISCOVERY_LOG_PATH]`
- Perspective Config: `[PERSPECTIVE_CONFIG_PATH]`

## Project Context Summary
[Short summary of repo structure, tech stack, and relevant constraints — provided by the controller]

## Rules
- The discovery log and raw user answers are authoritative.
- Ask 3-5 questions total. Default to 4.
- Prefer distributing questions across distinct perspectives.
- Only ask questions whose answers would materially change the design or implementation plan.
- Never ask for information that is already explicitly answered.
- Prefer multiple-choice or constrained questions when helpful.
- Do not propose code, implementation steps, or architecture choices.
- If there is already enough information to produce a design, return no questions and set status to "enough-context".
- If human tradeoff choices remain, surface the highest-leverage ones first.

## Output
Return JSON inside one fenced code block and nothing else.

```json
{
  "status": "needs-user-input | enough-context",
  "summary": "1-3 sentence rationale",
  "questions": [
    {
      "id": "Q1",
      "perspective_id": "product",
      "question": "Question text",
      "why_it_matters": "Why this answer changes scope, design, or planning",
      "answer_format": "multiple-choice | short-text | list",
      "suggested_choices": [
        "Optional suggested choice A",
        "Optional suggested choice B"
      ]
    }
  ],
  "authoritative_assumptions": [
    "Only include assumptions that are already explicitly supported by the discovery log"
  ]
}
```
````
