# Maieutics

*"The right questions draw out the right answers."*

[Japanese version](README.md)

## Why Questions Matter Most

Many failures in software development come not from poor code quality, but from an incomplete understanding of what should be built in the first place. An agent that starts writing code as soon as it hears a request may look efficient, but code written without disciplined questioning often solves the wrong problem.

**Maieutics is a skill library for Claude Code built around questioning first.** It examines a problem through five lenses: Product, Security, Maintainability, UX, and Architecture. Only after dialogue has clarified the substance of the work does it move on to implementation.

## Socrates and Maieutics

> "I teach nothing. I only ask questions." — Socrates

In 5th century BCE Athens, Socrates was a philosopher who kept questioning people in marketplaces and gymnasiums. He described himself as a "midwife of wisdom." Just as a midwife helps deliver a child, he believed questioning could help bring out the knowledge already latent within another person.

His method was distinctive. He did not hand out answers. If asked, "What is courage?", he would respond, "What do you think?" When the other person answered, he would continue: "Then what about a case like this?" Through repeated dialogue, people became aware of their assumptions and reached a deeper understanding.

In Plato's *Theaetetus*, Socrates says that the most important part of his midwifery is the ability to tell whether what a young mind is bringing forth is an illusion or the truth. The right questions sharpen vague thinking, expose what was overlooked, and illuminate the essence of the matter.

The name **Maieutics** comes from this Socratic art of midwifery. The goal is to give AI agents the same philosophy: question first, identify the essence, and only then begin to act. This library is an implementation of that idea.

## Features

- **Per-lens questioning**: Each lens runs in its own `codex exec`, while the calling session owns parallelism and aggregation
- **Evidence-backed questions**: Each lens explores the repo and returns questions grounded in observed files, configs, and boundaries
- **Inquiry Record**: Every user answer plus the evolving confirmed / working / invalidated assumptions are stored in a durable log
- **Codex-powered reviews**: Design reviews, plan reviews, and implementation reviews are run via `codex exec` — a separate model provides independent assessment
- **Review loops**: Reviews repeat until no Critical or Important issues remain

## Prerequisites

- [Claude Code](https://claude.com/claude-code)
- [Codex CLI](https://github.com/openai/codex) — required for question generation and reviews

## Installation

### Claude Code Plugin

```bash
# Clone the repository
git clone https://github.com/rarilurelo/maieutics.git ~/.claude/plugins/maieutics
```

### Manual (project-local)

```bash
# Copy skills to your project
cp -r path/to/maieutics/skills/ .claude/skills/
```

## The Workflow

```
inquiry → using-git-worktrees → execution-planning → delegated-execution → closing-the-branch
```

1. **inquiry** — Runs `codex exec` independently per lens, generates repo-exploration-backed questions, aggregates the results in the calling session, and records raw question + supplement plus assumption state in the Inquiry Record. Inquiry continues until every active lens reaches `meaningful-questions-exhausted`.

2. **using-git-worktrees** — Creates an isolated git worktree for the feature branch. Follows project conventions for location selection, then runs setup and baseline verification.

3. **execution-planning** — Creates bite-sized implementation tasks (2-5 min each) from the design. Reviews the plan via `codex exec` with fix loops. TDD, DRY, YAGNI.

4. **delegated-execution** — Dispatches a fresh Claude Code subagent per task. Two-stage review per task (spec compliance + code quality). Final lenses implementation review via `codex exec`.

5. **closing-the-branch** — Verifies tests, presents options (merge/PR/keep/discard), cleans up.

## Customization

### Lenses Config

Create `.maieutics/lenses.json` in your project root to customize lenses, display policy, and review settings. See `skills/inquiry/references/lenses.default.json` for the default configuration.

## Skills

| Skill | Purpose |
|---|---|
| `inquiry` | Per-lens question generation, Inquiry Record, Design Synthesis |
| `execution-planning` | Implementation plans with codex-powered review loops |
| `delegated-execution` | Subagent-per-task execution with review gates |
| `guided-execution` | Separate-session plan execution with human checkpoints |
| `using-git-worktrees` | Isolated workspace setup |
| `closing-the-branch` | Branch completion workflow |

## License

MIT License — see [LICENSE](LICENSE) for details.
