# claude-skills

Personal collection of [Claude Code](https://claude.com/claude-code) skills — reusable, `SKILL.md`-based capabilities that Claude loads on demand for specific tasks.

## Skills

| Skill | Description |
|---|---|
| [`safe-server-cleanup`](./safe-server-cleanup/SKILL.md) | Reclaim disk space on a live, multi-tenant Linux host without breaking running services. Investigates before touching anything, classifies every finding by reversibility, and requires explicit confirmation for anything that isn't provably safe to delete. |

Each skill was written from a real incident, not from theory — `safe-server-cleanup` came out of an actual disk-full production host (100% used, 125MB free) with Docker, cron-driven backups, and stale git worktrees all competing for space. The pitfalls table in the skill is a list of mistakes that were caught *during* that cleanup, not hypothetical ones.

## Installation

**Project-scoped** (this skill only applies inside one repo):

```bash
mkdir -p .claude/skills
cp -r safe-server-cleanup /path/to/your-project/.claude/skills/
```

**Personal** (available in every Claude Code session, any project):

```bash
mkdir -p ~/.claude/skills
cp -r safe-server-cleanup ~/.claude/skills/
```

Claude Code discovers skills automatically from either location — no registration step. Invoke one explicitly with `/safe-server-cleanup`, or just describe the task ("the disk is full, clean it up") and Claude will load it when the description matches.

## Philosophy

These skills favor **investigate → classify → confirm → act on the smallest safe unit → verify** over "run the obvious command." A skill that deletes the wrong volume on a shared host is worse than no skill at all — every workflow here is built to make the safe path the default path, and to make you stop and ask before anything irreversible.

## Contributing

This is a personal collection, but issues and PRs describing a real incident a skill didn't handle well are welcome.

## License

MIT — see individual `SKILL.md` frontmatter for per-skill attribution.
