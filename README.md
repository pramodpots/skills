# skills

My Claude / agent skills. Browse the `skills/` folder and copy any skill into your own `~/.claude/skills/`.

## Skills

| Skill | What it does |
|---|---|
| [github-setup](skills/github-setup) | Set up git, the `gh` CLI and SSH on a Mac from scratch so AI agents can create repos, push, and open/review PRs. Verifies with a real test. |

## Install a skill

```bash
mkdir -p ~/.claude/skills
cp -r skills/github-setup ~/.claude/skills/
```

Then ask Claude: "set up GitHub on this Mac".
