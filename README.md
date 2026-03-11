# Elenchus

*Named after the Socratic method of elenchus — reaching truth through disciplined questioning.*

Elenchus is a multi-perspective design and implementation workflow for Claude Code. It ensures your agent doesn't just jump into writing code — instead, it asks the right questions first, from multiple perspectives, to build the right thing.

## What makes it different

- **Multi-perspective questioning**: Questions are generated from 5 perspectives (Product, Security, Maintainability, UX, Architecture) via `codex exec`, not one-by-one by the main agent
- **Discovery Log**: Every user answer is recorded in a durable log that becomes the authoritative source of truth for all later stages
- **Codex-powered reviews**: Design reviews, plan reviews, and implementation reviews are run via `codex exec` — a separate model provides independent assessment
- **Review loops**: Reviews repeat until no Critical or Important issues remain

## Prerequisites

- [Claude Code](https://claude.com/claude-code)
- [Codex CLI](https://github.com/openai/codex) — required for question generation and reviews

## Installation

### Claude Code Plugin

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/elenchus.git ~/.claude/plugins/elenchus
```

### Manual (project-local)

```bash
# Copy skills to your project
cp -r path/to/elenchus/skills/ .claude/skills/
```

## The Workflow

```
brainstorming → writing-plans → subagent-driven-development → finishing-a-development-branch
```

1. **brainstorming** — Generates batched questions via `codex exec` from multiple perspectives. Records all answers in a Discovery Log. Produces an approved Design Doc.

2. **writing-plans** — Creates bite-sized implementation tasks (2-5 min each) from the design. Reviews the plan via `codex exec` with fix loops. TDD, DRY, YAGNI.

3. **subagent-driven-development** — Dispatches a fresh Claude Code subagent per task. Two-stage review per task (spec compliance + code quality). Final multi-perspective implementation review via `codex exec`.

4. **finishing-a-development-branch** — Verifies tests, presents options (merge/PR/keep/discard), cleans up.

## Customization

### Perspective Config

Create `.elenchus/multi-perspective.json` in your project root to customize perspectives, question batch size, and review settings. See `skills/brainstorming/references/multi-perspective.default.json` for the default configuration.

## Coexistence with Superpowers

Elenchus uses the `elenchus:` namespace and can coexist with the `superpowers` plugin. Both plugins will inject their entry-point skills at session start. Use whichever workflow fits your task.

## Skills

| Skill | Purpose |
|---|---|
| `brainstorming` | Multi-perspective question generation, Discovery Log, Design Doc |
| `writing-plans` | Implementation plans with codex-powered review loops |
| `subagent-driven-development` | Subagent-per-task execution with review gates |
| `executing-plans` | Separate-session plan execution with human checkpoints |
| `using-git-worktrees` | Isolated workspace setup |
| `finishing-a-development-branch` | Branch completion workflow |

## License

MIT License — see [LICENSE](LICENSE) for details.
