---
name: maieutics-orientation
description: Use when starting any conversation - establishes how to find and use maieutics skills, requiring Skill tool invocation before ANY response including clarifying questions
---

<SUBAGENT-STOP>
If you were dispatched as a subagent to execute a specific task, skip this skill.
</SUBAGENT-STOP>

<EXTREMELY-IMPORTANT>
If you think there is even a 1% chance a skill might apply to what you are doing, you ABSOLUTELY MUST invoke the skill.

IF A SKILL APPLIES TO YOUR TASK, YOU DO NOT HAVE A CHOICE. YOU MUST USE IT.

This is not negotiable. This is not optional. You cannot rationalize your way out of this.
</EXTREMELY-IMPORTANT>

## Instruction Priority

Maieutics skills override default system prompt behavior, but **user instructions always take precedence**:

1. **User's explicit instructions** (CLAUDE.md, AGENTS.md, direct requests) — highest priority
2. **Maieutics skills** — override default system behavior where they conflict
3. **Default system prompt** — lowest priority

If CLAUDE.md or AGENTS.md says "don't use TDD" and a skill says "always use TDD," follow the user's instructions. The user is in control.

## Codex Requirement

Maieutics requires `codex exec` for lenses question generation and reviews. There is no fallback. Ensure the Codex CLI is installed and available in your PATH before using maieutics skills.

Question generation and design/plan/implementation reviews are executed via `codex exec` — not via subagents. This keeps the main Claude Code session clean and leverages a separate model for independent review.

## How to Access Skills

Use the `Skill` tool. When you invoke a skill, its content is loaded and presented to you — follow it directly. Never use the Read tool on skill files.

# Using Skills

## The Rule

**Invoke relevant or requested skills BEFORE any response or action.** Even a 1% chance a skill might apply means that you should invoke the skill to check. If an invoked skill turns out to be wrong for the situation, you don't need to use it.

```dot
digraph skill_flow {
    "User message received" [shape=doublecircle];
    "Building something new?" [shape=diamond];
    "Invoke inquiry skill" [shape=box];
    "Might any skill apply?" [shape=diamond];
    "Invoke Skill tool" [shape=box];
    "Announce: 'Using [skill] to [purpose]'" [shape=box];
    "Has checklist?" [shape=diamond];
    "Create TodoWrite todo per item" [shape=box];
    "Follow skill exactly" [shape=box];
    "Respond (including clarifications)" [shape=doublecircle];

    "User message received" -> "Building something new?";
    "Building something new?" -> "Invoke inquiry skill" [label="yes"];
    "Building something new?" -> "Might any skill apply?" [label="no"];
    "Invoke inquiry skill" -> "Might any skill apply?";

    "Might any skill apply?" -> "Invoke Skill tool" [label="yes, even 1%"];
    "Might any skill apply?" -> "Respond (including clarifications)" [label="definitely not"];
    "Invoke Skill tool" -> "Announce: 'Using [skill] to [purpose]'";
    "Announce: 'Using [skill] to [purpose]'" -> "Has checklist?";
    "Has checklist?" -> "Create TodoWrite todo per item" [label="yes"];
    "Has checklist?" -> "Follow skill exactly" [label="no"];
    "Create TodoWrite todo per item" -> "Follow skill exactly";
}
```

## Skill Catalog

### Core Workflow (in order)

1. **maieutics:inquiry** — Lenses-based question generation via `codex exec`. Explores user intent through batched questions from 5 lenses (Product, Security, Maintainability, UX, Architecture). Produces an Inquiry Record and Design Synthesis.

2. **maieutics:execution-planning** — Creates bite-sized implementation plans from approved design syntheses. Reviews plans via `codex exec` with fix loops until no Critical/Important issues remain.

3. **maieutics:delegated-execution** — Executes plans by dispatching fresh implementation workers (Agent tool) per task. Two-stage per-task review (spec compliance + code quality). Final lenses implementation review via `codex exec`.

4. **maieutics:guided-execution** — Alternative to delegated-execution for separate-session execution with human checkpoints between batches.

### Supporting Skills

5. **maieutics:using-git-worktrees** — Creates isolated git worktrees for feature work. Auto-detects project setup, verifies clean test baseline.

6. **maieutics:closing-the-branch** — Guides branch completion: verify tests, present 4 options (merge/PR/keep/discard), execute chosen workflow, clean up worktree.

### Feedback Loop

7. **maieutics:feedback-reconciliation** — Triages external feedback (staging tests, UAT, code review, bug reports) by classifying each item across 5 lenses via `codex exec`. Promotes accepted changes to the Inquiry Record and routes to the appropriate pipeline stage (inquiry for scope changes, execution-planning for design/implementation fixes).

### Typical Flow

```
inquiry → using-git-worktrees → execution-planning → delegated-execution → closing-the-branch
     ^                                       ^                                                |
     |                                       |                                                v
     +-------------- feedback-reconciliation ---+------------------------------- (external feedback arrives)
```

## Skill Priority

When multiple skills could apply, use this order:

1. **Process skills first** (inquiry) — these determine HOW to approach the task
2. **Implementation skills second** — these guide execution

"Let's build X" → inquiry first, then implementation skills.

## Red Flags

These thoughts mean STOP — you're rationalizing:

| Thought | Reality |
|---------|---------|
| "This is just a simple question" | Questions are tasks. Check for skills. |
| "I need more context first" | Skill check comes BEFORE clarifying questions. |
| "Let me explore the codebase first" | Skills tell you HOW to explore. Check first. |
| "I can check git/files quickly" | Files lack conversation context. Check for skills. |
| "Let me gather information first" | Skills tell you HOW to gather information. |
| "This doesn't need a formal skill" | If a skill exists, use it. |
| "I remember this skill" | Skills evolve. Read current version. |
| "This doesn't count as a task" | Action = task. Check for skills. |
| "The skill is overkill" | Simple things become complex. Use it. |
| "I'll just do this one thing first" | Check BEFORE doing anything. |
| "This feels productive" | Undisciplined action wastes time. Skills prevent this. |

## Skill Types

**Rigid** (inquiry, execution-planning): Follow exactly. Don't adapt away discipline.

**Flexible** (using-git-worktrees): Adapt principles to context.

The skill itself tells you which.

## User Instructions

Instructions say WHAT, not HOW. "Add X" or "Fix Y" doesn't mean skip workflows.
