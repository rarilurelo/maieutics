# Maieutics

*Named after the Socratic method of maieutics — reaching truth through disciplined questioning.*

Maieutics is a lenses-driven design and implementation workflow for Claude Code. It ensures your agent doesn't just jump into writing code. It asks the right questions first, through multiple lenses, to build the right thing.

## What makes it different

- **Lenses-based questioning**: Questions are generated from 5 lenses (Product, Security, Maintainability, UX, Architecture) via `codex exec`, not one-by-one by the main agent
- **Inquiry Record**: Every user answer is recorded in a durable log that becomes the authoritative source of truth for all later stages
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
inquiry → execution-planning → delegated-execution → closing-the-branch
```

1. **inquiry** — Generates batched questions via `codex exec` from multiple lenses. Records all answers in an Inquiry Record. Produces an approved Design Synthesis.

2. **execution-planning** — Creates bite-sized implementation tasks (2-5 min each) from the design. Reviews the plan via `codex exec` with fix loops. TDD, DRY, YAGNI.

3. **delegated-execution** — Dispatches a fresh Claude Code subagent per task. Two-stage review per task (spec compliance + code quality). Final lenses implementation review via `codex exec`.

4. **closing-the-branch** — Verifies tests, presents options (merge/PR/keep/discard), cleans up.

## Customization

### Lenses Config

Create `.maieutics/lenses.json` in your project root to customize lenses, question batch size, and review settings. See `skills/inquiry/references/lenses.default.json` for the default configuration.

## Skills

| Skill | Purpose |
|---|---|
| `inquiry` | Lenses-based question generation, Inquiry Record, Design Synthesis |
| `execution-planning` | Implementation plans with codex-powered review loops |
| `delegated-execution` | Subagent-per-task execution with review gates |
| `guided-execution` | Separate-session plan execution with human checkpoints |
| `using-git-worktrees` | Isolated workspace setup |
| `closing-the-branch` | Branch completion workflow |

## License

MIT License — see [LICENSE](LICENSE) for details.
