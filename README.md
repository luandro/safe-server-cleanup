# safe-server-cleanup

An agent skill for reclaiming disk space on a live, multi-tenant Linux host without breaking running services. Works with any AI coding agent that supports the `SKILL.md` format — Claude Code, Cursor, and others.

Written from a real incident, not from theory: a production host at 100% disk usage (125MB free) — Docker containers, cron-driven backups, and stale git worktrees all competing for space, serving multiple unrelated projects. The pitfalls table in [`SKILL.md`](./safe-server-cleanup/SKILL.md) is mistakes actually caught *during* that cleanup, not hypothetical ones.

## Why

"Cleanup" and "delete" are not the same thing. Almost every byte on a real host is either live data, a live cache, or something that looks abandoned but isn't. This skill exists to tell those apart *before* anything is removed — investigate → classify by reversibility → confirm anything ambiguous → act on the smallest safe unit → verify.

An agent that deletes the wrong volume on a shared host is worse than no agent at all.

## What it covers

- Disk investigation (`df`, `du`, Docker, cron, systemd) without assuming what's safe
- Classifying every finding into safe/reversible, needs-verification, or never-without-sign-off
- Verifying a Docker volume is truly orphaned before removing it (running + exited containers, compose files)
- Catching config-based retention settings that look correct but are silently broken
- Verifying whether "duplicated" `node_modules`/build caches across project copies are already deduped via hardlinks before recommending a different tool
- Distinguishing gitignored cruft from tracked, deliberately-committed files before deleting anything
- A pitfalls table of concrete failure modes (forcing past a process lock, deleting an in-use log file, racing a backup cron job, and more)

## Installation

Via the [Skills CLI](https://skills.sh) (installs to whichever agent you're using):

```bash
npx skills add https://github.com/luandro/safe-server-cleanup --skill safe-server-cleanup
```

Manually, for Claude Code:

```bash
git clone https://github.com/luandro/safe-server-cleanup /tmp/ssc && \
  cp -r /tmp/ssc/safe-server-cleanup ~/.claude/skills/safe-server-cleanup
```

For other agents, copy the `safe-server-cleanup/` directory into whatever skills/rules directory your agent reads from — the `SKILL.md` format itself isn't Claude-specific.

## License

MIT
