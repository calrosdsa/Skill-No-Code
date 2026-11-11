# Skill-No-Code

`architecting-nocode-apps` — a Claude Code / Codex skill for designing, building, reviewing, debugging and refactoring enterprise apps on the App-Builder no-code platform through its MCP server.

## Install

```bash
git clone https://github.com/calrosdsa/Skill-No-Code.git .claude/skills/architecting-nocode-apps
```

### Codex

Codex loads skills from a folder per skill containing `SKILL.md` (user-level `~/.codex/skills/`, or `.agents/skills/` inside a repo):

```bash
git clone https://github.com/calrosdsa/Skill-No-Code.git ~/.codex/skills/architecting-nocode-apps
```

Windows (PowerShell):

```powershell
git clone https://github.com/calrosdsa/Skill-No-Code.git "$env:USERPROFILE\.codex\skills\architecting-nocode-apps"
```

Restart Codex, then ask for App-Builder work (or mention `$architecting-nocode-apps`). Update later with `git pull` in that folder.

### Claude Code

Use it with `/architecting-nocode-apps [design|build|review|refactor|debug|audit] …`, or just describe App-Builder work and it loads automatically.

## Layout

- `SKILL.md` — entry point: rules, discovery, modes, build order, gap categories
- `references/` — loaded on demand (architecture, patterns, forms, workflows, menus, security, reliability, testing, lifecycle, anti-patterns, gap analysis, MCP discovery/tools/safe mutations)
- `examples/` — worked examples
- `evals/scenarios.md` — evaluation scenarios and results log

## Maintaining

When the MCP server changes, update the "Known discrepancies" table in `references/mcp-discovery.md`, re-run the scenarios, and record results in the log.
