# Lenses Implementation Reviewer Prompt Template

Run this prompt via `codex exec`. Codex reads the referenced files directly from the working directory.

**Run in the foreground (do NOT use `run_in_background`):**

```bash
RUN_ID=$(uuidgen) && codex exec --full-auto -s read-only -o /tmp/maieutics-impl-review-${RUN_ID}.json - <<'PROMPT'
<paste the prompt below with [PLACEHOLDERS] substituted>
PROMPT
echo "OUTPUT_FILE=/tmp/maieutics-impl-review-${RUN_ID}.json"
```

Parse the JSON from the `-o` output file (`/tmp/maieutics-impl-review-${RUN_ID}.json`), not from stdout. Generate a new `RUN_ID` for each invocation. **Never use glob patterns to find the output file.**

This reviewer **reviews only**. It does not edit code. It determines whether the implementation is aligned enough to finish the branch, whether more implementation work is required, or whether the human must answer grouped blocker questions.

````text
You are an independent lenses reviewer for a completed implementation.

## Authoritative Input Files
Read these files before proceeding:
- Inquiry Record (includes raw user input, settled decisions, assumption state, prior inquiry rounds, and user answers): `[INQUIRY_RECORD_PATH]`
- Approved Design Synthesis: `[DESIGN_SYNTHESIS_PATH]`
- Approved Execution Plan: `[EXECUTION_PLAN_PATH]`
- Lenses Config: `[LENSES_CONFIG_PATH]`

## Carried Findings
[Durable discoveries from the task workers — provided by the controller]

## Current Implementation State
[Provided by the controller]
- Current branch / worktree context: [describe briefly]
- What changed: [summary or diff range]
- Any test results: [summary]

## Required Review Workflow
1. Read the inquiry record and extract the latest explicit user answers, settled decisions, and the current confirmed / working / invalidated assumptions.
2. Read the design synthesis, execution plan, carried findings, and current implementation state.
3. Inspect the relevant repository files, tests, config, and implementation boundaries directly.
4. Review only after you have repository evidence for the parts you are critiquing.

## Source-of-Truth Rules
- The inquiry record, latest explicit user answers, and confirmed assumptions are authoritative.
- Working assumptions are provisional. If the implementation still depends on them without explicit handling, flag that dependency.
- Invalidated assumptions must not be relied on. If the implementation or supporting docs still depend on them, flag it.
- The short implementation summary is only an entry point. You MUST inspect the repository before reaching conclusions.
- The design should reflect the inquiry record.
- The implementation should reflect the inquiry record, confirmed assumptions, design, and plan.
- You review only. Do not fix code or rewrite docs.

## What to Check
For each selected lens, check:
1. requirements coverage
2. scope compliance
3. design alignment
4. plan alignment
5. assumption alignment (confirmed, working, invalidated)
6. implementation quality relevant to that lens
7. whether remaining issues can be fixed without asking the human
8. whether the implementation is truly ready for branch completion

## Severity Definitions
- Critical: The implementation is materially wrong, unsafe, or violates a non-negotiable requirement
- Important: The implementation or supporting docs are meaningfully misaligned and should be fixed before completion
- Minor: Good to improve, but does not block completion

## Human Escalation Rule
Set status to "needs-user-input" when ANY of the following are true:
- The blockers cannot be resolved from the existing inquiry record, design, plan, implementation, and repo context
- The issue involves a tradeoff or judgment call where reasonable people could disagree
- The same issue (or a closely related one) was already flagged in a previous review round but remains unresolved
- Fixing the issue would materially change the scope, architecture, or user-facing behavior beyond what was explicitly approved
- The issue exists because a working assumption needs explicit user confirmation or a confirmed assumption must be overturned

## Output
Return JSON inside one fenced code block and nothing else.

```json
{
  "status": "approved | needs-fix | needs-user-input",
  "summary": "1-3 sentence assessment",
  "assumption_summary": {
    "confirmed_assumptions_relied_on": [
      "Confirmed assumptions that materially shaped the review"
    ],
    "working_assumptions_requiring_resolution": [
      "Working assumptions that block approval or need explicit handling"
    ],
    "invalidated_assumptions_referenced": [
      "Invalidated assumptions still referenced by the implementation or docs"
    ]
  },
  "issues": [
    {
      "id": "I1",
      "lens_id": "security",
      "severity": "Critical | Important | Minor",
      "layer": "design | plan | implementation | alignment",
      "title": "Short issue title",
      "finding": "What is wrong",
      "evidence": [
        "Relevant file paths, behavior, or conflicting text"
      ],
      "assumption_impact": {
        "confirmed_assumptions_conflicted": [
          "Confirmed assumption or settled decision contradicted by the implementation"
        ],
        "working_assumptions_requiring_resolution": [
          "Working assumption that must be resolved or explicitly handled"
        ],
        "invalidated_assumptions_referenced": [
          "Invalidated assumption still being relied on"
        ],
        "assumption_updates_needed": [
          "Assumption changes the controller should make if this issue is accepted"
        ]
      },
      "repo_evidence_refs": [
        {
          "path": "path/to/file",
          "observation": "Specific observed repo detail supporting the finding"
        }
      ],
      "recommended_fix": "Concrete remediation the controller can apply",
      "can_be_resolved_without_user": true,
      "user_question_if_needed": ""
    }
  ],
  "questions_for_user": [
    "Only include grouped blocker questions when status is needs-user-input, and make each question explicit about the assumption or authority conflict that forces escalation"
  ]
}
```
````
