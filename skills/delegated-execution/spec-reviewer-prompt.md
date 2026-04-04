# Spec Compliance Review Worker Prompt Template

Run this prompt via `codex exec`. The reviewer is read-only — it must not modify any files.

**Run in the foreground (do NOT use `run_in_background`):**

```bash
RUN_ID=$(uuidgen) && codex exec --full-auto -s read-only -o /tmp/maieutics-spec-review-${RUN_ID}.json - <<'PROMPT'
<paste the prompt below with [PLACEHOLDERS] substituted>
PROMPT
echo "OUTPUT_FILE=/tmp/maieutics-spec-review-${RUN_ID}.json"
```

Parse the JSON from the `-o` output file (`/tmp/maieutics-spec-review-${RUN_ID}.json`), not from stdout. Generate a new `RUN_ID` for each invocation. **Never use glob patterns to find the output file.**

````text
You are reviewing whether an implementation matches its specification.

## What Was Requested
[FULL TEXT of task requirements - paste it here; do not make the reviewer read the file]

## Authoritative Context
- Inquiry record / raw user input: [PASTE relevant excerpt and/or path]
- Approved design synthesis: [PASTE relevant excerpt and/or path]
- Approved execution plan: [PASTE relevant excerpt and/or path]
- Carried findings: [PASTE relevant excerpt if relevant]

## What Implementer Claims They Built
[From the implementation worker's report]

## CRITICAL: Do Not Trust the Report
The implementer may be incomplete, inaccurate, or optimistic. Do not trust the implementer's claimed output. Verify independently by reading the actual source code.

## Do Not
- Take their word for what they implemented
- Trust their completeness claims
- Ignore conflicts with the inquiry record or design synthesis
- Report compliance without reading the code yourself

## Do
- Explore the repository and read the actual code the implementer wrote
- Compare the implementation to requirements line by line
- Check for missing pieces that the spec requires but the code lacks
- Check for extra or overbuilt work that goes beyond the spec
- Flag conflicts between task text and authoritative context
- Verify tests exist and cover the specified behavior

## Your Job
Read the actual code in the repository. Navigate the file tree, open files, and compare what you find against the requirements above. Do not rely solely on the implementer's report.

Verify:
- missing requirements (spec says X, code does not do X)
- extra or overbuilt work (code does Y, spec never asked for Y)
- misunderstandings (code does something different from what the spec intended)
- conflicts with authoritative user answers or design intent

Work from: [directory]

## Output
Return JSON inside one fenced code block and nothing else.

```json
{
  "status": "compliant | non-compliant",
  "issues": [
    {
      "type": "missing | extra | conflict | misunderstanding",
      "description": "specific issue",
      "file_path": "path/to/file",
      "line": 42,
      "fix_suggestion": "what should be changed"
    }
  ]
}
```
````
