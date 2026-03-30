---
name: brainstorming
description: "You MUST use this before any creative work - creating features, building components, adding functionality, or modifying behavior. Explores user intent, requirements, and design before implementation, using multi-perspective question generation and discovery logging."
---

# Brainstorming Ideas Into Designs

## Overview

Help turn ideas into fully formed designs and specs through natural collaborative dialogue.

This version extends the normal brainstorming flow with **multi-perspective question generation**. Questions are generated via `codex exec` so the main Claude Code session stays clean. Codex's job is **question generation only**. It does not design, plan, or implement.

The user's raw input, every generated question, and every answer are saved and become **authoritative inputs** for the rest of the workflow.

Do NOT invoke any implementation skill, write any code, scaffold any project, or take any implementation action until you have presented a design and the user has approved it. This applies to EVERY project regardless of perceived simplicity.

## Anti-Pattern: "This Is Too Simple To Need A Design"

Every project goes through this process. A todo list, a single-function utility, a config change — all of them.

"Simple" projects are where unexamined assumptions cause the most wasted work. The design can be short for a truly simple project, but you MUST still present it and get approval.

## Authoritative Inputs and Artifacts

**Project-local perspective config:**
- Prefer `.maieutics/multi-perspective.json` if it exists
- Otherwise use [references/multi-perspective.default.json](references/multi-perspective.default.json)

**Discovery log:**
- Save to `docs/plans/YYYY-MM-DD-<topic>-discovery.md`
- Start from [references/discovery-log-template.md](references/discovery-log-template.md)

**Design doc:**
- Save to `docs/plans/YYYY-MM-DD-<topic>-design.md`
- Start from [references/design-doc-template.md](references/design-doc-template.md)

**Authority order:**
1. Latest explicit user answers in the discovery log
2. Raw user input captured in the discovery log
3. Approved design
4. Everything else

If design text later conflicts with the discovery log, update the design. Do NOT silently override the user's answers.

## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Explore project context** — check files, docs, recent commits
2. **Load perspective config** — use project-local config if present, otherwise the bundled default
3. **Create isolated workspace** — invoke `using-git-worktrees` to create a feature branch. Use the user's initial topic/request to derive the branch name. All subsequent artifacts (discovery log, design doc, etc.) are committed on this feature branch, never on main.
4. **Create or update discovery log** — save the user's raw input verbatim before asking more questions
5. **Generate next question batch via codex exec** — use [question-generator-prompt.md](question-generator-prompt.md), passing file paths to the discovery log and perspective config
6. **Ask the user a grouped batch of questions** — ask 3-5 questions in one message (default: 4). **Do NOT pass Codex's question text through as-is.** Rewrite each question for the user following these rules:
   - **Plain-language rewrite**: Rephrase the question so that a non-engineer stakeholder can understand it. If the original uses jargon (e.g. "idempotency", "eventual consistency", "trust boundary"), keep the term but add a short inline explanation in parentheses.
   - **Context bridge**: Add one sentence explaining *why this question came up now* — connect it to the user's earlier answers or the current state of the discovery log.
   - **Expanded choices**: When `suggested_choices` exist, present each choice with a brief pros/cons note or concrete example so the user can make an informed pick.
   - **`why_it_matters` integration**: Weave the `why_it_matters` value naturally into the question or present it as a short "This matters because …" line below the question — do not drop it silently.
   - **Preserve intent**: The rewritten question must ask for the same information Codex requested. Do not narrow, broaden, or redirect the question's scope.
7. **Record answers verbatim** — append each question (both the original Codex question and the rewritten version) and the user's answer to the discovery log
8. **Repeat until ready for design** — continue question rounds until the external question generator says the design can be written, or until only human-decided tradeoffs remain
9. **Propose 2-3 approaches** — with trade-offs and your recommendation
10. **Present design** — in sections scaled to complexity, get user approval after each section
11. **Write design doc** — save to `docs/plans/YYYY-MM-DD-<topic>-design.md` and commit (now on the feature branch)
12. **Transition to implementation planning** — invoke `writing-plans`

## Multi-Perspective Question Generation

### How to Generate Questions

Use `codex exec` so the main Claude Code session does not get polluted with intermediate brainstorming attempts. Codex can read files in the working directory directly — pass file paths, not file contents.

Preferred pattern:

1. Main session ensures the discovery log is saved to disk with the latest user answers
2. Main session runs `codex exec` with the prompt from [question-generator-prompt.md](question-generator-prompt.md), substituting:
   - `[DISCOVERY_LOG_PATH]` → actual path to the discovery log (e.g. `docs/plans/2026-03-10-slack-bot-discovery.md`)
   - `[PERSPECTIVE_CONFIG_PATH]` → `.maieutics/multi-perspective.json` or the bundled default
   - `Project Context Summary` → a short summary of the repo (tech stack, structure, relevant constraints)
3. Codex reads the files, generates questions, and returns **only structured JSON output**
4. Main session parses the JSON, asks the user the question batch, and records the answers in the discovery log

#### Exact Command

**IMPORTANT: Run this command in the foreground (do NOT use `run_in_background`).** Foreground execution keeps `RUN_ID` in scope so you can read the correct output file immediately after completion.

```bash
RUN_ID=$(uuidgen) && codex exec --full-auto -s read-only -o /tmp/maieutics-questions-${RUN_ID}.json - <<'PROMPT'
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

### Rules for Question Batches

- Ask **3-5 questions per round**; default to **4**
- Prefer different perspectives in the same batch unless one perspective contains the only remaining blocker
- Prefer multiple-choice or constrained questions when helpful
- Do NOT ask for information already answered
- Do NOT generate implementation steps, code, or plans in the external question worker
- Do NOT dump the entire subprocess transcript into the main conversation; save durable information to the discovery log instead

### When to Stop Asking Questions

Stop the question-generation loop when one of these is true:

- Codex returns `enough-context`
- The remaining unknowns are minor enough that the design can still be validated section-by-section
- The only remaining blockers are tradeoffs that must be decided by the human, and you are ready to ask them directly

## Presenting Approaches and the Design

### Approaches

Before writing the final design:

- Propose 2-3 different approaches
- Explain trade-offs and your recommendation
- Keep YAGNI in mind
- Remove scope that is not supported by the discovery log

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
  - stage-relevant constraints surfaced by the perspectives

If the user changes an answer during design review, append that change to the discovery log and revise the design accordingly.

## After the Design

### Documentation

- Write the validated design to `docs/plans/YYYY-MM-DD-<topic>-design.md`
- Keep a durable discovery log at `docs/plans/YYYY-MM-DD-<topic>-discovery.md`
- Commit both files to git (on the feature branch)

### Implementation Planning

After documentation is committed:

- Invoke `writing-plans`
- Do NOT invoke any implementation skill directly from brainstorming

`writing-plans` is the next step.

## Key Principles

- **Question generation happens via codex exec** — main session stays clean
- **Codex only asks questions** — no design, no planning, no implementation
- **Grouped questions, not one-by-one** — 3-5 per round
- **Discovery log is authoritative** — later stages must use it
- **Explore alternatives** — always propose 2-3 approaches before settling
- **Incremental validation** — present design, get approval before moving on
- **Be flexible** — go back and clarify when something does not make sense
