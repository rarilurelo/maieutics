---
name: inquiry
description: "You MUST use this before any creative work - creating features, building components, adding functionality, or modifying behavior. Explores user intent, requirements, and design before implementation, using per-lens question generation and inquiry recording."
---

# Inquiry Into Design

## Overview

Help turn ideas into fully formed designs and specs through natural collaborative dialogue.

This version extends the normal inquiry flow with **per-lens question generation**. Questions are generated via `codex exec` so the main Claude Code session stays clean. Each `codex exec` run is responsible for exactly one lens, explores the repository for evidence, and returns structured output for that lens only. Codex's job is **question generation only**. It does not design, plan, or implement.

The user's raw input, every generated question, every supplement shown to the user, every answer, and the evolving assumption state are saved and become **authoritative inputs** for the rest of the workflow.

Do NOT invoke any implementation skill, write any code, scaffold any project, or take any implementation action until you have presented a design and the user has approved it. This applies to EVERY project regardless of perceived simplicity.

## Anti-Pattern: "This Is Too Simple To Need A Design"

Every project goes through this process. A todo list, a single-function utility, a config change — all of them.

"Simple" projects are where unexamined assumptions cause the most wasted work. The design can be short for a truly simple project, but you MUST still present it and get approval.

## Authoritative Inputs and Artifacts

**Project-local lenses config:**
- Prefer `.maieutics/lenses.json` if it exists
- Otherwise use [references/lenses.default.json](references/lenses.default.json)

**Inquiry record:**
- Save to `docs/plans/YYYY-MM-DD-<topic>-inquiry-record.md`
- Start from [references/inquiry-record-template.md](references/inquiry-record-template.md)

**Design synthesis:**
- Save to `docs/plans/YYYY-MM-DD-<topic>-design-synthesis.md`
- Start from [references/design-synthesis-template.md](references/design-synthesis-template.md)

**Authority order:**
1. Latest explicit user answers in the inquiry record
2. Confirmed assumptions in the inquiry record
3. Raw user input captured in the inquiry record
4. Approved design synthesis
5. Everything else

If design text later conflicts with the inquiry record, update the design. Do NOT silently override the user's answers.

## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Explore project context** — check files, docs, recent commits
2. **Load lenses config** — use project-local config if present, otherwise the bundled default
3. **Create isolated workspace** — invoke `using-git-worktrees` to create a feature branch. Use the user's initial topic/request to derive the branch name. All subsequent artifacts (inquiry record, design synthesis, etc.) are committed on this feature branch, never on main.
4. **Create or update inquiry record** — save the user's raw input verbatim before asking more questions
5. **Generate per-lens question sets via codex exec** — use [question-generator-prompt.md](question-generator-prompt.md), passing file paths to the inquiry record and lenses config, and run one independent `codex exec` per active lens
6. **Aggregate lens outputs** — collect every per-lens result, merge duplicated questions, preserve lens attribution, and update assumption state before asking the user anything
7. **Ask the user the raw questions with supplements** — present Codex's original question text unchanged, then add a structured supplement under it with:
   - **Context**: why this question surfaced now, based on the inquiry record and the current lens run
   - **Why this matters**: the exact design, planning, scope, or risk implication
   - **Observed evidence**: the evidence refs that triggered the question
   - **Choices**: optional suggested choices with brief pros/cons or concrete examples
8. **Record answers verbatim** — append each raw Codex question, the supplement shown to the user, the evidence refs, and the user's answer to the inquiry record
9. **Maintain assumptions explicitly** — promote, update, or invalidate assumptions in the inquiry record after every round
10. **Repeat until all active lenses are exhausted** — continue inquiry rounds until every active lens returns `meaningful-questions-exhausted`
11. **Propose 2-3 approaches** — with trade-offs and your recommendation
12. **Present design** — in sections scaled to complexity, get user approval after each section
13. **Write design synthesis** — save to `docs/plans/YYYY-MM-DD-<topic>-design-synthesis.md` and commit (now on the feature branch)
14. **Transition to implementation planning** — invoke `execution-planning`

## Lenses Question Generation

### How to Generate Questions

Use `codex exec` so the main Claude Code session does not get polluted with intermediate inquiry attempts. Codex can read files in the working directory directly — pass file paths, not file contents.

Preferred pattern:

1. Main session ensures the inquiry record is saved to disk with the latest user answers and assumption state
2. Main session enumerates the active lens IDs from the current stage in the lenses config
3. Main session runs one independent `codex exec` per active lens using [question-generator-prompt.md](question-generator-prompt.md), substituting:
   - `[INQUIRY_RECORD_PATH]` → actual path to the inquiry record (e.g. `docs/plans/2026-03-10-slack-bot-inquiry-record.md`)
   - `[LENSES_CONFIG_PATH]` → `.maieutics/lenses.json` or the bundled default
   - `[TARGET_LENS_ID]` → the active lens for that run
   - `Project Context Summary` → a short summary of the repo (tech stack, structure, relevant constraints)
