# Skill: Disk Hygiene

**Purpose:** Reclaim workstation disk space consumed passively by AI-assisted development — stale git worktrees, package-manager caches, build artefacts, and container images and volumes. None of this is reclaimed by any other ceremony: every worktree an agent session starts adds a full dependency install, every container build adds to a cache, and every removed worktree can orphan a volume. Left alone it ends in a full disk, which fails builds with misleading errors. The skill reclaims **caches and artefacts only — never source, never data, never the engineer's own files.**

**Trigger:** Engineer-initiated ("clean up the disk", "what's eating the disk", "run disk hygiene"), or prompted automatically by the AI at session start when the `Next disk hygiene sweep` date in the master rule file Section 9 has been reached. Also run it immediately, unscheduled, whenever free space is observed below the project's headroom threshold — enough for a fresh worktree plus a full build.

**Scheduling:** The next sweep date is stored in the master rule file Section 9 (Process Configuration). If today is on or after it, the AI prompts before any other work begins:

> "A disk hygiene sweep is scheduled. Would you like to run it now, or set a new date?"

**About the commands below:** they are examples for a macOS workstation using npm and Docker. Substitute the equivalents for the engineer's OS, package manager, and container runtime — the steps, the hard rules, and the verification are what transfer.

---

## The Two Hard Rules

Everything else in this skill is reversible. Read these before running any command.

1. **Never delete anything holding uncommitted or unmerged work.** Before removing any worktree, check `git status --porcelain` and `git rev-list --count main..<branch>`. If either is non-zero, the work is committed to its own branch first, and nothing is removed until it is safe in `.git`.
2. **Never delete a development database, a data volume, or untracked local data.** Gitignored data directories (fixtures, local database or storage-emulator data) cannot be recovered from `.git` — they look like artefacts and are not. A database container started without a named volume stores its data in an **anonymous** volume that is indistinguishable by name from build residue.

---

## Step 1 — Measure Before Touching Anything

Record the starting free space so the final report is evidence, not an estimate:

```bash
df -h <data volume> | tail -1
```

Then find the actual consumers, top-down, one level at a time:

```bash
du -x -d1 -h ~ 2>/dev/null | sort -hr | head -25
find ~ -xdev -type f -size +2G -print0 2>/dev/null | xargs -0 du -h 2>/dev/null | sort -hr | head -20
```

- **Always pass `-x`** (stay on one filesystem). Without it `du` descends into mounted disk images and counts them twice.
- Use `du`, not `ls`, for sparse files such as VM disk images — `ls` reports apparent size, not allocated size.
- Present the findings as a table of buckets before deleting anything. The largest buckets are often the engineer's own files (media, asset libraries, applications); the engineer needs to see that to make the call on them.

---

## Step 2 — Sweep Stale Worktrees

```bash
git worktree list
git branch --merged origin/main
```

Classify each worktree before acting:

| State | Action |
|---|---|
| Clean **and** 0 commits ahead of `main` | Remove: `git worktree remove <path>`, then `git branch -d <branch>` |
| Clean but ahead of `main` | Keep — it holds unmerged work. Reclaim only its dependency directory |
| Dirty (any uncommitted file) | Commit to its own branch first, then re-classify |

- Commit a dirty worktree by **explicit paths**, never a bulk add — in a tree shared by concurrent sessions, another session's work may already be staged.
- Use `git -C <path>` rather than `cd <path> &&`: if the directory is gone, `cd` falls back silently and the command runs against the wrong checkout.
- `git branch -d` refuses to delete an unmerged branch. Never use `-D` during a hygiene sweep.
- Never strip the dependency directory of a worktree another session is actively building in.

---

## Step 3 — Regenerable Caches

Package-manager caches, build-tool caches, and IDE derived data hold no unique state and regenerate on demand. Examples (macOS, npm): `npm cache clean --force`, `~/.nvm/.cache`, `~/Library/Caches/<tool>`, `~/Library/Developer/Xcode/DerivedData`, `~/.gradle/caches`.

- **A cache-clean command can refuse to run and still exit 0** — a wrapper (for example a package-manager shim) may intercept it and print a message instead. When in doubt, delete the cache directory directly.
- **Verify by re-measuring, never by the command's output.** After the batch, confirm each target is actually gone or smaller.
- Defer a cache another session is using right now (a running install or build) and say so in the report.

---

## Step 4 — Containers

Usually the largest single reclaimable item, and the only one that can destroy data. Inventory first (`docker system df`, `docker ps -a`, `docker images`).

