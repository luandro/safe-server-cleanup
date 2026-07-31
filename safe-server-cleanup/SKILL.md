---
name: safe-server-cleanup
description: Reclaim disk space on a live, multi-tenant Linux host without breaking running services — investigate before touching anything, classify findings by reversibility, and require explicit confirmation for anything that isn't provably safe to delete.
version: 1.1.0
author: luandro
license: MIT
tags: [devops, disk-space, cleanup, docker, sysadmin, safety, node-modules, package-managers]
category: devops
---

# Safe Server Cleanup

Free disk space on a server that is actively running other people's services. The
premise of this skill is that "cleanup" and "delete" are not the same thing:
almost every byte on a real host is either live data, a live cache, or something
that looks abandoned but isn't. The workflow below exists to tell those apart
**before** anything is removed.

Never run this as "delete anything old." Run it as: investigate → classify →
confirm → act on the smallest safe unit → verify.

## When to use

- `df` shows a filesystem above ~85-90% used, or a healthcheck/alert fired for
  disk or inode pressure.
- User asks to "clean up the server," "free up disk space," or similar.
- Before provisioning something new on a host that's tight on space.

## Step 1 — Investigate (read-only, no destructive commands yet)

Build a full picture before forming an opinion about what to delete.

```bash
df -h                                   # which filesystem is actually full
du -xh --max-depth=1 / 2>&1 | sort -rh  # top-level space hogs (can be slow on a full/loaded disk — background it)
docker ps -a                            # what's actually running vs. exited
docker system df -v                     # images / containers / volumes breakdown
crontab -l                              # what automation already exists — don't fight it or duplicate it
systemctl list-units --type=service --state=running
journalctl --disk-usage
du -sh /var/cache/apt /var/log 2>&1
du -xh --max-depth=2 ~/.cache ~/.npm ~/.bun ~/.local ~/.nvm ~/.sdkman 2>&1 | sort -rh | head -20
du -sh ~/.local/share/Trash 2>&1        # desktop trash is never auto-emptied
find ~/.npm/_npx -maxdepth 1 -newermt "-30 days" 2>/dev/null | wc -l  # 0 = every npx cache entry is stale
```

If `du` on `/` hangs for minutes, that's information too (I/O pressure, near-full
disk) — background it with `run_in_background` rather than blocking, and narrow
scope to `/home/<user>` and `/var` in parallel instead of waiting.

**Check for cleanup automation that already exists** (retention scripts, log
rotation, healthchecks) before adding anything new — a host with real users
usually already has *some* of this, and duplicating it causes confusion or
double-deletion.

**A configured retention limit existing is not proof it's being enforced.** A
tool can read `keep: N` from its own config and still grow unbounded if an
internal safety check silently disables pruning under some condition (e.g.
"don't prune when the last snapshot was incomplete" — and it's *always*
incomplete because one file permanently exceeds a size cap). Before trusting
a `keep`/`retention` setting, count what's actually on disk and compare it to
the configured number — if there are more than configured, the enforcement
is broken somewhere, config aside.

## Step 2 — Classify every candidate

Sort findings into three buckets. The bucket determines what happens next, not
the size of the win.

| Bucket | Definition | Examples | Action |
|---|---|---|---|
| **Safe / reversible** | Regenerable from a cache, or explicitly designed to be pruned | apt package cache, dangling (untagged) Docker images, stopped-container-only `docker container prune`, old `journalctl` entries beyond a retention window, rotated `.log.1`/`.gz` files, desktop Trash, package-manager caches (`bun pm cache rm`, `npm cache clean --force`, `uv cache clean`), stale `~/.npm/_npx/*` entries | Do it directly, no confirmation needed |
| **Needs verification** | Looks unused but could still be load-bearing | Docker volumes with no *current* container reference, old kernel packages, large files in a project directory, unfamiliar top-level dirs | Verify first (see Step 3), then confirm with user before deleting |
| **Never touch without explicit sign-off** | Named/attached volumes, anything under an active service's data dir, production databases, `.env`/secrets, anything another user/team owns | `postgres` data volumes for running containers, `paperclip_data`-style app volumes, backup directories, retention-script targets | Ask first, every time — regardless of how confident you are |

Being "80% full" is never itself justification to skip Step 3 for a "needs
verification" item. Time pressure does not change the bucket.

## Step 3 — Verify before touching a "needs verification" item

For a Docker volume that looks orphaned:

```bash
docker volume inspect <name>                       # when created, by which compose project
docker ps -a --filter volume=<name>                # ANY container, including exited — if one shows up, it is NOT orphaned
find / -maxdepth 6 -iname "docker-compose*.y*ml" 2>/dev/null | xargs grep -l "<project>" 2>/dev/null
```

A volume is only a real deletion candidate when **all** of: no running
container, no exited container, and no compose file on the host references it.
Even then, prefer mounting it read-only in a throwaway container to eyeball
contents before deleting — a stale-looking volume can still be the one copy of
someone's data.