4. Each Codex run reads the files, explores the repository for that lens, and returns **only structured JSON output for that lens**
5. Main session aggregates the per-lens outputs, deduplicates overlapping questions, preserves evidence and lens attribution, and updates the inquiry record
6. Main session presents the resulting questions to the user and records the answers in the inquiry record

#### Exact Command

**IMPORTANT: Run each command in the foreground (do NOT use `run_in_background`).** Foreground execution keeps `RUN_ID` in scope so you can read the correct output file immediately after completion.

```bash
RUN_ID=$(uuidgen) && codex exec --full-auto --reasoning-effort high -s read-only -o /tmp/maieutics-questions-${RUN_ID}.json - <<'PROMPT'
<substituted prompt content from question-generator-prompt.md>
PROMPT
echo "OUTPUT_FILE=/tmp/maieutics-questions-${RUN_ID}.json"
```

- `RUN_ID=$(uuidgen)` — generates a unique ID per invocation to avoid stale file collisions across sessions
- `--full-auto` — non-interactive execution with sandboxed auto-approval
- `-s read-only` — Codex only reads files and generates output, never writes
- `-o /tmp/maieutics-questions-${RUN_ID}.json` — saves the last message to a uniquely-named file for reliable parsing
- `-` — read prompt from stdin (use heredoc)

Parse the output from the `-o` file, not from stdout (stdout contains progress logs). Generate a new `RUN_ID` for each invocation (including retries and subsequent rounds).

**NEVER use glob patterns (e.g. `ls /tmp/maieutics-questions-*.json`) to locate the output file.** Always use the exact `${RUN_ID}` path printed at the end of the command.

### Aggregation Responsibilities

The main session is responsible for all cross-lens coordination:

- Determine which lenses are active for the inquiry stage
- Run the lens workers independently so they can be parallelized by the calling environment
- Merge duplicate questions that are substantively asking for the same missing fact or tradeoff
- Preserve the original raw question text for every remaining question
- Combine evidence refs when deduplicating
- Order the final user-facing list by blockingness, then priority, then lens
- Update the inquiry record's assumptions and open questions after each aggregation pass

### Rules for User-Facing Questions

- Present the raw Codex question text unchanged
- Add supplements instead of rewriting the question
- Do NOT remove jargon from the raw question if that changes the wording; explain it in the supplement instead
- Show the evidence that triggered the question
- Group displayed questions by lens using the configured display policy
- Default display policy is `show-all-by-lens`
- If a project-local config introduces `display_limit`, treat it as a presentation limit only. It MUST NOT limit generation.

### When to Stop Asking Questions

Stop the question-generation loop only when **all active lenses** return `meaningful-questions-exhausted`.

A lens is exhausted only when:

- it has explored the repository for its own concerns
- it has no remaining meaningful unresolved questions
- the remaining unknowns would not materially change design, planning, scope, or risk handling

A question is **not** meaningful if:

- it is a rewording of something already asked
- it can already be answered from the inquiry record, confirmed assumptions, or repository evidence
- it is merely nice to know, curiosity-driven, or unsupported by evidence

Do not stop just because one lens is exhausted. Inquiry ends when the full active lens set is exhausted.

## Presenting Approaches and the Design

### Approaches

Before writing the final design:

- Propose 2-3 different approaches
- Explain trade-offs and your recommendation
- Keep YAGNI in mind
- Remove scope that is not supported by the inquiry record

### Design Presentation

Once you believe you understand what you're building:

- Present the design in sections scaled to complexity
- Ask after each section whether it looks right so far
- Cover:
  - goals and non-goals
  - architecture
  - components and data flow
  - error handling
  - testing strategy
  - stage-relevant constraints surfaced by the lenses

If the user changes an answer during design review, append that change to the inquiry record, update the assumption state if needed, and revise the design accordingly.

## After the Design

### Documentation

- Write the validated design to `docs/plans/YYYY-MM-DD-<topic>-design-synthesis.md`
- Keep a durable inquiry record at `docs/plans/YYYY-MM-DD-<topic>-inquiry-record.md`
- Commit both files to git (on the feature branch)

### Implementation Planning

After documentation is committed:

- Invoke `execution-planning`
- Do NOT invoke any implementation skill directly from inquiry

`execution-planning` is the next step.

## Key Principles

- **Question generation happens via codex exec** — main session stays clean
- **Each Codex run owns one lens** — the calling environment coordinates parallelism
- **Repository exploration is mandatory** — short summaries are not enough on their own
- **Evidence-backed questions only** — no generic or speculative prompts
- **Raw question plus supplement** — preserve wording, add context separately
- **Inquiry record is authoritative** — later stages must use it
- **Assumptions are explicit state** — confirmed, working, and invalidated assumptions persist across rounds
- **Explore alternatives** — always propose 2-3 approaches before settling
- **Incremental validation** — present design, get approval before moving on
- **Be flexible** — go back and clarify when something does not make sense