**Work in this order: containers → images → build cache → volumes.** An earlier tier pins the later ones — a cache entry shared by a tagged image cannot be pruned until the image goes — so a "0 B reclaimed" result on an early tier is not evidence the runtime is clean.

1. **Build cache — safe.** `docker builder prune -f`. Add an age filter (`--filter until=24h`) when another session may be mid-build.
2. **Dangling images — safe.** `docker image prune -f`.
3. **Tagged images and exited containers — ask.** Present each with its age and size. Check whether an image can still be pulled — one from a decommissioned registry is gone for good. Remove containers **by name**, never with a bulk prune: a bulk prune cannot tell an old build container from a stopped dev database.
4. **Volumes — never blind.** A stopped container still owns its volumes, and stopped is the normal state between sessions. Build an explicit target list, derive the in-use set from **every** container (`docker ps -aq`, not `-q`), and require the intersection to be empty before removing anything:

```bash
docker ps -aq | while read c; do docker inspect "$c" --format '{{range .Mounts}}{{if eq .Type "volume"}}{{.Name}}{{"\n"}}{{end}}{{end}}'; done | grep -v '^$' | sort -u > in-use.txt
comm -12 <(sort targets.txt) in-use.txt     # must print nothing
xargs -n 20 docker volume rm < targets.txt  # BSD xargs has no -a
```

- **Named volumes** with no container are named because someone meant to keep them. Take each to the engineer individually. Mount it read-only to see what it holds, and offer to **archive before deleting** — volume contents compress well, so an archive keeps the data at a fraction of the space. Verify the archive (integrity, non-empty, and the specific files that matter are inside) before removing the source.
- Afterwards, prove nothing load-bearing was lost: named volumes still present, containers still up, the dev database still reachable.
- Freeing space inside a container VM may not shrink the host disk image immediately. Report the host figure, not the runtime's own.

---

## Step 5 — Judgment-Call Items

Reclaimable, but each needs the engineer's decision. Present sizes and ask — never batch these:

- **Mobile toolchain state** — device support files, simulators and emulators, simulator runtimes. Stop if a simulator is booted; check last-use dates before deleting a runtime, since re-downloading is slow.
- **Release archives** — these are shipped builds. Never bulk-delete.
- **Dependency directories in other projects** — exclude this repo's own, every live worktree's, and any sibling repo a concurrent session may be using. Build a manifest, report the exclusions, and say afterwards which projects need a reinstall.
- **Downloads** — always the engineer's call.

Some deletions complete asynchronously (for example, a simulator runtime unmounts after the command returns). Wait for them to finish before measuring, or the reclaim will be understated.

---

## Step 6 — Never Touch

Do not delete, and do not propose deleting:

- Source and project data — documents, desktop, media folders, any repository.
- Asset and sample libraries for creative tools. They are often the largest buckets on the disk and are irreplaceable or slow to re-acquire.
- Gitignored local data directories in any project.
- Installed applications — surface the total if it is large, and stop there.
- Any volume backing a database, unless the engineer names it explicitly.

If one of these dominates the disk, **report it and move on.** That decision belongs to the engineer.

---

## Step 7 — Report, Then Reschedule

```
Disk hygiene sweep — YYYY-MM-DD

Before:          [N] GiB free
After:           [N] GiB free
Total reclaimed: [N] GiB   ← the df delta, the authoritative figure

Reclaimed (indicative, per-item du):
  [item]                   [N] G
Verified after:        [volumes present, containers up, DB reachable, worktrees with unmerged work intact]
Deferred to engineer:  [item, size, why it needs a decision]
Left untouched:        [user-data buckets, with sizes]
Needs reinstall:       [projects whose dependency directories were removed]
```

The per-item figures will not sum to the `df` delta — shared and hardlinked blocks are counted more than once, and background reclaim can land between measurements. Report the endpoint delta as the number; if the gap is large, say so and name the cause. If a figure already given turns out wrong, correct it out loud.

Then ask:

> "When should the next disk hygiene sweep be scheduled? The recommended interval is 30 days. You can give a date or say 'in [N] weeks/months'."

Convert to an absolute date (YYYY-MM-DD), defaulting to 30 days from today, and update the master rule file Section 9:

```markdown
| **Last disk hygiene sweep** | YYYY-MM-DD | Date this sweep was completed |
| **Next disk hygiene sweep** | YYYY-MM-DD | AI prompts at session start on or after this date |
```
