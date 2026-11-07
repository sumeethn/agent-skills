# two-phase-development

Claude Code skill. Design convergence between Claude (architect) and Codex (critic), user design
approval, then Codex implementation with Claude review and automatic fail-closed merge. Built on
Orca orchestration (the `orchestration` skill); it adds no orchestration framework of its own.

Project-specific settings, credentials and run history are not stored here. Run state lives in each
project under `$(git rev-parse --git-common-dir)/two-phase/`.

## Install (user scope)

```bash
git clone https://github.com/sumeethn/agent-skills ~/src/agent-skills
ln -s ~/src/agent-skills/skills/two-phase-development ~/.claude/skills/two-phase-development
```

`~/.claude/skills/two-phase-development/SKILL.md` then resolves into the clone. Update with
`git -C ~/src/agent-skills pull`.

## Invoke

In Claude Code inside any project repository, with Orca running:

```
/two-phase-development <task description>
```

or ask for "two-phase development" of a task. The skill inspects the project's instructions first
and asks only for what is missing.
