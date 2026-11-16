# sstack

My agent skills. Small, composable, work with any agent that reads `SKILL.md`
(Claude Code, Cursor, Codex, Copilot, Gemini CLI, ...).

## Install

Every skill at once:

```bash
npx skills add satyasootar/sstack
```

One skill:

```bash
npx skills add satyasootar/sstack --skill grill-me
```

Add `-g` to install globally, `-a claude-code` to target one agent.
Update later with `npx skills update`.

Claude Code plugin (auto-updates):

```bash
/plugin marketplace add satyasootar/sstack
/plugin install sstack@sstack
```

## Skills

| Skill | What it does |
|---|---|
| [grill-me](skills/grill-me/SKILL.md) | Interviews you to sharpen a plan before coding |
| [commit](skills/commit/SKILL.md) | Writes a conventional commit for staged changes |
| [jira-commit](skills/jira-commit/SKILL.md) | `/jira-commit <KEY>` — commits relevant changes as `KEY: summary`, never pushes. Needs Atlassian MCP |
| [jira-log](skills/jira-log/SKILL.md) | `/jira-log <KEY>` — creates subtask(s), assigns you, sets points/versions/component, logs work, marks Done. Needs Atlassian MCP |

## Adding a skill

1. `npx skills init skills/<name>` (or copy an existing folder).
2. Fill in `name` (must equal the folder name) and `description` (say *when* to use it; agents pick skills by this line).
3. Add a row to the table above, commit, push. It's installable immediately.

## License

MIT