For an old kernel package: confirm it is not the currently running kernel
(`uname -r`) and that at least one fallback kernel remains installed before
removing others.

**Before deleting any single file that looks like disposable cruft** (a stray
lockfile, a config left over from a migration, anything you're about to `rm`
one-off rather than as part of a whole gitignored directory): check whether
it's actually tracked in a repo before assuming it's cruft.

```bash
git status --porcelain <path>   # empty output + tracked = matches HEAD, it's real committed content
git check-ignore -v <path>      # non-empty = gitignored, safe to treat as disposable
git log -1 --oneline -- <path>  # a real commit history means someone put it there on purpose
```

A file sitting next to what looks like its replacement (e.g. an old
`package-lock.json` next to a newer `bun.lock`) is not proof it's stale
debris — it may be a deliberately committed file the repo still relies on.
Deleting a tracked file from a working copy doesn't even save disk long-term
(it comes back on the next `pull`/`clone`) and turns a disk-cleanup task into
an uncommitted repo change that isn't yours to make. Only delete what's
actually gitignored or genuinely outside any repo.

### Verifying a "duplication" claim before recommending a tooling fix

When several copies of the same project (worktrees, clones, CI checkouts) each
carry their own `node_modules`/`vendor`/build cache, don't assume the naive
sum of `du` numbers is real wasted disk — many modern package managers (bun,
pnpm, and others) already hardlink from a shared global cache, so the "3x
1.1GB" you see per-directory can already be near-free in reality. Verify
before recommending a migration to a different tool:

```bash
# Combined du in ONE invocation dedupes by (device, inode) across ALL paths given —
# if the combined total is close to a single copy's size, they're already hardlinked.
du -sh <copy1> <copy2> <copy3> --total 2>&1 | tail -1

# Direct proof: same inode/nlink>1 across copies means real sharing, not naive du math
stat -c '%n nlink=%h inode=%i' <copy1>/<same-file> <copy2>/<same-file>
```

If the copies are **not** actually sharing (different inodes, `nlink=1`), the
fix is usually consistency (make every install go through the same tool the
project already standardized on) rather than adopting a new package manager —
check for a stale alternate lockfile (§ above) causing some installs to bypass
the existing cache/hardlink mechanism before concluding the current tool is
insufficient.

## Step 4 — Act, smallest safe unit first

1. Do every "safe / reversible" item — these need no sign-off, but list what you
   did and how much space each step freed.
2. For each "needs verification" item that passed Step 3, present it to the
   user by name with its size and evidence of non-use, and wait for explicit
   confirmation before deleting. Batch these into one question rather than
   asking once per item.
3. Never touch the "never without sign-off" bucket even if the user's original
   request was broad ("clean up everything") — restate what's in that bucket and
   why, and let them decide per item.
4. Prefer non-destructive shrink over delete where both exist: `truncate -s 0`
   an actively-written log instead of `rm` (avoids breaking the writer's open
   file handle), `journalctl --vacuum-size=200M` instead of clearing
   `/var/log/journal` by hand.

## Step 5 — Verify after

```bash
df -h                      # confirm space was actually freed on the right filesystem
docker ps -a                # confirm nothing that should be running got removed
systemctl status <service>  # spot-check anything adjacent to what you touched
```

Report a before/after `df -h` line, not just "cleanup done."

## Pitfalls

| Pitfall | Why it happens | Fix |
|---|---|---|
| `docker system prune -a --volumes` | Fastest way to "fix" the disk, also fastest way to delete a running app's database | Never run the `--volumes` or bare `-a` forms without itemized, per-resource confirmation |
| Deleting an in-use log file | Process holds the fd open; `rm` doesn't reclaim space until the process exits/reopens, and can break log rotation assumptions | `truncate -s 0` instead |
| Removing the running kernel | Confused "old" with "not the one in `uname -r`" | Always check `uname -r` first |
| Treating "no container running" as "orphaned volume" | Exited-but-not-removed containers still count as references | Check `docker ps -a`, not `docker ps` |
| Racing a backup/retention cron job | Deleting something mid-backup, or right before a retention script needed it as a source | Check `crontab -l` for jobs touching the same paths first |
| Forcing a cache-clean past a lock | `--force`-ing past "cache in use, waiting for other process" just makes the wait go away, not the reason for it | If a cache tool refuses/waits on a live process's lock, leave it — that process is actively using the cache, not stale |
| Deleting a tracked file because it "looks stale" | An old lockfile/config next to a newer one can still be deliberately committed, not leftover | `git status --porcelain` + `git log -- <path>` before deleting anything outside a gitignored dir |
| Recommending a new tool to fix "duplication" that's already deduped | Separate `du` calls per directory don't reveal hardlinks; naive summed size looks like real waste even when it isn't | Combined `du --total` or `stat -c nlink` across the copies before concluding the current tool is insufficient |
